# watsonx Upgrade Runbook - v.5.2.1 to 5.2.2

---

## Upgrade documentation

[Upgrading from IBM Cloud Pak for Data Version 5.2.1 to Version 5.2.2](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=upgrading-from-version-52)

## Upgrade context

From

```
OCP: OpenShift Dedicated 4.17.26 on Google Cloud
CPD: 5.2.1
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Componenets: cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway
```

To

```
OCP: OpenShift Dedicated 4.17.26 on Google Cloud 
CPD: 5.2.2
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Componenets: cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway
```

## Pre-requisites

#### 1. Backup of the cluster is done.

Backup your Cloud Pak for Data cluster before the upgrade.
For details, see [Backing up and restoring Cloud Pak for Data](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=administering-backing-up-restoring-software-hub).

**Note:**
Some services don't support the offline OADP backup. Review the backup documentation and take the dedicate approach when necessary.

#### 2. The image mirroring completed successfully

If a private container registry is in-use to host the IBM Cloud Pak for Data software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry. 
<br>
Reference: 
[Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=registry-mirroring-images-private-container)


#### 3. The permissions required for the upgrade is ready

- Openshift cluster administrator permissions
- Cloud Pak for Data administrator permissions
- Permission to access the private image registry for pushing or pull images
- Access to the Bastion node for executing the upgrade commands

#### 4. A pre-upgrade health check is made to ensure the cluster's readiness for upgrade.

- The OpenShift cluster, persistent storage and Cloud Pak for Data platform and services are in healthy status.

## Table of Content

```
Part 1: Pre-upgrade
1.1 Collect information and review upgrade runbook
1.1.1 Review the upgrade runbook
1.1.2 Backup before upgrade
1.1.3 Uninstall all hotfixes and apply preventative measures
1.1.4 Uninstall the old RSI pathch
1.2 Set up client workstation 
1.2.1 Prepare a client workstation
1.2.2 Update cpd_vars.sh for the upgrade to Version 5.1.1
1.2.3 Obtain the olm-utils-v3 available
1.2.4 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image
1.2.5 Ensure the images were mirrored to the private container registry
1.2.6 Creating a profile for upgrading the service instances
1.3 Health check OCP & CPD

Part 2: Upgrade
2.1 Upgrade CPD to 5.1.1
2.1.1 Upgrading shared cluster components
2.1.2 Preparing to upgrade the CPD instance to IBM Software Hub
2.1.3 Upgrading to IBM Software Hub
2.1.4 Upgrading the operators for the services
2.1.5 Applying the RSI patches
2.2 Upgrade CPD services
2.2.1 Upgrading IBM Knowledge Catalog service and apply hot fixes
2.2.2 Upgrading MANTA service
2.2.3 Upgrading Analytics Engine service
2.2.4 Upgrading Watson Studio, Watson Studio Runtimes, Watson Machine Learning and OpenScale
2.2.5 Upgrading Db2 Warehouse

Part 3: Post-upgrade
3.1 Validate the external vault connection setting 
3.2 CCS post-upgrade tasks
3.3 WKC post-upgrade tasks

Part 4: Maintenance
4.1 Migrating from MANTA Automated Data Lineage to IBM Manta Data Lineage


Summarize and close out the upgrade

```

## Part 1: Pre-upgrade

### 1.1 Set up client workstation

#### 1.1.1 Prepare a client workstation

1. Prepare a RHEL 9 machine with internet

Create a directory for the cpd-cli utility.

```
export CPD_WORKSPACE=/ibm/cpd/522
mkdir -p ${CPD_WORKSPACE}
cd ${CPD_WORKSPACE}
```

Download the cpd-cli for 5.2.2

```
wget https://github.com/IBM/cpd-cli/releases/download/v14.2.2/cpd-cli-linux-EE-14.2.2.tgz
```

2. Install tools.

```
yum install openssl httpd-tools podman skopeo wget -y
```

The version in below commands may need to be updated accordingly.

```
tar xvf cpd-cli-linux-EE-14.2.2.tgz
mv cpd-cli-linux-EE-14.2.2-2727/* .
rm -rf cpd-cli-linux-EE-14.2.2-2727
```

3. Copy the cpd_vars.sh file used by the CPD 5.2.2 to the folder ${CPD_WORKSPACE}.

```
cd ${CPD_WORKSPACE}
cp <the file path of the cpd_vars.sh file used by the CPD 5.2.1 > cpd_vars_522.sh
```

4. Make cpd-cli executable anywhere

```
vi cpd_vars_522.sh
```

Add below two lines into the head of cpd_vars_522.sh

```
export CPD_WORKSPACE=/ibm/cpd/522
export PATH=${CPD_WORKSPACE}:$PATH
```

Update the CPD_CLI_MANAGE_WORKSPACE variable

```
export CPD_CLI_MANAGE_WORKSPACE=${CPD_WORKSPACE}
```

Run this command to apply cpd_vars_522.sh

```
source cpd_vars_522.sh
```

Check out with this commands

```
cpd-cli version
```

Output like this

```
cpd-cli
	Version: 14.2.2
	Build Date: 2025-10-25T13:04:21
	Build Number: 2727
	SWH Release Version: 5.2.2
```

