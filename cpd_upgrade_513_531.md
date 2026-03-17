
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
## 1.1 Checking the health of your cluster
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
### 1.2 Backup the RSI patches
```
cpd-cli manage get-rsi-patch-info \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--all
```

## 1.3 Update the cpd-cli utility
```
wget https://github.com/IBM/cpd-cli/releases/download/v14.3.1/cpd-cli-linux-EE-14.3.1.tgz
tar -xvf cpd-cli-linux-EE-14.3.1.tgz
```
Ensure the `cpd-cli manage` plug-in has the latest `olm-utils` image.
Check and confirm the olm-utils-v4 container is up and running.
```
cpd-cli manage restart-container
podman ps | grep olm-utils-v4
```

## 1.5 Install Helm CLI
Install Helm by following the https://www.ibm.com/links?url=https%3A%2F%2Fhelm.sh%2Fdocs%2Fintro%2Finstall%2F

```
sudo dnf install helm
```

## 1.6 Updating your environment variables script
Make a copy of the environment variables script used by the existing 5.1.3 variables with the name like cpd_vars_531.sh.

Update the environment variables script cpd_vars_531.sh as follows.
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
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cri-updating-your-environment-variables-script
```
export IMAGE_PULL_SECRET=<the name of the namespace-scoped secret that will contain the base64 encoded credentials for pulling images>
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

## 1.7 Mirror CPD 5.3.1 images
### 1.7.1 Obtaining the olm-utils-v4 image
```
podman pull cp.icr.io/cp/cpd/olm-utils-premium-v4:${VERSION}.amd64 --tls-verify=false

podman login ${PRIVATE_REGISTRY_LOCATION} -u ${PRIVATE_REGISTRY_PUSH_USER} -p ${PRIVATE_REGISTRY_PUSH_PASSWORD}

podman tag cp.icr.io/cp/cpd/olm-utils-premium-v4:${VERSION}.amd64 ${PRIVATE_REGISTRY_LOCATION}/cp/cpd/olm-utils-premium-v4:${VERSION}.amd64 

podman push ${PRIVATE_REGISTRY_LOCATION}/ cp/cpd/olm-utils-premium-v4:${VERSION}.amd64
```
### 1.7.2 Downloading CASE packages 
```
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION}
```

### 1.7.3 Mirroring images directly to the private container registry
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-1

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

cpd-cli manage mirror-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--arch=${IMAGE_ARCH} \
--case_download=false
```
Mirror the required vLLM images for Inference foundation models (ibm-watsonx-ai-ifm)
```
skopeo copy --all \
--src-username cp \
--src-password ${IBM_ENTITLEMENT_KEY} \
--src-tls-verify=false \
--dest-username ${PRIVATE_REGISTRY_PUSH_USER} \
--dest-password ${PRIVATE_REGISTRY_PUSH_PASSWORD} \
--dest-tls-verify=false \
docker://cp.icr.io/cp/cpd/vllm@sha256:cc95bc7619549a5fb9342f8c41c613df5cd65b4e1f90b408db062559a2fdcff9 \
docker://${PRIVATE_REGISTRY_LOCATION}/cp/cpd/vllm@sha256:cc95bc7619549a5fb9342f8c41c613df5cd65b4e1f90b408db062559a2fdcff9
```

## 1.8 Final checks before start the upgrade
### 1.8.1 Pre-upgade check 
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=hub-upgrading-software

### 1.8.2 Uninstall all hotfixes
Needs to check all CRs for any custom image used.

