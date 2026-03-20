
# CPD Upgrade From 5.1.3 to 5.3.1

## Upgrade Context
- **OCP:** 4.16
- **CPD:** 5.1.3 → 5.3.1
- **Storage:** ODF
- **Components:** ibm-licensing,cpfs,cpd_platform,analyticsengine,datastage_ent,datastage_ent_plus,dmc,dv,mantaflow,wkc
- **Airgapped:** Yes

# Table of Contents
- 1. Pre-upgrade
- 2. Upgrade
- 3. Post-upgrade tasks

# 1. Pre-upgrade

**Note:**
Sourcing the latest environment variables used this environment before proceeding with the following procedures. Here's an example:
```
source ./cpd_vars.sh
```

## 1.1 Pre-upgrade check

### 1.1.1 Checking the health of your cluster

```
${OC_LOGIN}
oc get nodes,co,mcp

${CPDM_OC_LOGIN}
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
cpd-cli health cluster
cpd-cli health nodes
cpd-cli health operators --operator_ns=${PROJECT_CPD_INST_OPERATORS} --control_plane_ns=${PROJECT_CPD_INST_OPERANDS}
cpd-cli health operands --control_plane_ns=${PROJECT_CPD_INST_OPERANDS}
```
### 1.1.2 Checking the Common Core services before the upgrade

- 1. Check whether Global Search configured properly
- 2. Run the `precheck_migration.sh` to determine whether you can run an automatic migration of the common core services or whether you need to configure common core services to run a semi-automatic migration.

