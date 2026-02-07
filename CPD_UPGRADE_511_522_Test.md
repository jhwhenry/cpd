# CPD Upgrade Runbook - v.5.1.1 to 5.2.2

---

## Upgrade documentation

[Upgrading from IBM Cloud Pak for Data Version 5.1 to Version 5.2](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=u-upgrading-from-version-51-29)

## Upgrade context

From

```
OCP: 4.16
CPD: 5.1.1
Storage: Storage Fusion 2.7.2 
Componenets: cpd_platform,wkc,analyticsengine,datalineage,ws,ws_runtimes,wml,openscale,db2wh,match360
```

To

```
OCP: 4.16
CPD: 5.2.2
Storage: Storage Fusion 2.7.2 
Componenets: cpd_platform,wkc,analyticsengine,datalineage,ws,ws_runtimes,wml,openscale,db2wh,match360
```

## Pre-requisites

#### 1. Backup of the cluster is done.

Backup your Cloud Pak for Data cluster before the upgrade.
For details, see [Backing up and restoring Cloud Pak for Data](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=administering-backing-up-restoring-software-hub).

**Note:**
Make sure there are no scheduled backups conflicting with the scheduled upgrade.

#### 2. The image mirroring completed successfully

If a private container registry is in-use to host the IBM Cloud Pak for Data software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry. 

<br>
Reference: 
<br>

[Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=prufpcr-mirroring-images-private-container-registry)


#### 3. Ensure the ImageDigestMirrorSet configured properly

With the proper configuration of ImageDigestMirrorSet, it can help avoid the issue - `ccs-cams-postgres` pod failing in pulling images.

#### 4. The permissions required for the upgrade is ready

- Openshift cluster permissions
<br>
  An Openshift cluster administrator can complete all of the installation tasks.
<br>

However, if you want to enable users with fewer permissions to complete some of the installation tasks, follow the steps in this documentation and get the roles with required permission prepared.

[Reauthorizing the instance administrator](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=hub-reauthorizing-instance-administrator)

<br>

- Cloud Pak for Data permissions

<br>
The Cloud Pak for Data administrator role or permissions is required for upgrading the service instances.

- Permission to access the private image registry for pushing or pull images
- Access to the Bastion node for executing the upgrade commands

#### 5. Migrate environments based on Watson Studio Runtime 22.2 and Runtime 23.1 from IBM Cloud Pak® for Data 5.1 (optional)

Runtime 22.2 and Runtime 23.1 are not included in IBM® Software Hub. To continue using notebooks, scripts, and jobs that use environments based on Runtime 22.2 or Runtime 23.1, you must review them and manually migrate them, if necessary.

<br>

[Migrating notebooks, scripts, and jobs that use outdated environments](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=upgrading-migrating-notebooks-scripts-jobs-that-use-outdated-environments)

#### 6. A pre-upgrade health check is made to ensure the cluster's readiness for upgrade.

- The OpenShift cluster, persistent storage and Cloud Pak for Data platform and services are in healthy status.

## Table of Content

```
Part 1: Pre-upgrade
1.1 Collect information and review upgrade runbook
1.1.1 Review the upgrade runbook
1.1.2 Backup before upgrade
1.1.3 Pre-check before upgrade
1.1.4 Uninstall all hotfixes and apply preventative measures
1.2 Set up client workstation 
1.2.1 Prepare a client workstation
1.2.2 Update cpd_vars.sh for the upgrade to Version 5.2.2
1.2.3 Obtain the olm-utils-v3 available
1.2.4 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image
1.2.5 Ensure the images were mirrored to the private container registry
1.2.6 Creating a profile for upgrading the service instances
1.3 Health check OCP & CPD

Part 2: Upgrade
2.1 Upgrade CPD to 5.2.2
2.1.1 Upgrading shared cluster components
2.1.2 Preparing to upgrade an instance of IBM Software Hub
2.1.3 Upgrading to IBM Software Hub
2.1.4 Upgrading the operators for the services
2.1.5 Applying the RSI patches
2.2 Upgrade CPD services
2.2.1 Upgrading IBM Knowledge Catalog service and apply hot fixes
2.2.2 Upgrading Datalineage service
2.2.3 Upgrading Analytics Engine service instance
2.2.4 Upgrading Watson Studio, Watson Studio Runtimes, Watson Machine Learning and OpenScale
2.2.5 Upgrading Db2 Warehouse
2.2.6 Upgrading Match360

Part 3: Post-upgrade
3.1 Validate platform customization settings
3.2 WKC post-upgrade tasks

Part 4: Maintenance

Summarize and close out the upgrade

```

