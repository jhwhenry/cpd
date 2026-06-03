# watsonx Services Upgrade Runbook - v.5.2.2 to 5.3.1 Patch 4

---

## Upgrade documentation

[Upgrading from IBM Cloud Pak for Data Version 5.2.2 to Version 5.3.](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading-from-version-52)

## Upgrade context

From

```
OCP: OpenShift On-prem
CPD: 5.2.2
Storage: Netapp 
Componenets: ibm-licensing, scheduler, cpfs, cpd_platform, watsonx_orchestrate, watsonx_ai, watsonx_data
```

To

```
OCP: OpenShift on-prem
CPD: 5.3.1 Patch 4
Storage: Netapp 
Componenets: ibm-licensing, scheduler, cpfs, cpd_platform, watsonx_orchestrate, watsonx_ai, watsonx_data
```

## Pre-requisites

#### 1. Backup of the cluster is done.

Backup your Cloud Pak for Data cluster before the upgrade.
For details, see [Backing up and restoring Cloud Pak for Data](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=administering-backing-up-restoring-software-hub).

#### 2. The image mirroring completed successfully

If a private container registry is in use to host the IBM Cloud Pak for Data software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry.
`<br>`
Reference:
[Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-1)

#### 3. The permissions required for the upgrade is ready

- Openshift cluster administrator permissions
- Cloud Pak for Data administrator permissions
- Permission to access the private image registry for pushing or pull images
- Access to the Bastion node for executing the upgrade commands

#### 4. A pre-upgrade health check is done to ensure the cluster's readiness for upgrade.

- The OpenShift cluster, persistent storage and Cloud Pak for Data platform and services are in healthy status.

#### 5. Backup the Routes

```
oc get Routes -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > Routes_Bak.yaml
```

#### 6. Backup the TemporaryPatch

```
oc get TemporaryPatch -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > TemporaryPatch_Bak.yaml
```
#### 7.Global Search legacy index compatibility check before upgrade