5.Update the OpenShift CLI
<br>
Check the OpenShift CLI version.

```
oc version
```

If the version doesn't match the OpenShift cluster version, update it accordingly.

#### 1.1.2 Update environment variables for the upgrade to Version 5.2.2

```
vi cpd_vars_522.sh
```

1.Locate the VERSION entry and update the environment variable for VERSION.

```
export VERSION=5.2.2
```

2.Locate the COMPONENTS entry and confirm the COMPONENTS entry is accurate.

```
export COMPONENTS=ibm-licensing,cpfs,cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway
```

Save the changes. <br>

Confirm that the script does not contain any errors.

```
bash ./cpd_vars_522.sh
```

Run this command to apply cpd_vars_522.sh

```
source cpd_vars_522.sh
```

#### 1.1.3 Obtaining the olm-utils-v3 image

**Note:** If the bastion node is internet connected, then you can ignore below steps in this section.

```
podman pull icr.io/cpopen/cpd/olm-utils-v3:latest --tls-verify=false

podman login ${PRIVATE_REGISTRY_LOCATION} -u ${PRIVATE_REGISTRY_PULL_USER} -p ${PRIVATE_REGISTRY_PULL_PASSWORD}

podman tag icr.io/cpopen/cpd/olm-utils-v3:latest ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v3:latest

podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v3:latest --remove-signatures 

export OLM_UTILS_IMAGE=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v3:latest
export OLM_UTILS_LAUNCH_ARGS=" --network=host"

```

For details please refer to IBM documentation [Obtaining the olm-utils-v3 image](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=pruirn-obtaining-olm-utils-v3-image)

#### 1.1.4 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image

```
cpd-cli manage restart-container
```

**Note:**
<br>Check and confirm the olm-utils-v3 container is up and running.

```
podman ps | grep olm-utils-v3
```

#### 1.1.5 Creating a profile for upgrading the service instances

Create a profile on the workstation from which you will upgrade the service instances. 

The profile must be associated with a Cloud Pak for Data user who has either the following permissions:

- Create service instances (can_provision)
- Manage service instances (manage_service_instances)

Click this link and follow these steps for getting it done.

[Creating a profile to use the cpd-cli management commands](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=cli-creating-cpd-profile)

### 1.2 Health check OCP & CPD

Check and make sure the cluster operators, nodes, and machine configure pool are in healthy status.
<br>
Log onto bastion node, in the termial log into OCP and run this command.

```
oc get nodes,co,mcp
```

2. Check Cloud Pak for Data status

Log onto bastion node, and make sure IBM Cloud Pak for Data command-line interface installed properly.

Run this command in terminal and make sure the Lite and all the services' status are in Ready status.

```
{CPDM_OC_LOGIN}
```

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Run this command and make sure all pods healthy.

```
oc get po --no-headers --all-namespaces -o wide | grep -Ev '([[:digit:]])/\1.*R' | grep -v 'Completed'
```


## Part 2: Upgrade

### 2.1 Upgrade CPD to 5.2.2

#### 2.1.1 Upgrading shared cluster components

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster

```
${CPDM_OC_LOGIN}
```

2.Upgrade the License Service.

Confirm the project in which the License Service is running.
<br>

```
oc get deployment -A |  grep ibm-licensing-operator
```

Make sure the project returned by the command matches the environment variable PROJECT_LICENSE_SERVICE in your environment variables script `cpd_vars_522.sh`.
<br>
Upgrade the License Service.

```
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```

**Note**:
<br>
Monitor the install plan and approved them as needed.
<br>
In another terminal, keep running below command and monitoring "InstallPlan" to find which one need manual approval.

```
watch "oc get ip -n ${PROJECT_CPD_INST_OPERATORS} -o=jsonpath='{.items[?(@.spec.approved==false)].metadata.name}'"
```

Approve the upgrade request and run below command as soon as we find it.

```
oc patch installplan $(oc get ip -n ${PROJECT_CPD_INST_OPERATORS} -o=jsonpath='{.items[?(@.spec.approved==false)].metadata.name}') -n ${PROJECT_CPD_INST_OPERATORS} --type merge --patch '{"spec":{"approved":true}}'
```

Confirm that the License Service pods are Running or Completed::

```
oc get pods --namespace=${PROJECT_LICENSE_SERVICE}
```

3.Upgrade the scheduling service
<br>
Confirm whether the scheduling service is installed on the cluster.

```
oc get scheduling -A
```

If the scheduling service is installed, the command returns information about the project where the scheduling service is installed and the version that is installed. Ensure that the COMPONENTS variable in your environment variables script includes the scheduler component.
<br>
If the scheduling service is not installed, the command returns an empty response.

#### 2.1.2 Preparing to upgrade the CPD instance to IBM Software Hub

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster

```
${CPDM_OC_LOGIN}
```

2.Applying your entitlements to monitor and report use against license terms
<br>
**Non-Production enironment**
<br>
Apply the IBM Cloud Pak for Data Enterprise Edition for the non-production environment.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cpd-enterprise \
--production=false
```

Apply the watsonx.ai license for the non-production environment.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-ai \
--production=false
```