## Part 1: Pre-upgrade

### 1.1 Collect information and review upgrade runbook

#### 1.1.1 Review the upgrade runbook

Review upgrade runbook

#### 1.1.2 Backup before upgrade

<br>
Login to the OCP cluster for cpd-cli utility.

```
${CPDM_OC_LOGIN}
```

Backup the RSI patches.

```
cpd-cli manage get-rsi-patch-info \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--all
```

#### 1.1.3 Pre-upgrade check

[Pre-upgrade check steps](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=uish-upgrading-software-hub-1#taskupgrade-instance__prereq__1)

<br>

**Note:**
<br>
No need to patch the CCS cr. Just complete the steps before Step 3 for determining approximately how long the service will be offline during the migration.

#### 1.1.4 Uninstall all hotfixes and apply preventative measures

- 1.Apply preventive measure for the ZenService.

<br>

Add below information under the ZenMinio section.
```
livenessProbe:
  initialDelaySeconds: 30
  periodSeconds: 30
  failureThreshold: 6
  timeoutSeconds: 10
readinessProbe:
  initialDelaySeconds: 60
  periodSeconds: 30
  failureThreshold: 6
  timeoutSeconds: 10
```

- 2.Uninstall the CCS hot fixes and apply the patch for migration preparation.
<br>

1) Remove the hot fix from the CCS custom resource

<br>

Edit the CCS custom resource.

```
oc edit ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

Remove the hot fix.

```
image_digests:
  catalog_api_image: sha256:6ac5cb00390d96a66540029b08e5abb47cf52e6142d7613757b1c252b6a6ecb0
  catalog_api_jobs_image: sha256:29dbaa7d9b4e6c19424b05e35e31db7cee1d66c06a0d633e56f5ad96f5786dab
  portal_catalog_image: sha256:1259d1d359bf04008ca2c6de56d5d0cdada36bcb4fe711170de29924e974c3ae
  portal_projects_image: sha256:69b6555dc22d346fe7c6e0235633a77b920a8f507dbf7b2c1c96c0383a20b7de
  wdp_connect_connection_image: sha256:02826fa27eed4813f62bce2eccd66ee8ab17c2ee56df811551746d683aa7ae0f
  wdp_connect_connector_image: sha256:c85fcfadda98e2f7d193b12234dbec013105e50b9f59f157505c28f5e572edcc
  wdp_connect_flight_image: sha256:cda30760185008c723a87bd251f60cb6402f4814ee1523c99a167ad979c5919b
```

2)Patch for the catalog-api migration
```
use_semi_auto_catalog_api_migration: true
catalog_api_postgres_migration_threads: 8
catalog_api_migration_job_resources:
  requests:
    cpu: 6
    ephemeral-storage: 10Mi
    memory: 6Gi
  limits:
    cpu: 10
    ephemeral-storage: 6Gi
    memory: 10Gi
```

3)Remove the maintenance mode

```
ignoreForMaintenance: true
```

Save and Exit. Wait until the CCS Operator reconcilation completed and also the ccs-cr in 'Completed' status.

```
oc get CCS ccs-cr -o yaml -n ${PROJECT_CPD_INST_OPERANDS}
```

- 3.Remove DataRefinery from the maintenance mode

<br>

Edit the DataRefinery custom resource.

```
oc edit DataRefinery datarefinery-cr -n ${PROJECT_CPD_INST_OPERANDS} 
```

Remove the maintenance mode.

```
ignoreForMaintenance: true
```

- 4.Remove AnalyticsEngine from the maintenance mode

<br>

Edit the AnalyticsEngine analyticsengine-sample.

```
oc edit AnalyticsEngine analyticsengine-sample -n ${PROJECT_CPD_INST_OPERANDS}
```

Remove below section.

```
ignoreForMaintenance: true
```

Save and Exit. Wait until the AnalyticsEngine Operator reconcilation completed and also the analyticsengine-sample in 'Completed' status.

```
oc get AnalyticsEngine analyticsengine-sample -o yaml -n ${PROJECT_CPD_INST_OPERANDS}
```

- 5.Remove WKC maintenance mode and the hot fixes

<br>
Edit the WKC CR.

```
oc edit wkc wkc-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