### 1.8.3 Backup the RSI patches
```
cpd-cli manage get-rsi-patch-info \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--all
```
### 1.8.4 Final health check OCP & CPD
Check OCP status
Log onto the bastion node, in the termial log into OCP and run this command.
```
oc get co
```
Make sure all the cluster operators are in AVAILABLE status. And not in PROGRESSING or DEGRADED status.
Run this command and make sure all nodes are in Ready status.
```
oc get nodes
```
Run this command and make sure all the machine configuretion pool are in a healthy status.
```
oc get mcp
```
Check Cloud Pak for Data status
Log onto the bastion node, and make sure the IBM Cloud Pak for Data command-line interface is installed properly.
Run this command in the terminal and make sure the Lite and all the services' status are in Ready status.
```
${CPDM_OC_LOGIN}
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```
Run this command and make sure all pods are healthy.
```
oc get po --no-headers --all-namespaces -o wide | grep -Ev '([[:digit:]])/\1.*R' | grep -v 'Completed'
```
Check the private container registry status if installed
Log into bastion node, where the private container registry is usually installed, as root. Run this command in the terminal and make sure it succeeds.
```
podman login --username $PRIVATE_REGISTRY_PULL_USER --password $PRIVATE_REGISTRY_PULL_PASSWORD $PRIVATE_REGISTRY_LOCATION --tls-verify=false
```
You can run this command to verify the images are in the private container registry.
```
curl -k -u ${PRIVATE_REGISTRY_PULL_USER}:${PRIVATE_REGISTRY_PULL_PASSWORD} https://${PRIVATE_REGISTRY_LOCATION}/v2/_catalog?n=6000 | jq .
```

# 2. Upgrade
## 2.1 Migrate to Red Hat OpenShift certificate manager
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading-migrating-red-hat-openshift-certificate-manager

The IBM Certificate Manager is deprecated.

### 2.1.1. Backing up your existing certificates before migrating to Red Hat OpenShift certificate manager
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=manager-backing-up-your-existing-certificates

### 2.1.2. Uninstalling IBM Certificate manager
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=manager-uninstalling-certificate

### 2.1.3. Mirroring Red Hat OpenShift certificate manager images to a private container registry
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=manager-mirroring-red-hat-openshift-certificate-images

<b>Note:</b> Make sure you can obtain your Red Hat pull secret from the Red Hat OpenShift Cluster Manager.

<b>Note:</b> Ensure that the oc-mirror plug-in V2 is installed on the client workstation. For more information, see Mirroring images for a disconnected installation by using the oc-mirror plugin v2 in the Red Hat OpenShift Container Platform documentation.

### 2.1.4. Installing the Red Hat OpenShift Container Platform cert-manager Operator
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=manager-installing-cert-operator

<b>Installing the cert-manager Operator for Red Hat OpenShift</b>

Log in to the OpenShift Container Platform web console.

Navigate to Operators → OperatorHub.

Enter `cert-manager` Operator for Red Hat OpenShift into the filter box.

Select the cert-manager Operator for Red Hat OpenShift version from Version drop-down list, and click Install.

On the Install Operator page:

Update the Update channel, if necessary. The channel defaults to `stable-v1`, which installs the latest stable release of the cert-manager Operator for Red Hat OpenShift.

Choose the Installed Namespace for the Operator. The default Operator namespace is `cert-manager-operator`.

If the `cert-manager-operator` namespace does not exist, it is created for you.

choose the `AllNamespaces` installation mode. 

Select an Update approval strategy Automatic

Verification

Navigate to Operators → Installed Operators.

Verify that cert-manager Operator for Red Hat OpenShift is listed with a Status of Succeeded in the cert-manager-operator namespace.

## 2.2 Upgrade shared cluster components
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pyc-upgrading-shared-cluster-components

### 2.2.1 If you're not sure which project the License Service is in, run the following command:
```
oc get deployment -A | grep ibm-licensing-operator
```
If you're not sure whether the scheduling service is installed on the cluster, run the following command:
```
oc get scheduling -A
```
If the scheduling service is installed, ensure that the COMPONENTS variable in your environment variables script includes the scheduler component.

### 2.2.2  Log in to the Red Hat OpenShift Container Platform cluster:
```
${CPDM_OC_LOGIN}
```
Verify install plans allow upgrade approval
```
 oc get ip -A
```
If approval is manual and approved is false, change approved to true to allow the upgrade.  One can change approved back to false after upgrade is completed.
```
oc patch installplan <installplan-name> -n <namespace> --type merge -p '{"spec":{"approved":true}}'
```
Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