Apply the watsonx.governance license for the non-production environment.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-gov-mm \
--production=false
```

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-gov-rc \
--production=false
```

Apply the watsonx Orchestrate license for the non-production environment.
```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-orchestrate \
--production=false
```

Apply the watson Speech license.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=speech-to-text
```

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=text-to-speech
```

Reference: 
<br>

[Applying your entitlements](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=aye-applying-your-entitlements-without-node-pinning-3)

#### 2.1.3 Upgrading to IBM Software Hub

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
${CPDM_OC_LOGIN}
```

2.Upgrade the required operators and custom resources for the instance.

```
cpd-cli manage setup-instance \
--release=${VERSION} \
--license_acceptance=true \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

In another terminal, keep running below command and monitoring "InstallPlan" to find which one need manual approval.

```
watch "oc get ip -n ${PROJECT_CPD_INST_OPERATORS} -o=jsonpath='{.items[?(@.spec.approved==false)].metadata.name}'"
```

Approve the upgrade request and run below command as soon as we find it.

```
oc patch installplan $(oc get ip -n ${PROJECT_CPD_INST_OPERATORS} -o=jsonpath='{.items[?(@.spec.approved==false)].metadata.name}') -n ${PROJECT_CPD_INST_OPERATORS} --type merge --patch '{"spec":{"approved":true}}'
```

Once the above command `cpd-cli manage setup-instance` complete, make sure the status of the IBM Software Hub is in 'Completed' status.

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \ 
--components=cpd_platform
```

#### 2.1.4 Upgrade the operators for the services

```
cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--upgrade=true
```

In another terminal, keep running below command and monitoring "InstallPlan" to find which one need manual approval.

```
watch "oc get ip -n ${PROJECT_CPD_INST_OPERATORS} -o=jsonpath='{.items[?(@.spec.approved==false)].metadata.name}'"
```

Approve the upgrade request and run below command as soon as we find it.

```
oc patch installplan $(oc get ip -n ${PROJECT_CPD_INST_OPERATORS} -o=jsonpath='{.items[?(@.spec.approved==false)].metadata.name}') -n ${PROJECT_CPD_INST_OPERATORS} --type merge --patch '{"spec":{"approved":true}}'
```

Confirm that the operator pods are Running or Copmleted:

```
oc get pods --namespace=${PROJECT_CPD_INST_OPERATORS}
```

Check the version for both CSV and Subscription and ensure the CPD Operators have been upgraded successfully.

```
oc get csv,sub -n ${PROJECT_CPD_INST_OPERATORS}
```

[Operator and operand versions](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=planning-operator-operand-versions)

#### 2.1.5 Applying the RSI patches

1).Log the cpd-cli in to the Red Hat OpenShift Container Platform cluster.

```
${CPDM_OC_LOGIN}
```

2)Enable the zen-rsi-evictor-cron-job CronJob:

```
oc patch CronJob zen-rsi-evictor-cron-job \
--namespace=${PROJECT_CPD_INST_OPERANDS} \
--type=merge \
--patch='{"spec":{"suspend": false}}'
```

3).Run the following command to re-apply your existing custom patches.

```
cpd-cli manage apply-rsi-patches \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

4).Creat new patches required for migrating profiling results
<br>
a).Identify the location of the `work` directory and create the `rsi` folder under it.

```
podman inspect olm-utils-play-v3 | grep -i -A5  mounts
```

The `Source` property value in the output is the location of the `work` directory.

```
  "Mounts": [
       {
            "Type": "bind",
            "Source": "/ibm/cpd/511/work",
            "Destination": "/tmp/work",
            "Driver": "",
```

For example, `/ibm/cpd/511/work` is the location of the `work` directory.

<br>

Create the `rsi` folder. **Note: Change the value for the environment variable `CPD_CLI_WORK_DIR` based on the location of the `work` directory.**

```
export CPD_CLI_WORK_DIR=/ibm/cpd/511/work
mkdir -p $CPD_CLI_WORK_DIR/rsi
```

b).Create a json patch file named `annotation-spec.json` under the `rsi` directory with the following content:

```
[{"op":"add","path":"/metadata/annotations/io.kubernetes.cri-o.TrySkipVolumeSELinuxLabel","value":"true"}]
```

c).Create a json patch file named `specpatch.json` under the `rsi` directory with the following content:

```
[{"op":"add","path":"/spec/runtimeClassName","value":"selinux"}]
```

d).Create the annotation patch for wdp profiling postgres migration pods.

```
cpd-cli manage create-rsi-patch --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --patch_type=rsi_pod_annotation --patch_name=prof-pg-migration-annotation-selinux --description="This is annotation patch is for selinux relabeling disabling on CSI based storages for wdp profiling postgres migration pods" --include_labels=job-name:wdp-profiling-postgres-migration --state=active --spec_format=json --patch_spec=/tmp/work/rsi/annotation-spec.json
```

e).Create the spec patch for wdp profiling postgres migration pods.

```
cpd-cli manage create-rsi-patch --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --patch_type=rsi_pod_spec --patch_name=prof-pg-migration-runtimes-pod-spec-selinux --description="This is spec patch is for selinux relabeling disabling on CSI based storages for wdp profiling postgres migration pods" --include_labels=job-name:wdp-profiling-postgres-migration --state=active --spec_format=json --patch_spec=/tmp/work/rsi/specpatch.json
```