Remove maintenance mode and the hot fixes

```
ignoreForMaintenance: true
image_digests:
  metadata_discovery_image: sha256:b89559cea54616a530557956dd806895b454bd5180cb7ce3656c440325f92591
  wdp_kg_ingestion_service_image: sha256:0b77632e2406dff9b2bb6bbcdbf6a06f7748f5aa702a021fde1eeeebf44bda9b
  wkc_bi_data_service_image: sha256:df96efc9d94cb6e335ce6ea1815b4c29867eee6fbd91f7ba78b11561dbfcb2ad
  wkc_data_lineage_service_image: sha256:bc0a37a460f383f9a5fce0f7decd0a074db83b9df56d541f61835ea32a486c88
  wkc_mde_service_manager_image: sha256:a7a3ea48d72baaae484c6dde0ff910a89164993795cf530054e2a39ee9bf90ce
  wkc_metadata_imports_ui_image: sha256:1487c666890f13494a9d2fe14453cd0c46234bc0b799b354ca9526f090404506
```

Save and Exit. 

<br>

**Note:** 
<br>

Check whether the `image_digests` property exists in the custom resources of WKC components. 

```
for crd in $(oc get crd | grep volumereplication | awk '{print $1}'); do
  oc get crd "$crd" -o jsonpath='{.spec.names.plural}' -A -o yaml
done
```

Review the custom resource content one by one. If it contains the `image_digests` property, then remove it from the custom resource. 


Wait until the WKC Operator reconcilation completed and also the wkc-cr in 'Completed' status.

```
oc get WKC wkc-cr -o yaml -n ${PROJECT_CPD_INST_OPERANDS}
```

- 6.Remove DataLineage from the maintenance mode and uninstall the hot fixes

<br>

Edit the DataLineage CR.

```
oc edit datalineage datalineage-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

Remove the maintenance mode and uninstall the hot fixes.

```
ignoreForMaintenance: true
datalineage_scanner_service_image_tag: 3a851c76cde46def29f2e489338a040d1e430034982fa6d6c87f5b95ae99b4e8
datalineage_scanner_service_image_tag_metadata: 2.3.1
datalineage_scanner_worker_image_tag: 1c731288ca446c22df24d4062a1ed15ac6a69305af0ecc5288d3d44fba92d2b1
datalineage_scanner_worker_image_tag_metadata: 2.3.4
```

Save and Exit. Wait until the DataLineage Operator reconcilation completed and also the datalineage-cr in 'Completed' status.

- 7.Remove IBM Match360 hot fix
Run the following commands to remove the image digest update from the IBM Match 360 custom resource.

```
oc patch mdm mdm-cr --type=json --patch '[{ "op": "remove", "path": "/spec/image_digests"}]' -n ${PROJECT_CPD_INST_OPERANDS}

oc patch mdm mdm-cr --type=json --patch '[{ "op": "remove", "path": "/spec/mdm_data/spark/image"}]' -n ${PROJECT_CPD_INST_OPERANDS}
```

Wait for the IBM Match 360 operator reconciliation to complete. 
```
oc get mdm mdm-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

- 8.Remove Db2aaserviceService from the maintenance mode

<br>

Edit the Db2aaserviceService db2aaservice-cr.

```
oc edit Db2aaserviceService db2aaservice-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

Remove below section.

```
ignoreForMaintenance: true
```

Save and Exit. Wait until the Db2aaserviceService Operator reconcilation completed and also the db2aaservice-cr in 'Completed' status.

```
oc get Db2aaserviceService db2aaservice-cr -o yaml -n ${PROJECT_CPD_INST_OPERANDS}
```

- 9.Remove Db2whService from the maintenance mode

<br>

Edit the Db2whService db2wh-cr.

```
oc edit Db2whService db2wh-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