[Known Issue: Global Search Legacy Index Compatibility](https://www.ibm.com/support/pages/node/7268540#pre-upgrade-checklist)

<br>

Edit the CCS custom resource.

```
oc edit ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

Add the following properties to the CCS Custom Resource prior to initiating the upgrade:

```
opensearch_legacy_core_version: "2.19.3"
opensearch_legacy_plugin_version: "2.19.3.0"
```

## Table of Content

```
Part 1: Pre-upgrade
1.1 Update client workstation
1.1.1 Set up the utilities
1.1.2 Update environment variables
1.1.3 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image
1.1.4 Download CASE packages and mirror images
1.1.5 Create a profile for upgrading the service instances
1.2 Health check OCP & CPD


Part 2: Upgrade
2.1 Upgrade CPD to 5.3.1
2.1.1 Upgrade shared cluster components
2.1.2 Upgrade Knative Eventing
2.1.3 Upgrade Events Operator
2.1.4 Prepare to upgrade IBM Software Hub
2.1.5 Create image pull secrets for IBM Software Hub instance
2.1.6 Upgrade IBM Software Hub
2.2 Upgrade services
2.2.1 Upgrade the Watsonx.data service
2.2.2 Upgrade watsonx.ai service
2.2.3 Upgrade watsonx Orchestrate service
2.3 Upgade CPD BR service

Part 3: Post-upgrade

Summarize and close out the upgrade

```

## Part 1: Pre-upgrade

### 1.1 Update client workstation

#### 1.1.1 Set up the utilities

**1. Update the cpd-cli utility**

<br>

Update the cpd-cli utility following the steps in [Updating client workstations](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=52-updating-client-workstations)

**Note:** Download the cpd-cli v14.3.1.5 (patch 5) which can support the `--patch_id` parameter.
<br>

[cpd-cli v14.3.1.5](https://github.com/IBM/cpd-cli/releases/download/v14.3.1.5/cpd-cli-linux-EE-14.3.1.tgz)

<br>

After the update is done, run below commands for the confirmation:

```
cpd-cli version
```

Output like this

```
cpd-cli
	Version: 14.3.1
    Build Number: 3169
	SWH Release Version: 5.3.1
```

**2. Update the OpenShift CLI**
`<br>`
Check the OpenShift CLI version.

```
oc version
```

If the version doesn't match the OpenShift cluster version, update it accordingly.

**3. Install the Helm CLI**

<br>

Install Helm by following the [Helm documentation](https://www.ibm.com/links?url=https%3A%2F%2Fhelm.sh%2Fdocs%2Fintro%2Finstall%2F)

#### 1.1.2 Update environment variables

Make a copy of the environment variables script used by the existing 5.2.2 instance with the name like `cpd_vars_531.sh`.

Update the environment variables script `cpd_vars_531.sh` as follows.

```bash
vi cpd_vars_531.sh
```

1.Locate the VERSION entry and update the environment variable for VERSION.

```bash
export VERSION=5.3.1
```

2.Locate the COMPONENTS entry and confirm the COMPONENTS entry is accurate.

```bash
export COMPONENTS=ibm-licensing,scheduler,cpfs,cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_data
```

3.Add a new section called Image pull configuration to your script and add the following environment variables

```
export IMAGE_PULL_SECRET=ibm-entitlement-key
export IMAGE_PULL_CREDENTIALS=$(echo -n "$PRIVATE_REGISTRY_PULL_USER:$PRIVATE_REGISTRY_PULL_PASSWORD" | base64 -w 0)
export IMAGE_PULL_PREFIX=icr.io
```
**Note:**
`export IMAGE_PULL_PREFIX=icr.io` is a special setting for adapting to the subpaths in Verizon's image registry. The ImageDigestMirrorSet will be used for mapping the image pull request to the private image registry.

<br>

Save the changes.

Confirm that the script does not contain any errors.

```
bash ./cpd_vars_531.sh
```

Run this command to apply cpd_vars_531.sh

```
source cpd_vars_531.sh
```

#### 1.1.3 Ensure the cpd-cli manage plug-in has the latest olm-utils image

```
cpd-cli manage restart-container
```

**Note:**
<br>

Check and confirm the olm-utils-v4 container is up and running.

```
podman ps | grep -i olm-utils-v4
```

#### 1.1.4 Downloading CASE packages and mirror images

Downloading CASE packages before running IBM Software Hub upgrade commands.

**Note:**

If the CASE packages have already been downloaded when mirroring the images, this step can be skipped.

[Downloading CASE packages](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-downloading-case-packages-1)

```
export PATCH_ID=4

cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--from_oci=true \ 
--patch_id=${PATCH_ID}

```

Mirroring images directly to the private container registry

Log in to the IBM Entitled Registry registry:

```
cpd-cli manage login-entitled-registry \
${IBM_ENTITLEMENT_KEY}
```

Log in to the private container registry.

The following command assumes that you are using private container registry that is secured with credentials:

```
cpd-cli manage login-private-registry \
${PRIVATE_REGISTRY_LOCATION} \
${PRIVATE_REGISTRY_PUSH_USER} \
${PRIVATE_REGISTRY_PUSH_PASSWORD}
```

The models and optional images that are mirrored are determined by the ${IMAGE_GROUPS} variable, from the installation environment variables script.
<br>
For each model we already installed find the image group from this documentation. [Determining which models and optional images to mirror to your private container registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=information-determining-which-models-optional-images-mirror#mirror-model-list__watsonxai-models)
<br>
For example image_group for model `gpt-oss-120b` is `ibmwxGptOss120B`.

```
export IMAGE_GROUPS=<comma separated values. eg:ibmwxGptOss120B,ibmwxMinistral14BInstruct2512, ..and so on  >
```

Mirror the images to the private container registry.

```
export PATCH_ID=4

cpd-cli manage mirror-images \
--components=${COMPONENTS} \
--groups=${IMAGE_GROUPS} \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--arch=${IMAGE_ARCH} \
--case_download=false
```

The output is saved to the `list_images.csv` file in the work/offline/${VERSION} directory.
Check the output for errors:
```
grep "level=fatal" list_images.csv
```

#### 1.1.5 Creating a profile for upgrading the service instances

Create a profile on the workstation from which you will upgrade the service instances.

The profile must be associated with a Cloud Pak for Data user who has either the following permissions:

- Create service instances (can_provision)
- Manage service instances (manage_service_instances)

Click this link and follow these steps for getting it done.

[Creating a profile to use the cpd-cli management commands](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cli-creating-cpd-profile)

### 1.2 Health check OCP & CPD

Check and make sure the cluster operators, nodes, and machine configure pool are in healthy status.
Log onto bastion node and then log into OCP.

```
${OC_LOGIN}
```

Check the status of nodes, cluster operators and machine config pool.

```
oc get nodes,co,mcp
```

2. Check Cloud Pak for Data status

Log onto bastion node, and make sure IBM Cloud Pak for Data command-line interface installed properly.

Run this command in terminal and make sure the Lite and all the services' status are in Ready status.

```
${CPDM_OC_LOGIN}
```

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Run this command and make sure all pods healthy.

```
oc get po --no-headers --all-namespaces -o wide | grep -Ev '([[:digit:]])/\1.*R' | grep -v 'Completed'
```

## Part 2: Upgrade

### 2.1 Upgrade CPD to 5.3.1

#### 2.1.1 Upgrade shared cluster components (No Cert manager)

**1.Log in to the OCP cluster**

```
${CPDM_OC_LOGIN}
```

**2.Upgrade the License Service.**

Confirm the project in which the License Service is running.

```
oc get deployment -A |  grep ibm-licensing-operator
```

Make sure the project returned by the command matches the environment variable `PROJECT_LICENSE_SERVICE` in your environment variables script `cpd_vars_531.sh`.
<br>
Upgrade the License Service.

```
export PATCH_ID=4

cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--license_acceptance=true \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```

Confirm that the License Service pods are Running or Completed:

```
oc get pods --namespace=${PROJECT_LICENSE_SERVICE}
```

**3. Upgrade the scheduling service**

1)Create image pull secret for shared cluster component

Create dockerconfig.json

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

2)Create the secret

```
oc create secret docker-registry ibm-entitlement-key-scheduler \
--from-file ".dockerconfigjson=dockerconfig.json" \
--namespace=${PROJECT_SCHEDULING_SERVICE}
```

3)Generate the cluster-scoped resource definitions for the scheduling service

```
export PATCH_ID=4

cpd-cli manage case-download \
--components=scheduler \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
--cluster_resources=true
```

4)Get the path of the work directory.

```
OLM_UTILS_CONTAINER_NAME=$(podman ps --format '{{.Names}}' | grep -E '^olm-utils-play-v4$'| head -n 1)
WORK_DIR=$(podman inspect "${OLM_UTILS_CONTAINER_NAME}" 2>/dev/null | jq -r '.[0].Mounts[] | select(.Destination == "/tmp/work") | .Source' | head -n 1)
```

5)Apply the cluster-scoped resources for the scheduling service from the `cluster_scoped_resources.yaml` file.
```
oc apply -f $WORK_DIR/cluster_scoped_resources.yaml \
--server-side \
--force-conflicts
```

7)Upgrade the scheduling service
```
export PATCH_ID=4

cpd-cli manage apply-scheduler \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--license_acceptance=true \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=ibm-entitlement-key-scheduler
```

Validate Scheduling service pods are fully running post upgrade

```
oc get pods --namespace=${PROJECT_SCHEDULING_SERVICE}
```

#### 2.1.2 Upgrade Knative Eventing (For WxO and Wx.ai Cluster Only)

Generate the CRD files for IBM events operator

```
export PATCH_ID=4

cpd-cli manage case-download --release=${VERSION} --components=ibm_events_operator --patch_id=${PATCH_ID}
```

```
cpd-cli manage deploy-events-operator \
--release=${VERSION} \
--cluster_resources=true
```

Apply the resource files

```
OLM_UTILS_CONTAINER_NAME=$(podman ps --format '{{.Names}}' | grep -E '^olm-utils-play-v4$'| head -n 1)
WORK_DIR=$(podman inspect "${OLM_UTILS_CONTAINER_NAME}" 2>/dev/null | jq -r '.[0].Mounts[] | select(.Destination == "/tmp/work") | .Source' | head -n 1)
```

```
oc apply \
-f $WORK_DIR/ibm-events-operator-crds.yaml \
--server-side \
--force-conflicts
```

Relogin to the cpd-cli utility

```
${CPDM_OC_LOGIN}
```

Upgrade Knative Eventing

```
cpd-cli manage deploy-knative-eventing \
--release=${VERSION} \
--block_storage_class=${STG_CLASS_BLOCK} \
--upgrade=true
```

#### 2.1.3 Upgrade Events Operator (For WxO and Wx.ai Cluster Only)

```
cpd-cli manage deploy-events-operator \
--release=${VERSION} \
--events_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--events_operand_ns=${PROJECT_CPD_INST_OPERANDS}
```

#### 2.1.4 Prepare to upgrade IBM Software Hub

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster

```
${CPDM_OC_LOGIN}
```

2.Updating the cluster-scoped resources for the platform and services

```
export PATCH_ID=4

cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cluster_resources=true
```

Get the path of the work directory. 

```
OLM_UTILS_CONTAINER_NAME=$(podman ps --format '{{.Names}}' | grep -E '^olm-utils-play-v4$'| head -n 1)
WORK_DIR=$(podman inspect "${OLM_UTILS_CONTAINER_NAME}" 2>/dev/null | jq -r '.[0].Mounts[] | select(.Destination == "/tmp/work") | .Source' | head -n 1)
```

Log in to Red Hat® OpenShift® Container Platform as a cluster administrator.

```
${OC_LOGIN}
```

Apply the cluster-scoped resources for the from the `cluster_scoped_resources.yaml` file.

```
oc apply --server-side --force-conflicts -f $WORK_DIR/cluster_scoped_resources.yaml
```

Have a record of the resources that you generated.

```
mv cluster_scoped_resources.yaml ${VERSION}-${PROJECT_CPD_INST_OPERATORS}-cluster_scoped_resources.yaml
```

#### 2.1.5 Create image pull secrets for IBM Software Hub instance

Log in to OpenShift cluster.

```
${OC_LOGIN}
```

Generate the image pull credentials:

```
export IMAGE_PULL_CREDENTIALS=$(echo -n "$PRIVATE_REGISTRY_PULL_USER:$PRIVATE_REGISTRY_PULL_PASSWORD" | base64 -w 0)
```

Create a file named dockerconfig.json based on where your cluster pulls images from:

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

Create the image pull secret in the `operators` project for the instance.

```
oc create secret docker-registry ${IMAGE_PULL_SECRET} --from-file ".dockerconfigjson=dockerconfig.json" --namespace=${PROJECT_CPD_INST_OPERATORS}
```

Create the image pull secret in the `operands` project for the instance.

```
oc create secret docker-registry ${IMAGE_PULL_SECRET} --from-file ".dockerconfigjson=dockerconfig.json" --namespace=${PROJECT_CPD_INST_OPERANDS}
```

#### 2.1.6 Upgrade IBM Software Hub

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
${CPDM_OC_LOGIN}
```

2.Upgrade the required operators and custom resources for the instance.

```
export PATCH_ID=4

cpd-cli manage install-components \
--license_acceptance=true \
--components=cpd_platform \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--run_storage_tests=false \
--upgrade=true
```

Once the above command `cpd-cli manage install-components` complete, make sure the status of the IBM Software Hub is in 'Completed' status.

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \ 
--components=cpd_platform
```

### 2.2 Upgrade Services

#### 2.2.1 Upgrade the watsonx.data service

- Log in to the cluster

```
${CPDM_OC_LOGIN}
```

- Check for any Analytics engine hotfixes

  ```
  oc get ae analyticsengine-sample -n ${PROJECT_CPD_INST_OPERANDS} -oyaml |grep image_digests
  ```

  - If Analytic Engine patches exist, please remove patches and then proceed with next command

    ```
    oc patch ae analyticsengine-sample -n ${PROJECT_CPD_INST_OPERANDS} --type=json --patch '[{"op":"remove","path":"/spec/image_digests"}]'
    ```
- Upgrade the Service

  ```
  export PATCH_ID=4

  cpd-cli manage install-components \
  --license_acceptance=true \
  --components=watsonx_data \
  --release=${VERSION} \
  --patch_id=${PATCH_ID} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --upgrade=true
  ```
  
- Validate the upgrade

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=watsonx_data
```

#### 2.2.2 Upgrade watsonx.ai service

- Upgrade Wx.ai service

```
export PATCH_ID=4

cpd-cli manage install-components \
--license_acceptance=true \
--components=watsonx_ai \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

**Note:**
<br>
1.Apply preventative measures to avoid the problem TS022183476 - "ccs-cams-postgres and ccs-jobs-postgres pods stuck with image pulling problem"
<br>
Check the private registry location.
```
echo ${PRIVATE_REGISTRY_LOCATION}
```

Replace the `${PRIVATE_REGISTRY_LOCATION}` with the proper value returned in the above command and then patch the CCS custom resource with the right image name using below command.

```
oc patch ccs ccs-cr --type='merge' -p '
spec:
  existing_postgres_image:
    image_name: "<YOUR_PRIVATE_REGISTRY_LOCATION>/icr.io/cpopen/edb/postgresql:16.13-5.31.1-amd64@sha256:31d36e076118478fea58a90fe63632b17b6b795a84a071f0872bb410dbe059dc"
'
```

2.Monitor whether any foundation model pods stuck in `Pending` with the message "Insufficient nvidia.com/gpu".
If so, delete existing foundation model pods to release the GPU resource.

- Validate the upgrade

  ```
  cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=watsonx_ai
  ```
  
#### 2.2.3 Upgrade watsonx Orchestrate service

- Create the install-options.yml and specify below install options in it.

```
---
# ............................................................................
# watsonx Orchestrate parameters
# ............................................................................
non_olm:
  watsonxOrchestrate: 
    watsonxAI:
      watsonxaiifm: true
```

- Move install-options.yml to the work directory

```
OLM_UTILS_CONTAINER_NAME=$(podman ps --format '{{.Names}}' | grep -E '^olm-utils-play-v4$'| head -n 1)
WORK_DIR=$(podman inspect "${OLM_UTILS_CONTAINER_NAME}" 2>/dev/null | jq -r '.[0].Mounts[] | select(.Destination == "/tmp/work") | .Source' | head -n 1)
mv install-options.yml $WORK_DIR/install-options.yml
```
**Note:**
<br>
Change the file owner and group of the install-options.yml if needed. You may want to change this accordingly based on your environment.

- Clean up Events Operator Depedency

  ```
  oc delete rolebinding ibm-lakehouse-leader-election-rolebinding -n ${PROJECT_CPD_INST_OPERATORS} || true
  oc delete role ibm-uab-ads-operator-role -n ${PROJECT_CPD_INST_OPERATORS} || true
  ```
- Upgrade wxO Service

  ```
  export PATCH_ID=4
  
  cpd-cli manage install-components \
  --license_acceptance=true \
  --components=watsonx_orchestrate \
  --release=${VERSION} \
  --patch_id=${PATCH_ID} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --param-file=/tmp/work/install-options.yml \
  --upgrade=true
  ```
- Apply the preventative measures for avoiding the problem TS022191347 - wxo service upgrade stuck with the error message "WatsonxAiifm version mismatch"
  Patch WO custom resource
  
  ```
	oc patch watsonxorchestrate wo -n ${PROJECT_CPD_INST_OPERANDS} --type=merge -p '{
	  "spec": {
	    "watsonxaiifm": {
	      "version": "12.1.2"
	    },
	    "redis": {
	      "version": "1.3.1"
	    }
	  }
	}'
  ```
 	
  Clear Stale Status Messages
  ```
  oc patch watsonxorchestrate wo -n ${PROJECT_CPD_INST_OPERANDS} --subresource=status --type=merge -p '{"status":{"progressMessage":""}}'
  ```
  Restart WO Operator.
  
- Validate the upgrade

  ```
  cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=watsonx_orchestrate
  ```

### 2.3 Upgrade the cpdbr service

1.Set the `OADP_OPERATOR_NS` environment variable to the project where the OADP operator is installed:

```
export OADP_OPERATOR_NS=<oadp-operator-project>
cpd-cli oadp client config set namespace=${OADP_OPERATOR_NS}
```

2.Upgrade the cpdbr-tenant component for the instance.

```
cpd-cli oadp install \
--component=cpdbr-tenant \
--cpdbr-hooks-image-prefix=${PRIVATE_REGISTRY_LOCATION} \
--cpfs-image-prefix=${PRIVATE_REGISTRY_LOCATION} \
--namespace=${OADP_OPERATOR_NS} \
--tenant-operator-namespace=${PROJECT_CPD_INST_OPERATORS} \
--skip-recipes=true \
--upgrade=true \
--log-level=debug \
--verbose
```

**Note:**
<br>
Monitor the cpdbr pod during this command execution. 
If it stuck in the `ImagePullBackOff` status, then update the image URL in the cpdbr-tenant deployment accordingly.


## Part 3 (Post-upgrade tasks)
- Have sanity testing before releasing back to end users.

## Summarize and close out the upgrade
- Schedule a wrap-up meeting and review the upgrade procedure and lessons learned from it.
- Evaluate the outcome of upgrade with pre-defined goals.

---

End of document