Complete the above two checks by following the steps of the `Before you begin` section in this documentation [Pre-upgrade check for CCS](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=hub-upgrading-software#taskupgrade-instance__prereq__1_)


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

## 1.3 Install Helm CLI

[Installing Helm](https://www.ibm.com/links?url=https%3A%2F%2Fhelm.sh%2Fdocs%2Fintro%2Finstall%2F)

## 1.4 Updating your environment variables script
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

### 1.5.1 Downloading CASE packages and the cluster-scoped resource definitions for the scheduling service

Download from GitHub.

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

### 1.5.2 Downloading CASE packages and the cluster-scoped resources for the platform and services

Download from GitHub.

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

## 1.6 Mirror images

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

Mirror the images to the private container registry.
```
cpd-cli manage mirror-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--arch=${IMAGE_ARCH} \
--case_download=false
```

For each component, the command generates a log file in the work directory.Run the following command to print out any errors in the log files:
```
grep "error" mirror_*.log
```

Confirm by inspecting the contents of the private container registry:
```
cpd-cli manage list-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--case_download=false
```

The output is saved to the `list_images.csv` file in the `work/offline/${VERSION}` directory. Run below command by detecting images that are missing or that cannot be inspected.

```
grep "level=fatal" list_images.csv
```

[Mirroring images directly to the private container registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-1)

### 1.6.2 Mirroring Red Hat OpenShift certificate manager images to a private container registry
Mirror the Red Hat OpenShift certificate manager images to your private container registry before you install the certificate manager.
[Mirroring Red Hat OpenShift certificate manager images](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=manager-mirroring-red-hat-openshift-certificate-images)


# 2. Upgrade
## 2.1 Migrate to Red Hat OpenShift certificate manager

The IBM Certificate manager is deprecated.

If the IBM Certificate manager (ibm-cert-manager) is installed on your cluster, use the following steps to migrate your certificates from the IBM Certificate manager to the Red Hat OpenShift certificate manager (cert-manager Operator).

[Migrating from the IBM Certificate manager to the Red Hat OpenShift certificate manager](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading-migrating-red-hat-openshift-certificate-manager)

## 2.2 Upgrade the Scheduling service

If you're not sure whether the Scheduling service is installed on the cluster, run the following command:
```
oc get scheduling -A
```

### 2.2.1 Updating cluster-scoped resources for the Scheduling service

1.Generate the cluster-scoped resource definitions for the Scheduling service:
```
cpd-cli manage case-download \
--components=scheduler \
--release=${VERSION} \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
--cluster_resources=true
```

2.Change to the work directory. The default location of the work directory is `cpd-cli-workspace/olm-utils-workspace/work`.
<br>

3.Log in to Red Hat® OpenShift® Container Platform as a cluster administrator.

```
${OC_LOGIN}
```

4.Apply the cluster-scoped resources for the Scheduling service from the cluster_scoped_resources.yaml file:
```
oc apply -f cluster_scoped_resources.yaml \
--server-side \
--force-conflicts
```

### 2.2.2 Creating image pull secrets for the Scheduling service
1.Create a file named dockerconfig.json based on where your cluster pulls images from:

```
cat <<EOF > dockerconfig.json 
{
  "auths": {
    "${PRIVATE_REGISTRY_LOCATION}": {
      "auth": "${IMAGE_PULL_CREDENTIALS}"
    }
  }
}
EOF
```

2.Create the image pull secret in the project where the Scheduling service is installed:
```
oc create secret docker-registry ${IMAGE_PULL_SECRET} \
--from-file ".dockerconfigjson=dockerconfig.json" \
--namespace=${PROJECT_SCHEDULING_SERVICE}
```

### 2.2.3 Upgrading the Scheduling service
```
cpd-cli manage apply-scheduler \
--release=${VERSION} \
--license_acceptance=true \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET}
```

Confirm that the Scheduling service pods are Running or Completed:

```
oc get pods --namespace=${PROJECT_SCHEDULING_SERVICE}
```

## 2.3 Upgrade the License Service

### 2.3.1 Get the project of the License service

If you're not sure which project the License Service is in, run the following command:
```
oc get deployment -A | grep ibm-licensing-operator
```

### 2.3.2  Log in to the Red Hat OpenShift Container Platform cluster
```
${CPDM_OC_LOGIN}
```

### 2.3.3 Upgrade the License Service

```
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```
Confirm that the License Service pods are Running or Completed:

```
oc get pods --namespace=${PROJECT_LICENSE_SERVICE}
```

## 2.4 Prepare to upgrade IBM Software Hub

### 2.4.1 Updating the cluster-scoped resources for the platform and services

1.Generate cluster-scoped resources for platform and services

```
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cluster_resources=true
```

2.Change to the `work` directory. 
<br>
The default location of the work directory is `cpd-cli-workspace/olm-utils-workspace/work`.

```
cd cpd-cli-workspace/olm-utils-workspace/work
```

3.Log in to Red Hat® OpenShift® Container Platform as a cluster administrator
```
${OC_LOGIN}
```

4.Apply the cluster-scoped resources for the from the cluster_scoped_resources.yaml file
```
oc apply -f cluster_scoped_resources.yaml \
--server-side \
--force-conflicts
```


### 2.4.2 Applying your entitlements to monitor and report use against license terms
Applying the entitlements of `cpd-enterprise `.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cpd-enterprise \
--production=false
```
Applying the entitlements of `datastage`.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=datastage \
--production=false
```

## 2.5 Upgrade IBM Software Hub
## 2.5.1 Creating image pull secrets for an instance of IBM Software Hub
1.Log in to Red Hat® OpenShift® Container Platform as a user with sufficient permissions to complete the task.
```
${OC_LOGIN}
```

2.Create a file named dockerconfig.json based on where your cluster pulls images from.
```
cat <<EOF > dockerconfig.json 
{
  "auths": {
    "${PRIVATE_REGISTRY_LOCATION}": {
      "auth": "${IMAGE_PULL_CREDENTIALS}"
    }
  }
}
EOF
```

3.Create the image pull secret in the operators project for the instance.

```
oc create secret docker-registry ${IMAGE_PULL_SECRET} \
--from-file ".dockerconfigjson=dockerconfig.json" \
--namespace=${PROJECT_CPD_INST_OPERATORS}
```

4.Create the image pull secret in the operands project for the instance:
```
oc create secret docker-registry ${IMAGE_PULL_SECRET} \
--from-file ".dockerconfigjson=dockerconfig.json" \
--namespace=${PROJECT_CPD_INST_OPERANDS}
```
### 2.5.2 Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```
### 2.5.3 Upgrade the required operators and custom resources for the instance
```
cpd-cli manage install-components \
--license_acceptance=true \
--components=cpd_platform \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--run_storage_tests=true \
--upgrade=true
```

Once the above command `cpd-cli manage install-components` is complete, make sure the status of the IBM Software Hub is in 'Completed' status.

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \ 
--components=cpd_platform
```

### 2.5.4 Apply the RSI patches
Run the following command to re-apply your existing custom patches.
```
cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check the RSI patches status again: 
```
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --all
```

## 2.6 Upgrade WKC
### 2.6.1 
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading-preparing-upgrade-knowledge-catalog

### 2.6.2
Create the `install-options.yml` file in the cpd-cli work directory (For example: cpd-cli-workspace/olm-utils-workspace/work)
```
---
# ............................................................................
# IBM Knowledge Catalog parameters
# ............................................................................
non_olm:
  wkc:
    enableDataQuality: true
    enableKnowledgeGraph: true
    useFDB: true
```

Upgrade WKC with the custom option.

```
cpd-cli manage install-components \
--license_acceptance=true \
--components=wkc \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--param-file=/tmp/work/install-options.yml \
--upgrade=true
```

Check ccs progress first:
```
watch oc get ccs 
```

Check WKC progress:
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \ 
--components=wkc
```

## 2.7 Apply the CCS Patch2 or hot fix for 5.3.1 


## 2.8 Upgrade DataStage
```
cpd-cli manage install-components \
--license_acceptance=true \
--components=datastage_ent_plus \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```
Check DataStage status
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \ 
--components=datastage_ent_plus
```

## 2.9 Upgrade MANTA Automated Data Lineage 
```
cpd-cli manage install-components \
--license_acceptance=true \
--components=mantaflow \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```
Check MantaFlow progress
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=mantaflow
```

## 2.10 Upgrade Db2 Data Management Console
```
cpd-cli manage install-components \
--license_acceptance=true \
--components=dmc \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```
Check DataStage progress
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \ 
--components=dmc
```

## 2.11 Upgrade Data Virtualization
```
cpd-cli manage install-components \
--license_acceptance=true \
--components=dv \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```
Check DV progress
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=dv
```

Upgrading existing service instances.
```
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=u-upgrading-from-version-51-34#cli-upgrade__svc-inst__title__1
```

# 3. Post-upgrade tasks
## 3.1 Post-upgrade of WKC
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=u-upgrading-from-version-51-29#cli-upgrade__next-steps__title__1

## 3.2 Post-upgrade of DV
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=u-upgrading-from-version-51-34#cli-upgrade__next-steps__title__1