Remove below section.

```
ignoreForMaintenance: true
```

Save and Exit. Wait until the Db2whService Operator reconcilation completed and also the db2wh-cr in 'Completed' status.

```
oc get Db2whService db2wh-cr -o yaml -n ${PROJECT_CPD_INST_OPERANDS}
```

### 1.2 Set up client workstation

#### 1.2.1 Prepare a client workstation

1. Prepare a RHEL 9 machine with internet

Create a directory for the cpd-cli utility.

```
export CPD522_WORKSPACE=/ibm/cpd/522
mkdir -p ${CPD522_WORKSPACE}
cd ${CPD522_WORKSPACE}

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

3. Copy the cpd_vars.sh file used by the CPD 5.1.1 to the folder ${CPD522_WORKSPACE}.

```
cd ${CPD522_WORKSPACE}
cp <the file path of the cpd_vars.sh file used by the CPD 5.1.1 > cpd_vars_522.sh
```

4. Make cpd-cli executable anywhere

```
vi cpd_vars_522.sh
```

Add below two lines into the head of cpd_vars_522.sh

```
export CPD522_WORKSPACE=/ibm/cpd/522
export PATH=${CPD522_WORKSPACE}:$PATH
```

Update the CPD_CLI_MANAGE_WORKSPACE variable

```
export CPD_CLI_MANAGE_WORKSPACE=${CPD522_WORKSPACE}
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
        Build Date: 2024-12-05T14:18:50
        Build Number: 1189
        CPD Release Version: 5.2.2
```

5.Update the OpenShift CLI

<br>

Check the OpenShift CLI version.

```
oc version
```

If the version doesn't match the OpenShift cluster version, update it accordingly.

#### 1.2.2 Update environment variables for the upgrade to Version 5.2.2

```
vi cpd_vars_522.sh
```

1.Locate the VERSION entry and update the environment variable for VERSION.

```
export VERSION=5.2.2
```

2.Locate the COMPONENTS entry and confirm the COMPONENTS entry is accurate.

```
export COMPONENTS=ibm-licensing,cpfs,cpd_platform,ws,ws_runtimes,wml,wkc,datastage_ent,datastage_ent_plus,analyticsengine,datalineage,openscale,db2wh,match360
```

Save the changes. `<br>`

Confirm that the script does not contain any errors.

```
bash ./cpd_vars_522.sh
```

Run this command to apply cpd_vars_522.sh

```
source cpd_vars_522.sh
```

3.Locate the Cluster section of the script and add the following environment variables.

```
export SERVER_ARGUMENTS="--server=${OCP_URL}"
export LOGIN_ARGUMENTS="--username=${OCP_USERNAME} --password=${OCP_PASSWORD}"
export CPDM_OC_LOGIN="cpd-cli manage login-to-ocp ${SERVER_ARGUMENTS} ${LOGIN_ARGUMENTS}"
export OC_LOGIN="oc login ${OCP_URL} ${LOGIN_ARGUMENTS}"
```

#### 1.2.3 Obtaining the olm-utils-v3 image

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

#### 1.2.4 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image

```
cpd-cli manage restart-container
```

**Note:**
`<br>`Check and confirm the olm-utils-v3 container is up and running.

```
podman ps | grep olm-utils-v3
```

#### 1.2.5 Ensure the images were mirrored to the private container registry

- Check the log files in the work directory generated during the image mirroring

```
grep "error" ${CPD_CLI_MANAGE_WORKSPACE}/work/mirror_*.log
```

- Log in to the private container registry.

```
cpd-cli manage login-private-registry \
${PRIVATE_REGISTRY_LOCATION} \
${PRIVATE_REGISTRY_PULL_USER} \
${PRIVATE_REGISTRY_PULL_PASSWORD}
```

- Confirm that the images were mirrored to the private container registry:
  Inspect the contents of the private container registry:

```
cpd-cli manage list-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--case_download=false
```

The output is saved to the list_images.csv file in the `work/offline/${VERSION}` directory.

<br>

Check the output for errors:

```
grep "level=fatal" ${CPD_CLI_MANAGE_WORKSPACE}/work/offline/${VERSION}/list_images.csv
```

The command returns images that are missing or that cannot be inspected which needs to be addressed.

#### 1.2.6 Creating a profile for upgrading the service instances

Create a profile on the workstation from which you will upgrade the service instances. `<br>`

The profile must be associated with a Cloud Pak for Data user who has either the following permissions:

- Create service instances (can_provision)
- Manage service instances (manage_service_instances)

Click this link and follow these steps for getting it done.

[Creating a profile to use the cpd-cli management commands](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=cli-creating-cpd-profile)

### 1.3 Health check OCP & CPD

1. Check OCP status

Log onto bastion node, in the termial log into OCP and run this command.

```
oc get co
```

Make sure All the cluster operators should be in AVAILABLE status. And not in PROGRESSING or DEGRADED status.

Run this command and make sure all nodes in Ready status.

```
oc get nodes
```

Run this command and make sure all the machine configure pool are in healthy status.

```
oc get mcp
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

