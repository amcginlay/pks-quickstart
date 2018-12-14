# pks-quickstart

## Prerequisites

- A registered top-level domain (e.g. pivotaledu.io)
- An ppropriately configured subdomain (e.g. cls66env99) to represent this instance
- A [GCP account](https://cloud.google.com/) (credit card identification is required)
- A pristine GCP project
- A [PivNet](https://network.pivotal.io/) account

## Creating a jumpbox on GCP

New GCP users may need to [activate Cloud Shell](http://cloud.google.com/shell)

From within your _pristine_ GCP project, open the Cloud Shell.

![gcp_cli_launch](gcp_cli_launch.png)

From your Cloud Shell session, execute the following `gcloud` command 
to create your jumpbox.

```bash
gcloud compute instances create "jbox-pcf" \
  --image-project "ubuntu-os-cloud" \
  --image-family "ubuntu-1804-lts" \
  --boot-disk-size "200" \
  --machine-type=f1-micro \
  --tags="jbox-pcf" \
  --zone us-central1-a
```

## SSH to your new jumpbox

```bash
gcloud compute ssh ubuntu@jbox-pcf --zone us-central1-a
```

If you would prefer to use the [Google Cloud SDK](https://cloud.google.com/sdk/install) from 
your local machine, remember to first authenticate with `gcloud auth login` and add the
`--project <TARGET_PROJECT_ID>` argument to the commands shown above.

## Initialize the `gcloud` CLI on the jumpbox:

From your jumpbox SSH session, authenticate the SDK with your GCP account

```bash
gcloud auth login
```

Follow the on-screen prompts. We will need to copy-paste the URL from 
our `gcloud` CLI jumpbox session into a local browser in order to select 
the account you have registered for use with Google Cloud. Additionally, 
you'll need copy-paste the verification code back into your jumpbox 
session to complete the login sequence.

## Enable the GCP services APIs in current project

```bash
gcloud services enable compute.googleapis.com && \
gcloud services enable iam.googleapis.com && \
gcloud services enable cloudresourcemanager.googleapis.com && \
gcloud services enable dns.googleapis.com && \
gcloud services enable sqladmin.googleapis.com
```

## Prepare your toolbox

Install tools:

```bash
sudo apt update --yes && \
sudo apt install --yes unzip && \
sudo apt install --yes jq && \
sudo apt install --yes build-essential && \
sudo apt install --yes ruby-dev && \
sudo gem install --no-ri --no-rdoc cf-uaac
```

```bash
VERSION=0.11.10
wget -O terraform.zip https://releases.hashicorp.com/terraform/${VERSION}/terraform_${VERSION}_linux_amd64.zip && \
  unzip terraform.zip && \
  sudo mv terraform /usr/local/bin

VERSION=0.46.0
wget -O om https://github.com/pivotal-cf/om/releases/download/${VERSION}/om-linux && \
  chmod +x om && \
  sudo mv om /usr/local/bin/

VERSION=0.0.55
wget -O pivnet https://github.com/pivotal-cf/pivnet-cli/releases/download/v${VERSION}/pivnet-linux-amd64-${VERSION} && \
  chmod +x pivnet && \
  sudo mv pivnet /usr/local/bin/
  
VERSION=5.4.0
wget -O bosh https://s3.amazonaws.com/bosh-cli-artifacts/bosh-cli-${VERSION}-linux-amd64 && \
  chmod +x bosh && \
  sudo mv bosh /usr/local/bin/
```

Verify that these tools were installed:

```bash
which unzip; which jq; which uaac; which terraform; which om; which pivnet; which bosh
```

## Fetch the and configure the platform automation scripts

Clone the platform automation scripts and change into the directory of our cloned repo
to keep our task script commands short

```bash
git clone https://github.com/amcginlay/ops-manager-automation.git ~/ops-manager-automation
cd ~/ops-manager-automation
```

Create a `~/.env` configuration file to describe the specifics of your environment.

```bash
./scripts/create-env.sh
```

_Note_ you should not proceed until you have __customized__ the "CHANGE_ME" settings 
in your `~/.env` file to suit your target environment.

## Register the configuration file

Now that we have the `~/.env` file loaded with the essential variables, we need 
to ensure that these get set into the shell, both now and every subsequent time
the `ubuntu` user connects to the jumpbox.

```bash
source ~/.env
echo "source ~/.env" >> ~/.bashrc
```

To review your currently active variable settings:

```bash
set | grep PCF
```

## Create a GCP service account for terraform in current project

```bash
gcloud iam service-accounts create terraform --display-name terraform

gcloud projects add-iam-policy-binding $(gcloud config get-value core/project) \
  --member "serviceAccount:terraform@$(gcloud config get-value core/project).iam.gserviceaccount.com" \
  --role 'roles/owner'

# generate and download a key for the service account
gcloud iam service-accounts keys create 'gcp_credentials.json' \
  --iam-account "terraform@$(gcloud config get-value core/project).iam.gserviceaccount.com"
```

## Download an Ops Manager image identifier from Pivotal Network

```bash
OPSMAN_VERSION=2.3.5

PRODUCT_NAME="Pivotal Cloud Foundry Operations Manager" \
DOWNLOAD_REGEX="Pivotal Cloud Foundry Ops Manager YAML for GCP" \
PRODUCT_VERSION=${OPSMAN_VERSION} \
  ./scripts/download-product.sh

OPSMAN_IMAGE=$(bosh interpolate ./downloads/ops-manager_${OPSMAN_VERSION}_*/OpsManager*onGCP.yml --path /us)
```

Check the value of `OPSMAN_IMAGE` before continuing.

## Download and unzip the Terraform scripts from Pivotal Network

The Terraform scripts which deploy the Ops Manager are also responsible for
building the IaaS plumbing to support the PAS.
That is why we turn our attention to the PAS product in PivNet when sourcing the
Terraform scripts.

Please note:
- Ops Manager and PAS versions are often in-sync but this is not enforced.
- These Terraform scripts can support the infrastructure for PAS _or_ PKS.

```bash
PAS_VERSION=2.3.3

PRODUCT_NAME="Pivotal Application Service (formerly Elastic Runtime)" \
DOWNLOAD_REGEX="GCP Terraform Templates" \
PRODUCT_VERSION=${PAS_VERSION} \
  ./scripts/download-product.sh
    
unzip ./downloads/elastic-runtime_${PAS_VERSION}_*/terraforming-gcp-*.zip -d .
```

## Generate a wildcard SAN certificate

```no-highlight
./scripts/mk-ssl-cert-key.sh
```

## Create the `terraform.tfvars` file

```bash
cd ~/ops-manager-automation/pivotal-cf-terraforming-gcp-*/terraforming-pks/

cat > terraform.tfvars <<-EOF
env_name            = "${PCF_SUBDOMAIN_NAME}"
project             = "$(gcloud config get-value core/project)"
region              = "${PCF_REGION}"
zones               = ["${PCF_AZ_2}", "${PCF_AZ_1}", "${PCF_AZ_3}"]
dns_suffix          = "${PCF_DOMAIN_NAME}"
opsman_image_url    = "https://storage.googleapis.com/${OPSMAN_IMAGE}"
create_gcs_buckets  = "false"
external_database   = 0
isolation_segment   = "false"
ssl_cert            = <<SSL_CERT
$(cat ../${PCF_SUBDOMAIN_NAME}.${PCF_DOMAIN_NAME}.crt)
SSL_CERT
ssl_private_key     = <<SSL_KEY
$(cat ../${PCF_SUBDOMAIN_NAME}.${PCF_DOMAIN_NAME}.key)
SSL_KEY
service_account_key = <<SERVICE_ACCOUNT_KEY
$(cat ../gcp_credentials.json)
SERVICE_ACCOUNT_KEY
EOF
```

## Apply the PKS terraform declarions to deploy your Ops Manager VM

```bash
terraform init
terraform apply --auto-approve
```

This will take about 5 minutes to complete but you should allow some 
extra time for the DNS updates to propagate.

Once `dig` can resolve the Ops Manager FQDN to an IP address within its __AUTHORITY SECTION__, we're probably good to move on.  This may take about 5 minutes from your local machine.

```bash
watch dig ${PCF_OPSMAN_FQDN}
```

Probe the resolved FQDN to make sure we get a response.

```bash
curl ${PCF_OPSMAN_FQDN}
```

Successful DNS resolution is fully dependent on having a ${PCF_SUBDOMAIN_NAME}.${PCF_DOMAIN_NAME} NS record-set attached to your registered domain.  This record-set must point to _every_ google domain server, for example:

(screenshot from [AWS Route 53](https://aws.amazon.com/route53))

![route_53_ns](route_53_ns.png)

## Use automation tools to deploy PKS

```bash
cd ~/ops-manager-automation

./scripts/configure-authentication.sh

IMPORTED_VERSION=2.3.5 TARGET_PLATFORM=pks ./scripts/configure-director-gcp.sh

PRODUCT_NAME="Pivotal Container Service (PKS)" \
PRODUCT_VERSION="1.2.2" \
DOWNLOAD_REGEX="Pivotal Container Service" \
  ./scripts/import-product.sh

IMPORTED_NAME="pivotal-container-service" IMPORTED_VERSION="1.2.2-build.3" ./scripts/stage-product.sh
IMPORTED_NAME="pivotal-container-service" IMPORTED_VERSION="1.2.2-build.3" ./scripts/configure-product.sh

./scripts/apply-changes.sh
```

## Set shortcut variables
- Execute `PCF_OPSMAN=pcf.${PCF_SUBDOMAIN_NAME}.${PCF_DOMAIN_NAME}`
- Execute `PCF_PKS=pks.${PCF_SUBDOMAIN_NAME}.${PCF_DOMAIN_NAME}`

## Find the product guid and UAA admin password for PKS

1. Navigate to the page resolved by executing:
   
   ```bash
   echo "https://${PCF_OPSMAN}/api/v0/deployed/products"
   ```

1. Identify the guid for the product with `"type": "pivotal-container-service"`
1. Store this value in a shell variable named `PCF_PKS_GUID`
1. Navigate to the page resolved by executing:
   
   ```bash
   echo "https://${PCF_OPSMAN}/api/v0/deployed/products/${PCF_PKS_GUID}/credentials/.properties.uaa_admin_password"
   ```
   
1. Identify the value of `credential.value.secret`
1. Store this value in a shell variable named `PCF_PKS_UAA_ADMIN_PASSWORD`

## Connect to PKS

```bash
pks login \
  --api api.${PCF_PKS} \
  --username admin \
  --password ${PCF_PKS_UAA_ADMIN_PASSWORD} \
  --skip-ssl-validation
```

## Use the PKS client to create your Kubernetes cluster

Increase the value of `--num-nodes` to 3 if you'd like to use multiple availability zones:

```bash
pks create-cluster k8s \
  --external-hostname k8s.${PCF_PKS} \
  --plan small  \
  --num-nodes 1 \
  --wait
```

## Discover the external IP of the master node

```bash
K8S_MASTER_INTERNAL_IP=$(pks cluster k8s --json | jq --raw-output '.kubernetes_master_ips[0]')
K8S_MASTER_EXTERNAL_IP=$(gcloud compute instances list --project ${GCP_PROJECT_ID} --format json | \
  jq --raw-output --arg V "${K8S_MASTER_INTERNAL_IP}" '.[] | select(.networkInterfaces[].networkIP | match ($V)) | .networkInterfaces[].accessConfigs[].natIP')
```

## Create an A Record in the DNS for your Kubernetes cluster

```bash
gcloud dns record-sets transaction start --project ${GCP_PROJECT_ID} --zone=${PCF_SUBDOMAIN_NAME}-zone

  gcloud dns record-sets transaction \
    add ${K8S_MASTER_EXTERNAL_IP} \
    --name=k8s.${PCF_PKS}. \
    --project ${GCP_PROJECT_ID} \
    --ttl=300 --type=A --zone=${PCF_SUBDOMAIN_NAME}-zone

gcloud dns record-sets transaction execute --project ${GCP_PROJECT_ID} --zone=${PCF_SUBDOMAIN_NAME}-zone
```

## Verify the DNS changes

This step may take about 5 minutes to propagate.

When run locally, the following watch commands should eventually yield different external IP addresses

```bash
watch dig api.${PCF_PKS}
watch dig k8s.${PCF_PKS}
```

## Use the PKS client to cache the cluster creds

```bash
pks get-credentials k8s
```

This will create/modify the `~/.kube` directory used by the `kubectl` tool.

## Deploy the nginx webserver docker image

```bash
kubectl create -f - <<-EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF
```

## Make our app externally discoverable

Let our cluster know that we wish to expose our app via a `NodePort` service:

```bash
kubectl create -f - <<-EOF
apiVersion: v1
kind: Service
metadata:
  name: web-service
  labels:
    run: web-service
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
EOF
```

## Inspect our app

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods
kubectl get services
```

Return to the Kubernetes dashboard to inspect these resources via the web UI.

## Expose our app to the outside world

This section is more about manipulating GCP to expose an endpoint than PKS or k8s.

1. Extract the port from the your Kubernetes Service:

   ```bash
   SERVICE_PORT=$(kubectl get services --output=json | \
     jq '.items[] | select(.metadata.name=="web-service") | .spec.ports[0].nodePort')
   ```

1. Add a firewall rule for the web-service port, re-using the SERVICE_PORT for consistency:

   ```bash
   gcloud compute firewall-rules create nginx \
     --network=${PCF_SUBDOMAIN_NAME}-pcf-network \
     --action=ALLOW \
     --rules=tcp:${SERVICE_PORT} \
     --target-tags=worker \
     --project=${GCP_PROJECT_ID}
   ```
   
1. Add a target pool to represent all the worker nodes:
   
   ```bash
   gcloud compute target-pools create "nginx" \
     --region "us-central1" \
     --project "${GCP_PROJECT_ID}"
     
   WORKERS=$(gcloud compute instances list --project=${GCP_PROJECT_ID} --filter="tags.items:worker"    --format="value(name)")
   
   for WORKER in ${WORKERS}; do
     gcloud compute target-pools add-instances "nginx" \
       --instances-zone "us-central1-a" \
       --instances "${WORKER}" \
       --project "${GCP_PROJECT_ID}"
   done
   ```
   
1. Create a forwarding rule to expose our app:
   
   ```bash
   gcloud compute forwarding-rules create nginx \
     --region=us-central1 \
     --network-tier=STANDARD \
     --ip-protocol=TCP \
     --ports=${SERVICE_PORT} \
     --target-pool=nginx \
     --project=${GCP_PROJECT_ID}
   ```
   
1. Extract the external IP address:
   
   ```bash
   LOAD_BALANCER_IP=$(gcloud compute forwarding-rules list \
     --filter="name:nginx" \
     --format="value(IPAddress)" \
     --project=${GCP_PROJECT_ID})
   ```

# Verify accessibility
Navigate to the page resolved by executing:

```bash
echo http:/${LOAD_BALANCER_IP}:${SERVICE_PORT}
```