4).Check the RSI patches status again:

```
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --all

cat $CPD_CLI_WORK_DIR/get_rsi_patch_info.log
```

### 2.2 Upgrade CPD services to 5.1.1

#### 2.2.1 Upgrading IBM Knowledge Catalog service and apply customizations

Check if the IBM Knowledge Catalog service was installed with the custom install options.

##### 1. For custom installation, check the previous install-options.yaml or wkc-cr yaml, make sure to keep original custom settings

Specify the following options in the `install-options.yml` file in the `work` directory. Create the `install-options.yml` file if it doesn't exist in the `work` directory.

```
################################################################################
# IBM Knowledge Catalog parameters
################################################################################
custom_spec:
  wkc:
    enableKnowledgeGraph: True
    enableDataQuality: False
    useFDB: True
```

**Note:**
<br>
1)Make sure you edit or create the `install-options.yml` file in the right `work` folder.

<br>

Identify the location of the `work` folder using below command.

```
podman inspect olm-utils-play-v3 | grep -i -A5  mounts
```

The `Source` property value in the output is the location of the `work` folder.

<br>

2)Make sure the `useFDB` is set to be `True` in the install-options.yml file.
<br>

##### 2.Upgrade WKC with custom installation

Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
${CPDM_OC_LOGIN}
```

Update the custom resource for IBM Knowledge Catalog.

```
cpd-cli manage apply-cr \
--components=wkc \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--param-file=/tmp/work/install-options.yml \
--license_acceptance=true \
--upgrade=true
```

##### 3.Validate the upgrade

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

##### 4.Apply the customizations

**1).Apply the change for supporting CyberArk Vault with a private CA signed certificate**: <br>

```
oc patch ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge -p '{"spec":{"vault_bridge_tls_tolerate_private_ca": true}}'
```

**2)Resolve Mismatch from Catalog API**

```
oc project ${PROJECT_CPD_INST_OPERANDS}
```

[Run the script for resolving the mismatch from Catalog API](https://github.com/jhwhenry/cpd/blob/main/delete_rabbitmq_queues.sh.zip)

**3)Combined CCS patch command** (Reducing the number of operator reconcilations): <br>

- Configuring reporting settings for IBM Knowledge Catalog.

```
oc patch configmap ccs-features-configmap -n ${PROJECT_CPD_INST_OPERANDS} --type=json -p='[{"op": "replace", "path": "/data/enforceAuthorizeReporting", "value": "true"},{"op": "replace", "path": "/data/defaultAuthorizeReporting", "value": "true"}]'
```

- Apply the patch for 1)asset-files-api deployment tuning and 2)Couchdb search container resource tuning

```
oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type=merge -p '{
   "spec":{
      "image_digests":{
         "wdp_connect_connection_image":"sha256:02826fa27eed4813f62bce2eccd66ee8ab17c2ee56df811551746d683aa7ae0f",
         "wdp_connect_connector_image":"sha256:c85fcfadda98e2f7d193b12234dbec013105e50b9f59f157505c28f5e572edcc",
         "wdp_connect_flight_image":"sha256:cda30760185008c723a87bd251f60cb6402f4814ee1523c99a167ad979c5919b",
         "catalog_api_image":"sha256:b03bd07d86992a057194c369d9b4211d14fdcf726e954af7a4ddc70caa30749e",
         "catalog_api_jobs_image":"sha256:29dbaa7d9b4e6c19424b05e35e31db7cee1d66c06a0d633e56f5ad96f5786dab",
	 "portal_catalog_image":"sha256:cb6cabfc370214ed4d23a778414188b671b6efc3f0f6c74a7d0be4a2a89a0200"
      },
      "catalog_api_jobs_resources":{
         "requests":{
            "cpu":"20m",
            "ephemeral-storage":"50Mi",
            "memory":"256Mi"
         },
         "limits":{
            "cpu":"300m",
            "ephemeral-storage":"500Mi",
            "memory":"2Gi"
         }
      },
      "wdp-connect-connection_resources":{
         "requests":{
            "cpu":"150m",
            "memory":"650Mi"
         },
         "limits":{
            "cpu":"1",
            "memory":"4Gi"
         }
      },
      "asset_files_call_socket_timeout_ms":60000,
      "asset_files_api_resources":{"limits":{"cpu":"4","memory":"32Gi","ephemeral-storage":"1Gi"},"requests":{"cpu":"200m","memory":"256Mi","ephemeral-storage":"10Mi"}},
      "asset_files_api_replicas":6,
      "asset_files_api_command":["/bin/bash"],
      "asset_files_api_args":["-c","cd /home/node/${MICROSERVICENAME};source /scripts/exportSecrets.sh;export npm_config_cache=~node;node --max-old-space-size=12288 --max-http-header-size=32768 index.js"]
   }
}'
```

**4)Combined WKC patch command** (Reducing the number of operator reconcilations): <br>

- Figure out a proper PVC size for the PostgreSQL used by profiling migration.
  <br>
  Check the asset-files-api pvc size. Specify the same or a bigger storage size for preparing the postgresql with the proper storage size to accomendate the profiling migration.
  <br>
  Get the file-api-claim pvc size.

```
oc get pvc -n ${PROJECT_CPD_INST_OPERANDS} | grep file-api-claim | awk '{print $4}'
```

Specify the same or a bigger storage size for postgres storage accordingly in next step.

- Patch the WKC : a)Setting a proper PVC size for PostgreSQL (profiling db) and b) Hotfix for WKC BI Data, Lineage Performance, wkc-gov-ui

```
oc patch wkc wkc-cr -n ${PROJECT_CPD_INST_OPERANDS} --type=merge -p '{
   "spec":{
      "wdp_profiling_edb_postgres_storage_size":"1500Gi",
      "wkc_bi_data_service_liveness_probe_failure_threshold":3,
      "wkc_bi_data_service_liveness_probe_period_seconds":30,
      "wkc_bi_data_service_readiness_probe_failure _threshold":3,
      "wkc_bi_data_service_readiness_probe_period_seconds":30,
      "image_digests":{
         "wkc_bi_data_service_image":"sha256:430e1ba3203de4ef896834d17ef76bc69375b3dd5602e237b5500a535b208775",
         "wkc_data_lineage_service_image":"sha256:bc0a37a460f383f9a5fce0f7decd0a074db83b9df56d541f61835ea32a486c88",
         "wdp_kg_ingestion_service_image":"sha256:349f6cf2e36388afe336ac6f9119a64c2be1804a3c255a6d538cc125e2507ae0",
         "wkc_metadata_imports_ui_image":"sha256:8187a39d259037be66eebc2377e7300ec38afe4b5bf250bc0bffdfa370cc42c7"
      },
      "wkc_gov_ui_image":{
         "name":"wkc-gov-ui@sha256",
         "tag":"f88bbdee4c723e96ba72584f186da8a1618bd1234d5e7dc32a007af3b250a5e6",
         "tag_metadata":"5.1.1501-amd64"
      }
   }
}'
```

#### 2.2.2 Upgrading MANTA service

```
export COMPONENTS=mantaflow