3. Check private container registry status if installed

Log into bastion node, where the private container registry is usually installed, as root.
Run this command in terminal and make sure it can succeed.

```
podman login --username $PRIVATE_REGISTRY_PULL_USER --password $PRIVATE_REGISTRY_PULL_PASSWORD $PRIVATE_REGISTRY_LOCATION --tls-verify=false
```

You can run this command to verify the images in private container registry.

```
curl -k -u ${PRIVATE_REGISTRY_PULL_USER}:${PRIVATE_REGISTRY_PULL_PASSWORD} https://${PRIVATE_REGISTRY_LOCATION}/v2/_catalog?n=6000 | jq .
```

## Part 2: Upgrade

### 2.1 Upgrade CPD to 5.2.2

#### 2.1.1 Upgrading shared cluster components

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster

```
${CPDM_OC_LOGIN}
```

2.Confirm the project into which the License Service is installed.
Run the following command:

```
oc get deployment -A |  grep ibm-licensing-operator
```

Make sure the project returned by the command matches the environment variable PROJECT_LICENSE_SERVICE in your environment variables script `cpd_vars_522.sh`.

<br>

3.Upgrade the Certificate manager and License Service.

<br>

Check whether the Certificate manager is IBM certificate manager. If yes, then run below command.

```
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--cert_manager_ns=${PROJECT_CERT_MANAGER} \
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

Confirm that the Certificate manager pods in the ${PROJECT_CERT_MANAGER} project are Running:

```
oc get pod -n ${PROJECT_CERT_MANAGER}
```

Confirm that the License Service pods are Running or Completed::

```
oc get pods --namespace=${PROJECT_LICENSE_SERVICE}
```

#### 2.1.2 Preparing to upgrade an instance of IBM Software Hub

1.Log into the cluster

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

Reference: 
<br>

[Applying your entitlements](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=puish-applying-your-entitlements)

#### 2.1.3 Upgrading to IBM Software Hub

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
${CPDM_OC_LOGIN}
```

2.Review the license terms for the software that is installed on this instance of IBM Software Hub.

<br>

The licenses are available online. Run the appropriate commands based on the license that you purchased:

```
cpd-cli manage get-license \
--release=${VERSION} \
--license-type=EE
```

3.Check to see if Elastic Search and catalog-api is running before upgrading.

```
oc get pods --namespace=${PROJECT_CPD_INST_OPERANDS} | grep elasticsea-0ac3

oc get pods --namespace=${PROJECT_CPD_INST_OPERANDS} | grep catalog-api
```

4.Upgrade the required operators and custom resources for the instance. 

```
cpd-cli manage setup-instance \
--release=${VERSION} \
--license_acceptance=true \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--run_storage_tests=flase
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

<br>

Increase the resource limits of the CCS operator for avoiding potention problems when dealing with large data volume.

<br>

Have a backup of the CCS CSV yaml file.

```
oc get csv ibm-cpd-ccs.v11.2.0 -n ${PROJECT_CPD_INST_OPERATORS} -o yaml > ibm-cpd-ccs-csv-522.yaml
```

Edit the CCS CSV:

```
oc edit csv ibm-cpd-ccs.v11.2.0 -n ${PROJECT_CPD_INST_OPERATORS} 
```

Make changes to the limits like below.

```
    resources:
      limits:
        cpu: 4
        ephemeral-storage: 5Gi
        memory: 8Gi
