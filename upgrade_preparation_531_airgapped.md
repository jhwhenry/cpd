
# Preparation for the CPD Upgrade to 5.3.1
It basically covers the cpd-cli update, image mirroring, CASEs and cluster resources files download. 

## 1.1 Sourcing the environment variables 

Sourcing the latest environment variables used this environment before proceeding with the following procedures. Here's an example:
```
source ./cpd_vars.sh
```

## 1.2 Updating the IBM Software Hub command-line interface
### 1.2.1 Obtaining the olm-utils-v4 image

```
podman pull icr.io/cpopen/cpd/olm-utils-v4:${VERSION}.amd64 --tls-verify=false

podman login ${PRIVATE_REGISTRY_LOCATION} -u ${PRIVATE_REGISTRY_PUSH_USER} -p ${PRIVATE_REGISTRY_PUSH_PASSWORD}

podman tag icr.io/cpopen/cpd/olm-utils-v4:${VERSION}.amd64 ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v4:${VERSION}.amd64

podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v4:${VERSION}.amd64
```

### 1.2.2 Updating the IBM Software Hub command-line interface

[Update the cpd-cli utility](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=workstations-updating-software-hub-cli)

## 1.3 Updating your environment variables script
Make a copy of the environment variables script used by the existing 5.1.3 variables with the name like `cpd_vars_531.sh`.

Update the environment variables script `cpd_vars_531.sh` as follows.
```
vi cpd_vars_531.sh
```
1. Locate the VERSION entry and update the environment variable for VERSION.
```
export VERSION=5.3.1
```
2. Locate the COMPONENTS entry and confirm the COMPONENTS entry is accurate.

<br>
For example:

```
export COMPONENTS=ibm-licensing,cpfs,cpd_platform,analyticsengine,datastage_ent,datastage_ent_plus,dmc,dv,mantaflow,wkc
```
3. Add a new section called Image pull configuration to your script and add the following environment variables
```
export IMAGE_PULL_SECRET=<the name of the namespace-scoped pull secret that will contain the base64 encoded credentials for pulling images>
export IMAGE_PULL_CREDENTIALS=$(echo -n "$PRIVATE_REGISTRY_PULL_USER:$PRIVATE_REGISTRY_PULL_PASSWORD" | base64 -w 0)
export IMAGE_PULL_PREFIX=${PRIVATE_REGISTRY_LOCATION}
```
4. Locate the OLM_UTILS_IMAGE entry and update the value
```
export OLM_UTILS_IMAGE=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v4:${VERSION}.amd64
export OLM_UTILS_LAUNCH_ARGS=" --network=host"
```
5. Save the changes. 

6. Confirm that the script does not contain any errors.
```
bash ./cpd_vars_531.sh
```
7. Run this command to apply cpd_vars_531.sh
```
source ./cpd_vars_531.sh
```

Reference: [Updating your environment variables script](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=information-updating-your-environment-variables-script)

## 1.5 Downloading CASE packages and cluster-scoped resource files 

### 1.5.1 Downloading CASE packages

```
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION}
```

### 1.5.2 Downloading the cluster-scoped resource definitions for the scheduling service
This step is only required when the scheduling services installed.

```
cpd-cli manage case-download \
--components=scheduler \
--release=${VERSION} \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
--cluster_resources=true
```

Rename the `cluster_scoped_resources.yaml`

```
mv cluster_scoped_resources.yaml ${VERSION}-${PROJECT_SCHEDULING_SERVICE}-cluster_scoped_resources.yaml
```

### 1.5.3 Downloading the cluster-scoped resources for the platform and services

```
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cluster_resources=true
```

Rename the `cluster_scoped_resources.yaml`.

```
mv cluster_scoped_resources.yaml ${VERSION}-${PROJECT_CPD_INST_OPERATORS}-cluster_scoped_resources.yaml
```

## 1.6 Mirroring images

### 1.6.1 Mirroring IBM Software Hub images directly to the private container registry

Log in to the IBM Entitled registry:
```
cpd-cli manage login-entitled-registry ${IBM_ENTITLEMENT_KEY}
```
Log in to the private container registry.

The following command assumes that you are using a private container registry that is secured with credentials:
```
cpd-cli manage login-private-registry \
${PRIVATE_REGISTRY_LOCATION} \
${PRIVATE_REGISTRY_PUSH_USER} \
${PRIVATE_REGISTRY_PUSH_PASSWORD}
```

If you don't need to mirror models or optional images, run below command for the image mirroring.
```
cpd-cli manage mirror-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=127.0.0.1:12443 \
--arch=${IMAGE_ARCH} \
--case_download=false
```

If you need to mirror models or optional images, then use below commands instead.

```
export IMAGE_GROUPS = <image group of the models>
```

```
cpd-cli manage mirror-images \
--components=${COMPONENTS} \
--groups=${IMAGE_GROUPS} \
--release=${VERSION} \
--target_registry=127.0.0.1:12443 \
--arch=${IMAGE_ARCH} \
--case_download=false
```

For each component, the command generates a log file in the work directory.Run the following command to print out any errors in the log files:
```
grep "error" mirror_*.log
```

Confirm by inspecting the contents of the intermediary container registry :
```
cpd-cli manage list-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=127.0.0.1:12443 \
--case_download=false
```

The output is saved to the `list_images.csv` file in the `work/offline/${VERSION}` directory. Run below command by detecting images that are missing or that cannot be inspected.

```
grep "level=fatal" list_images.csv
```

[Mirroring images directly to the intermediary container registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-1)

### 1.6.2 Mirroring Red Hat OpenShift certificate manager images to a private container registry
If the migration to Red Hat OpenShift certificate manager required, mirror the Red Hat OpenShift certificate manager images to your private container registry before you install the certificate manager.
[Mirroring Red Hat OpenShift certificate manager images](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=manager-mirroring-red-hat-openshift-certificate-images)