### 2.2.3 Upgrade the License Service.
Confirm the project in which the License Service is running.
```
oc get deployment -A |  grep ibm-licensing-operator
```
Make sure the project returned by the command matches the environment variable PROJECT_LICENSE_SERVICE in your environment variables script `cpd_vars_531.sh`.

Upgrade the License Service.
```
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```

### 2.2.4 Confirm that the License Service pods are Running or Completed:
```
oc get pods --namespace=${PROJECT_LICENSE_SERVICE}
```

## 2.3 Creating image pull secrets for an instance of IBM Software Hub (Upgrading from Version 5.2 to Version 5.3)
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=uish-creating-image-pull-secrets-instance-1

Follow the steps from the above link. Consider the `Private container registry` option.

## 2.4 Prepare to upgrade IBM Software Hub
### 2.4.1 Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```
### 2.4.2 Updating the cluster-scoped resources for the platform and services
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=puish-updating-cluster-scoped-resources-instance
```
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cluster_resources=true
```
Change to the work directory. The default location of the work directory is `cpd-cli-workspace/olm-utils-workspace/work`.
```
cd cpd-cli-workspace/olm-utils-workspace/work
```
Log in to Red Hat® OpenShift® Container Platform as a cluster administrator.
```
${OC_LOGIN}
```
Apply the cluster-scoped resources from the `cluster_scoped_resources.yaml` file.
```
oc apply -f cluster_scoped_resources.yaml \
--server-side \
--force-conflicts
```
Have a record of the resources that you generated.
```
mv cluster_scoped_resources.yaml ${VERSION}-${PROJECT_CPD_INST_OPERATORS}-cluster_scoped_resources.yaml
```

### 2.4.3 Applying your entitlements to monitor and report use against license terms
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=aye-applying-your-entitlements-without-node-pinning-1

Applying your entitlements without node pinning
```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cpd-enterprise

cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=ikc-premium

cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=data-lineage \
--production=false
```

## 2.5 Upgrade IBM Software Hub
### 2.5.1. Run the cpd-cli manage login-to-ocp command to log in to the cluster.
```
${CPDM_OC_LOGIN}
```
### 2.5.2 Upgrade the required operators and custom resources for the instance.
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=uish-upgrading-software-hub

See all available license URLs
```
cpd-cli manage get-license --release=${VERSION}


cpd-cli manage install-components \
--license_acceptance=true \
--components=cpd_platform \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--run_storage_tests=false \
--upgrade=true
```
Once the above command `cpd-cli manage install-components` is complete, make sure the status of the IBM Software Hub is in 'Completed' status.
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \ 
--components=cpd_platform
```
### 2.5.3 Apply the RSI patches
Run the following command to re-apply your existing custom patches.
```
cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```
Check the RSI patches status again: 
```
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --all

cat $CPD_CLI_WORK_DIR/get_rsi_patch_info.log
```

## 2.6 Upgrade WKC
Create the install-options.yml file in the cpd-cli work directory (For example: cpd-cli-workspace/olm-utils-workspace/work)
<br><b>These need to be checked</b>
```
---
# ............................................................................
# IBM Knowledge Catalog Premium parameters
# ............................................................................
non_olm:
  ikcPremium:
    enableDataQuality: False
    enableKnowledgeGraph: False
    useFDB: False
    enableAISearch: False
    enableSemanticAutomation: False
    enableSemanticEnrichment: True
    enableSemanticEmbedding: False
    enableTextToSql: False
    enableModelsOn: 'cpu'
    customModelTextToSQL: granite-3-3-8b-instruct
```

Upgrade with the custom option
```
cpd-cli manage install-components \
--license_acceptance=true \
--components=${IKC_TYPE} \
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
Check WKC Premium progress:
```
oc get ikc_premium
```

## 2.7 Upgrade DataLineage
```
cpd-cli manage install-components \
--license_acceptance=true \
--components=datalineage \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```
Check DataLineage progress:
```
oc get datalineage
```


## 2.8 Upgrade DataStage
```
cpd-cli manage install-components \
--license_acceptance=true \
--components=datastage_ent \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```
Check DataStage progress
```
oc get DataStage
```

# 3. Post-upgrade tasks
RSI patches, hotfix links, and DataStage patches.