```

This change can be reverted after the upgrade completed successfully.

#### 2.1.5 Apply the RSI patches

1).Log the cpd-cli in to the Red Hat OpenShift Container Platform cluster.

```
${CPDM_OC_LOGIN}
```

2).Enable the zen-rsi-evictor-cron-job CronJob
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

4).Check the RSI patches status again: (Note: Profiling RSI path from 5.1.1 to 5.2.2 should still be configured)

```
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --all

cat $CPD_CLI_WORK_DIR/get_rsi_patch_info.log
```

### 2.2 Upgrade CPD services to 5.2.2

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
    enableDataQuality: true
    enableFactSheet: false
    enableKnowledgeGraph: true
    enableMANTA: true
    useFDB: false
```

**Note:**
<br>
1)Make sure you edit or create the `install-options.yml` file in the right `work` folder.

<br>

Identify the location of the `work` folder using below command.

```
podman inspect olm-utils-play-v4 | jq -r '.[0].Mounts' |jq -r '.[] | select(.Destination == "/tmp/work") | .Source'
```

<br>

2)Make sure the `useFDB` is set to be `False` in the install-options.yml file. And remove FDBcluster for WKC if present.

```bash
oc get fdbcluster -n ${PROJECT_CPD_INST_OPERANDS}

(If present)
oc delete fdbcluster wkc-foundationdb-cluster -n ${PROJECT_CPD_INST_OPERANDS}
```

<br>

##### 2.Upgrade WKC with custom installation

Log into the OCP cluster.

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

**Note:**

<br>

Right after the above command triggered, make change to the `asset_files_api_args` property of the CCS custom resource as below. 

```
oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type=merge -p '{
   "spec":{
      "asset_files_api_args":["-c","cd /home/node/${MICROSERVICENAME};source /scripts/exportSecrets.sh;export npm_config_cache=~node;node --max-old-space-size=12288 --max-http-header-size=32768 index.mjs"]
   }
}'
```

##### 3.Validate the upgrade

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

##### 4.Apply the customizations (Should not need any hotfixes)

#### 2.2.2 Upgrading IBM MANTA Lineage service

```
export COMPONENTS=datalineage

```

- Log in to the OpenShift cluster

```
${CPDM_OC_LOGIN}
```

- Run the command for upgrading DataLineage service.

```
cpd-cli manage apply-cr \
--components=datalineage \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--upgrade=true
```

Validating the upgrade.

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=datalineage
```


#### 2.2.3 Upgrading Analytics Engine service instance

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

Log in to the OpenShift cluster.

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

Log in to the OpenShift cluster

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
`<br>`

- Get a list of your Db2 Warehouse service instances

```
cpd-cli service-instance list \
--service-type=db2wh \
--profile=${CPD_PROFILE_NAME}
```

- Upgrade Db2 Warehouse service instances
  `<br>`
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

#### 2.2.6 Upgrading Match360

```
export COMPONENTS=match360
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

## Part 3: Post-upgrade

### 3.1 Validate platform customization settings

1)Add Verizon logo on CPD homepage

<br>

[Customizing the branding of the web client](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=users-customizing-branding-web-client)

<br>

2)Create a custom route

<br>

[Create a custom route using cpd-cli](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=platform-modifying-route)


### 3.2 WKC post-upgrade tasks

[1.Complete the catalog-api service migration to PostgreSQL](https://www.ibm.com/docs/en/SSNFH6_5.2.x/hub/admin/post-install-services-catalog-api-migration.html)

[2.Post-upgrade setup for IBM Knowledge Catalog](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=upgrading-post-upgrade-setup-knowledge-catalog)

## Part 4: Maintenance 

This part is beyond the upgrade scope. 

## Summarize and close out the upgrade

1)Schedule a wrap-up meeting and review the upgrade procedure and lessons learned from it.

2)Evaluate the outcome of upgrade with pre-defined goals.

---

End of document
