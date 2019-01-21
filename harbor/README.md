# Harbor

## Prerequisites

- Follow [these steps](https://github.com/amcginlay/ops-manager-automation) to deploy Opsmanager and PKS on GCP.

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
```