```

- Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
${CPDM_OC_LOGIN}
```

- Run the command for upgrading MANTA service.

```
cpd-cli manage apply-cr \
--components=mantaflow \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--upgrade=true
```

Validating the upgrade.

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=mantaflow
```

#### 2.2.3 Upgrading Analytics Engine service

##### 2.2.3.1 Upgrading the service

Check the Analytics Engine service version and status.

```
export COMPONENTS=analyticsengine

cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=${COMPONENTS}
```

The Analytics Engine serive should have been upgraded as part of the WKC service upgrade. If the Analytics Engine service version is **not 5.1.1**, then run below commands for the upgrade. <br>

Check if the Analytics Engine service was installed with the custom install options. <br>

```
cpd-cli manage apply-cr \
--components=analyticsengine \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--upgrade=true
```

Validate the service upgrade status.

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=${COMPONENTS}
```

##### 2.2.3.2 Upgrading the service instances

**Note:**  cpd profile api key may expire after upgrade. If we are not able to list the instances, should be attempted once the Custom route is created so that the Admin can login.
<br>
Find the proper CPD user profile to use.

```
cpd-cli config profiles list
```

Upgrade the Spark service instance

```
cpd-cli service-instance upgrade \
--service-type=spark \
--profile=${CPD_PROFILE_NAME} \
--all
```

Validate the service instance upgrade status.

```
cpd-cli service-instance list \
--service-type=spark \
--profile=${CPD_PROFILE_NAME}
```

#### 2.2.4 Upgrading Watson Studio, Watson Studio Runtimes, Watson Machine Learning and OpenScale

```
export COMPONENTS=ws,ws_runtimes,wml,openscale
```

Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
${CPDM_OC_LOGIN}
```

Run the upgrade command.

```
cpd-cli manage apply-cr \
--components=${COMPONENTS}  \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--upgrade=true
```

Validate the service upgrade status.

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=${COMPONENTS}
```

#### 2.2.5 Upgrading Db2 Warehouse

```
export COMPONENTS=db2wh
```

Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
${CPDM_OC_LOGIN}
```

Run the upgrade command.

```
cpd-cli manage apply-cr \
--components=db2wh \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--upgrade=true
```

Validate the service upgrade status.

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=${COMPONENTS}
```

Upgrading Db2 Warehouse service instances:
<br>

- Get a list of your Db2 Warehouse service instances

```
cpd-cli service-instance list \
--service-type=db2wh \
--profile=${CPD_PROFILE_NAME}
```

- Upgrade Db2 Warehouse service instances
  <br>
  Run the following command to check whether your Db2 Warehouse service instances is in running state(You can refer to the web console for getting the service instance name) :

```
cpd-cli service-instance status ${INSTANCE_NAME} \ 
--profile=${CPD_PROFILE_NAME} \ 
--service-type=db2wh
```

Upgrade the service instance:

```
cpd-cli service-instance upgrade --profile=${CPD_PROFILE_NAME} --instance-name=${INSTANCE_NAME} --service-type=${COMPONENTS}
```

Verifying the service instance upgrade

