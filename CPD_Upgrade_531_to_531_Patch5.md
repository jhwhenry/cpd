# CPD Upgrade From 5.3.1 to 5.3.1 Patch 5

---

## Upgrade documentation

[Upgrading from IBM Cloud Pak for Data Version 5.3.1 to Version 5.3.1 Patch 5](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading-from-version-53)

## Upgrade context

```
OCP: OpenShift On-prem
CPD: 5.3.1 To 5.3.1 Patch 5
Storage: Spectrum Scale Native
Componenets: ibm-licensing,cpfs,cpd_platform,datastage_ent_plus,ws_pipelines,watsonx_dataintelligence
```

## Table of Content

```
Part 1: Pre-upgrade
1.1 Update client workstation
1.1.1 Set up the utilities
1.1.2 Update environment variables
1.1.3 Creating a profile for upgrading the service instances
1.2 Health check OCP & CPD

Part 2: Upgrade
2.1 Upgrade CPD to 5.3.1
2.1.1 Upgrade shared cluster components
2.1.2 Updating the cluster-scoped resources for the platform and services
2.1.3 Applying your entitlements without node pinning
2.1.4 Upgrading IBM Software Hub
2.2 Upgrade services
2.2.1 Upgrade the datastage_ent_plus
2.2.2 Upgrade the ws_pipelines
2.2.3 Upgrade the watsonx_dataintelligence
```

## Part 1: Pre-upgrade

### 1.1 Update client workstation

#### 1.1.1 Set up the utilities

**1. Update the cpd-cli utility**

Download Version 14.3.1 of the cpd-cli from the [IBM/cpd-cli](https://github.com/IBM/cpd-cli/releases)repository on GitHub
```

wget https://github.com/IBM/cpd-cli/releases/download/v14.3.1.5/cpd-cli-linux-EE-14.3.1.tgz

tar -xvf cpd-cli-linux-EE-14.3.1.tgz
cd cpd-cli-linux-EE-14.3.1-3169
```

Ensure the cpd-cli manage plug-in has the latest olm-utils image

```
./cpd-cli manage restart-container
```

**Note:**
<br>

Check and confirm the olm-utils-v4 container is up and running.

```
podman ps | grep -i olm-utils-v4
```

#### 1.1.2 Update environment variables

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
export COMPONENTS=ibm-licensing,cpfs,cpd_platform,datastage_ent_plus,ws_pipelines,watsonx_dataintelligence
```

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

#### 1.1.3 Creating a profile for upgrading the service instances

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

#### 2.1.1 Upgrade shared cluster components

**1.Log in to the OCP cluster**

```
${CPDM_OC_LOGIN}
```

**2.Upgrade the License Service.**

https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pyc-upgrading-shared-cluster-components-1

Confirm the project in which the License Service is running.

```
oc get deployment -A |  grep ibm-licensing-operator
```

```
oc get scheduling -A
```

Make sure the project returned by the command matches the environment variable `PROJECT_LICENSE_SERVICE` in your environment variables script `cpd_vars_531.sh`.
<br>
Upgrade the License Service.

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

#### 2.1.2 Updating the cluster-scoped resources for the platform and services
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=puish-updating-cluster-scoped-resources-instance-1

1)Download the CASE package from GitHub (github.com/IBM)
```
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cluster_resources=true
```

2)Get the path of the work directory.

```
OLM_UTILS_CONTAINER_NAME=$(podman ps --format '{{.Names}}' | grep -E '^olm-utils-play-v4$'| head -n 1)
WORK_DIR=$(podman inspect "${OLM_UTILS_CONTAINER_NAME}" 2>/dev/null | jq -r '.[0].Mounts[] | select(.Destination == "/tmp/work") | .Source' | head -n 1)
```

3)Apply the cluster-scoped resources for the scheduling service from the `cluster_scoped_resources.yaml` file.
```
oc apply -f $WORK_DIR/cluster_scoped_resources.yaml \
--server-side \
--force-conflicts
```

#### 2.1.3 Applying your entitlements without node pinning

Apply the the IBM Cloud Pak for Data Enterprise Edition Non-production license
```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cpd-enterprise \
--production=false
```

Apply the IBM DataStage Enterprise Plus Cartridge license for the non-production environment.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=datastage-plus \
--production=false
```

Apply the watsonx.data intelligence Non-production license
```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=${LICENSE_NAME} \
--production=false
```

#### 2.1.4 Upgrading IBM Software Hub

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
${CPDM_OC_LOGIN}
```

2.Upgrade the required operators and custom resources for the instance.

```
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

Once the above command `cpd-cli manage install-components` complete, make sure the status of the IBM Software Hub is in 'Completed' status.

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \ 
--components=cpd_platform
```

### 2.2 Upgrade Services

#### 2.2.1 Upgrade the datastage_ent_plus

- Log in to the cluster

```
${CPDM_OC_LOGIN}
```
Update the operator and custom resource for DataStage.
```
export PATCH_ID=5
```
```
cpd-cli manage install-components \
--license_acceptance=true \
--components=datastage_ent_plus \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

#### 2.2.2 Upgrade the ws_pipelines

```
export PATCH_ID=5
```
```
cpd-cli manage install-components \
--license_acceptance=true \
--components=ws_pipelines \
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
  --components=watsonx_ai
  ```

#### 2.2.3 Upgrade the watsonx_dataintelligence
https://www.ibm.com/docs/en/software-hub/5.3.x?topic=u-upgrading-from-version-53-31

```
export PATCH_ID=5
```
```
cpd-cli manage install-components \
--license_acceptance=true \
--components=watsonx_dataintelligence \
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
  --components=watsonx_ai
  ```
---

End of document
