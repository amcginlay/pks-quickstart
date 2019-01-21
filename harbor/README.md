# Harbor

## Prerequisites

- Follow [these steps](https://github.com/amcginlay/ops-manager-automation) to deploy Opsmanager and PKS on GCP.
- Export `GCP_PROJECT_ID` as an environment variable. You can do this by running:

```bash
export GCP_PROJECT_ID="$(gcloud config get-value core/project)"
```

- On the jumpbox, install Docker CE. Follow the instructions [here](https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-using-the-repository).

## Deploy Harbor

Harbor can be deployed on its own, or alongside __PAS__ or __PKS__. You can also not deploy it at all.

```bash
# install Harbor
cd ~/ops-manager-automation

PRODUCT_NAME="VMware Harbor Container Registry for PCF" \
PRODUCT_VERSION="1.7.1" \
DOWNLOAD_REGEX="^VMware Harbor" \
  ./scripts/import-product.sh
IMPORTED_NAME="harbor-container-registry" IMPORTED_VERSION="1.7.1-build.3" ./scripts/stage-product.sh
IMPORTED_NAME="harbor-container-registry" IMPORTED_VERSION="1.7.1-build.3" ./scripts/configure-product.sh

# If necessary, apply changes once configuration completes
./scripts/apply-changes.sh
```

## Configure Harbor DNS

Set Harbor's DNS name as a variable

```bash
export PCF_HARBOR="harbor.${PCF_SUBDOMAIN_NAME}.${PCF_DOMAIN_NAME}"
```

Add a DNS entry for Harbor.

```bash
# Get Harbor public IP
HARBOR_EXTERNAL_IP="$(gcloud compute instances list --project ${GCP_PROJECT_ID} --format json | jq -r \
  '.[] | select(.labels.instance_group == "harbor-app") | .networkInterfaces[].accessConfigs[].natIP')"

gcloud dns record-sets transaction start --project ${GCP_PROJECT_ID} --zone=${PCF_SUBDOMAIN_NAME}-zone

gcloud dns record-sets transaction \
  add ${HARBOR_EXTERNAL_IP} \
  --name=${PCF_HARBOR}. \
  --project ${GCP_PROJECT_ID} \
  --ttl=300 --type=A --zone=${PCF_SUBDOMAIN_NAME}-zone

gcloud dns record-sets transaction execute --project ${GCP_PROJECT_ID} --zone=${PCF_SUBDOMAIN_NAME}-zone
```

## Verify the DNS changes

This step may take about 5 minutes to propagate.

When run locally, the following watch command should eventually yield a different external IP address

```bash
watch dig "$PCF_HARBOR"
```

## Allow outside traffic to hit Harbor

Add a firewall rule to GCP to allow traffic to the Harbor VM on port 443.

```bash
gcloud compute firewall-rules create harbor \
  --network=${PCF_SUBDOMAIN_NAME}-pcf-network \
  --action=ALLOW \
  --rules=tcp:443 \
  --target-tags=harbor-app \
  --project=${GCP_PROJECT_ID}
```

You should now be able to view the Harbor UI. You can see the URL with `echo "https://${PCF_HARBOR}"`. The username is "admin". You can see the password with `echo "${OM_PASSWORD}"`

## Push a hello world app to Harbor


Pull a sample Docker image:

```bash
docker pull nginx
```

Log into Harbor:

```bash

/etc/docker/daemon.json
```