```
oc get db2ucluster <instance_id> -o jsonpath='{.status.state} {"\n"}'
```

Repeat the preceding steps to upgrade each service instance associated with this instance of IBM Software Hub.

- Check the service instances have updated

```
cpd-cli service-instance list \ 
--profile=${CPD_PROFILE_NAME} \
--service-type=db2wh
```

## Part 3: Post-upgrade

### 3.1 Validate the external vault connection setting

1)Validate and ensure the patch for external vault connection applied.

<br>

Found out the following variables set to false

```
oc set env deployment/zen-core-api --list | grep -i vault
```

The values are true like this:

```
VAULT_BRIDGE_TOLERATE_SELF_SIGNED=true
VAULT_BRIDGE_TLS_RENEGOTIATE=true
```

2)Add Verizon logo on CPD homepage

<br>

[Customizing the branding of the web client](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=users-customizing-branding-web-client)

### 3.2 CCS post-upgrade tasks

**1.Add the label back to cronjobs**
<br>
After the reconciliation is completed, add the label back to cronjobs

```
oc project ${PROJECT_CPD_INST_OPERANDS} 
for cj in $(oc get cronjob -l runtimeAssembly --no-headers | awk '{print $1}'); do oc label cronjob $cj created-by=spawner 2>/dev/null; done
```

**2.Check if uploading JDBC drivers enabled**

```
oc get ccs ccs-cr -o yaml | grep -i wdp_connect_connection_jdbc_drivers_repository_mode
```

Make sure the `wdp_connect_connection_jdbc_drivers_repository_mode` parameter set to be enabled.

**3.Check the heap size in asset-files-api deployment**
<br>
Check if the heap size 12288 is set as expected.

```
oc get deployment asset-files-api -o yaml | grep -i -A5 'max-old-space-size=12288'
```

### 3.3 WKC post-upgrade tasks

**1.Validate the 'Allow ' settings for Catalogs and Projects**
<br>
1)Check the Reporting settings in the ccs-features-configmap.

```
oc get configmaps ccs-features-configmap -o yaml -n ${PROJECT_CPD_INST_OPERANDS} | grep -i reporting
```

The following output is expected:

```
defaultAuthorizeReporting: "true"
enforceAuthorizeReporting: "true"
```

2)Verify that the environemnt variable is set for ngp-projects-api.

```
oc set env -n ${PROJECT_CPD_INST_OPERANDS} deployment/ngp-projects-api --list | grep -i reporting
```

The following output is expected:

```
DEFAULT_AUTHORIZE_REPORTING=True
ENFORCE_AUTHORIZE_REPORTING=True
```

3)Verify that the environment variable is set for catalog-api

```
oc exec -it $(oc get pods --no-headers | grep -i catalog-api- | head -n 1 | awk '{print $1}') -- env | grep -i reporting

```

The following output is expected:

```
defaultAuthorizeReporting=true
enforceAuthorizeReporting=true
```

If any of the above output inconsistent with the expected ones, then follow below documentation for applying 'Allow Reporting' settings.

<br>

[Configuring reporting settings for IBM Knowledge Catalog](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=administering-configuring-reporting-settings)

**2.Bulk sync assets for global search**
<br>
As we aim to have bulk re-sync run in the background of the day to day operations, let's tweak concurrency to a level that allows for adequate throughput for the rest of the wkc-search clients.

<br>

1)Switch to the CPD instance project.

```
oc project ${PROJECT_CPD_INST_OPERANDS}
```

2)Create a script named `wkc-search-reindexing-concurrency-tweak.sh` with the following content.

```
# Modify these parameters to tweak the degree of concurrency #
export writer_max_concurrency_threshold=1
export writer_average_concurrency_threshold=0
export elasticsearch_bulk_size=1000
export max_processing_rate=1000
##############################################################

# Extracts CM as json, extracts json config as pure json
oc get cm wkc-search-search-sync-columns-cm -ojson > wkc-search-search-sync-columns-cm.json
cat wkc-search-search-sync-columns-cm.json | jq '.data["config.json"]' > config.json
cat config.json | sed "s/\\\n//g" | sed 's/\\"/TTT/g' | sed 's/"//g' | sed 's/TTT/"/g' | sed 's/\\t//g' | jq > config.json_tmp
mv config.json_tmp config.json

# Modify json config with desired parameters
jq ".asset_flow.processors.asset_processor.configuration.writer_max_concurrency_threshold = \"$writer_max_concurrency_threshold\" | .asset_flow.processors.asset_processor.configuration.writer_average_concurrency_threshold = \"$writer_average_concurrency_threshold\" | .asset_flow.processors.asset_processor.configuration.max_processing_rate = \"$max_processing_rate\" | .asset_flow.processors.asset_processor.writer.configuration.elasticsearch_bulk_size = \"$elasticsearch_bulk_size\"" config.json > config2.json
mv config2.json config.json

# Prepare original CM with updated data
export config_json=$(cat config.json | jq --compact-output | sed 's/"/\\"/g')
jq ".data[\"config.json\"]=\"$config_json\"" wkc-search-search-sync-columns-cm.json > wkc-search-search-sync-columns-cm.json_tmp
mv wkc-search-search-sync-columns-cm.json_tmp wkc-search-search-sync-columns-cm.json

# Update CM
oc apply -f wkc-search-search-sync-columns-cm.json

```

3)Run the `wkc-search-reindexing-concurrency-tweak.sh` script

```
chmod +x wkc-search-reindexing-concurrency-tweak.sh

./wkc-search-reindexing-concurrency-tweak.sh
```

4)Run bulk script cpd_gs_sync.sh following this documentation [Bulk sync assets for global search](https://www.ibm.com/docs/en/SSNFH6_5.1.x/wsj/admin/admin-bulk-sync.html).
<br>

**Note**

<br>
- If there are a large number of assets, the wkc-search-reindexing-job may take a few hours to complete. You can monitor the status with below command.

```
oc logs $(oc get pods --no-headers | grep -i wkc-search-reindexing-job- | head -n 1 | awk '{print $1}') | grep "CAMSStatisticsCollector reports" -A10
```

<br>

- Be aware that the changes will be overwritten after a CCS reconcile cycle, so if you are planning to run bulk with tweaked concurrency parameters - its adviced to always apply the above script beforehand.

<br>

**3.Add potential missing permissions for the pre-defined Data Quality Analyst and Data Steward roles**
<br>

```
oc delete pod $(oc get pod -n ${PROJECT_CPD_INST_OPERANDS} -o custom-columns="Name:metadata.name" -l app.kubernetes.io/component=zen-watcher --no-headers) -n ${PROJECT_CPD_INST_OPERANDS}
```

**4. Migrating profiling results after upgrading**
<br>
In Cloud Pak for Data 5.1.1, profiling results are stored in a PostgreSQL database instead of the asset-files storage. To make existing profiling results available after upgrading from an earlier release, migrate the results following this IBM documentation.
[Migrating profiling results after upgrading](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=administering-migrating-profiling-results)

<br>

Sample override.yaml file:

```
namespace: hptv-prodcloudpak
blockStorageClass: ocs-storagecluster-ceph-rbd
fileStorageClass: ocs-storagecluster-cephfs
docker_registry_prefix: cp.icr.io/cp/cpd
use_dynamic_provisioning: true
ansible_python_interpreter: /usr/bin/python3
allow_reconcile: true
wdp_profiling_postgres_action: MIGRATE
```

**Note**
<br>
1).The nohup command is recommended for the migration of a large number of records.

```
nohup ansible-playbook /opt/ansible/5.1.1/roles/wkc-core/wdp_profiling_postgres_migration.yaml --extra=@/tmp/override.yaml -vvvv &
```

2).Validate the job log for successful migration of profiling data; then run the `CLEAN` option.
<br>
**Important:** The data is permanently deleted and can't be restored. Therefore, use this option only after all results are copied successfully and you do no longer need the results in the asset-files storage. So recommend taking a note of this procedure and run it after the tests passed from the end-users.

<br>

## Part 4: Maintenance

This part is beyond the upgrade scope. And we are not commited to complete them in the two days time window.

### 4.1 Migrating from MANTA Automated Data Lineage to IBM Manta Data Lineage

#### 4.1.1 Uninstall MANTA Automated Data Lineage

- Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster.

```
${CPDM_OC_LOGIN}
```

- Delete the custom resource for MANTA Automated Data Lineage.

```
cpd-cli manage delete-cr \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=mantaflow \
--include_dependency=true
```

Wait for the cpd-cli to return the following message before you proceed to the next step:

```
[SUCCESS]... The delete-cr command ran successfully
```

- Delete the OLM objects for MANTA Automated Data Lineage:

```
cpd-cli manage delete-olm-artifacts \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--components=mantaflow
```

Wait for the cpd-cli to return the following message:

```
[SUCCESS]... The delete-olm-artifacts command ran successfully
```

#### 4.1.2 Remove the fdbcluster and deploy the Neo4jCluster

- Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster.

```
${CPDM_OC_LOGIN}
```

- Make IBM Knowledge Catalog using Neo4j as the knowledge graph database

<br>
Delete the fdbcluster.

```
oc delete fdbcluster wkc-foundationdb-cluster -n ${PROJECT_CPD_INST_OPERANDS}
```

Patch the wkc-cr to deploy the Neo4j cluster.

```
oc patch wkc wkc-cr -n ${PROJECT_CPD_INST_OPERANDS} --type=merge -p '{"spec":{"useFDB":false}}'
```

Monitor the WKC operator reconcilation by checking the WKC operator log.

<br>
If the IBM Knowledge Catalog custom resource (wkc-cr) reconciliation fails, check if the `wdp-kg-ingestion-service` and `wkc-data-lineage-service` pods are stuck in the `ContainerCreating` status. If yes, then delete these two deployments:

```
oc delete deploy wdp-kg-ingestion-service -n ${PROJECT_CPD_INST_OPERANDS}
oc delete deploy wkc-data-lineage-service -n ${PROJECT_CPD_INST_OPERANDS}
```

Wait for the next reconciliation of the wkc-cr or restart the IBM Knowledge Catalog operator.

```
watch -n 60 "cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=wkc,datalineage"
```

Ensure the Neo4jCluster is in 'Completed' status.

```
oc get Neo4jCluster data-lineage-neo4j -n ${PROJECT_CPD_INST_OPERANDS}
```

#### 4.1.3 Install the IBM Manta Data Lineage

- Run the following command to create the required OLM objects for IBM Manta Data Lineage .

```
cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--components=datalineage
```

Wait for the cpd-cli to return the following message before you proceed to the next step:

```
[SUCCESS]... The apply-olm command ran successfully
```

- Create the custom resource for IBM Manta Data Lineage.

```
cpd-cli manage apply-cr \
--components=datalineage \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true
```

Validating the upgrade.

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=datalineage
```

Set the scale to be `Large`.

```
export SCALE=level_4
```

Run the following command to scale the component by updating the custom resource.

```
cpd-cli manage apply-scale-config \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=datalineage \
--scale=${SCALE}
```

- Apply Lineage Hotfix by using the new images

```
oc patch datalineage datalineage-cr -n ${PROJECT_CPD_INST_OPERANDS} --type=merge -p '{"spec":{"datalineage_scanner_service_image_tag":"3a851c76cde46def29f2e489338a040d1e430034982fa6d6c87f5b95ae99b4e8","datalineage_scanner_service_image_tag_metadata":"2.3.1","datalineage_scanner_worker_image_tag":"1c731288ca446c22df24d4062a1ed15ac6a69305af0ecc5288d3d44fba92d2b1","datalineage_scanner_worker_image_tag_metadata":"2.3.4"}}'
```

Confirm Datalineage reconcile successfully.

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=datalineage
```

- Apply the Lineage Performance fix by creating index and constrains for the Neo4j database

```
#1) Exec into datalineage pod -
oc -n ${PROJECT_CPD_INST_OPERANDS} exec -it data-lineage-neo4j-server1-0 -- bash

#2) Execute the cyper shell =
cypher-shell -a "neo4j+ssc://localhost:7687" -u neo4j -p "$(cat /config/neo4j-auth/NEO4J_AUTH | cut -d/ -f2)"

#3) Initialize a new databse index- 
CREATE INDEX lineage_graph_anchor_property_deleted_index IF NOT EXISTS FOR(n:LineageGraphAnchor) ON (n.deleted);

#4) Verify if new index is present -
SHOW INDEX YIELD name, state, labelsOrTypes, properties WHERE"lineage_graph_anchor_property_deleted_index" = name;

#Output should be similar: "lineage_graph_anchor_property_deleted_index" "ONLINE" ["LineageGraphAnchor"]["deleted"]

#5) Initialize new constraints
CREATE CONSTRAINT lineage_graph_anchor_deleted_is_boolean_constraint IF NOT EXISTS FOR (n:LineageGraphAnchor) REQUIRE n.deleted IS :: BOOLEAN;

#6) Verify constraint has been initialized -
SHOW CONSTRAINT YIELD name, type, entityType, labelsOrTypes, properties WHERE"lineage_graph_anchor_deleted_is_boolean_constraint" = name;

#Output should be similar: "lineage_graph_anchor_deleted_is_boolean_constraint" "NODE_PROPERTY_TYPE""lineage_graph_anchor_deleted_is_boolean_constraint" "NODE_PROPERTY_TYPE""NODE" ["LineageGraphAnchor"] ["deleted"]

#7) Exit out of lineage pod -
:exit

```

#### 4.1.4 Migrating from MANTA Automated Data Lineage to IBM Manta Data Lineage

**Note:**
<br>

- Migration needs to be run as root or by a user with sudo access.

[Migrating from MANTA Automated Data Lineage to IBM Manta Data Lineage](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=lineage-migrating)

#### 4.1.5 Post-migration tasks

**1. Resync glossary assets**
<br>
1)Get the Bearer token for calling CPD REST API

```
curl -X POST \
  'https://$CPD_URL/icp4d-api/v1/authorize'\
  -H 'Content-Type: application/json' \
  -d' { \
    "username":"<change it to your CPD user name>", \
    "password":"<change it the corresponding CPD password>" \
  }'
```

2).Run the resync command for glossary assets

```
curl -k -X POST  -H "Content-Type: application/json" -H "Accept: application/json" -H "Authorization: Bearer $TOKEN" "https://$CPD_URL/v3/glossary_terms/admin/resync?artifact_type=all&sync_destinations=KNOWLEDGE_GRAPH" -d '{}'
```

**2. Resync of lineage metadata**

<br>

Resynchronize your catalog metadata to start seeing the Knowledge Graph. Follow the steps in below IBM Documentation.

<br>

**Note:** Resync all catalogs when specifying what catalogs to resyc.

<br>

[Resync of lineage metadata](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=administering-resync-lineage-metadata)

**3.Apply the workaround for addressing the issue: Lineage Tab page is keep on spinning**

Refer to the detailed steps updated by Sanjit 2/17/2025 in the ticket TS018466973.

**4.Apply the wxO hotfix**
https://www.ibm.com/support/pages/node/7249508
https://www.ibm.com/support/pages/node/7247038

## Summarize and close out the upgrade

1)Schedule a wrap-up meeting and review the upgrade procedure and lessons learned from it.

2)Evaluate the outcome of upgrade with pre-defined goals.

---

End of document
