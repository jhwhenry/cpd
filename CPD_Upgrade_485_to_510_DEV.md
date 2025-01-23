# CPD Upgrade Runbook - v.4.8.5 to 5.1.0

---
## Upgrade documentation
[Upgrading from IBM Cloud Pak for Data Version 4.8 to Version 5.1](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=upgrading-from-cloud-pak-data-version-48)

## Upgrade context
From

```
OCP: 4.14
CPD: 4.8.5
Storage: Storage Fusion 2.7.2
Componenets: cpd_platform,wkc,analyticsengine,mantaflow,datalineage,ws,ws_runtimes,wml,openscale,db2wh,match360
```

To

```
OCP: 4.14
CPD: 5.1.0
Storage: Storage Fusion 2.7.2
Componenets: cpd_platform,wkc,analyticsengine,mantaflow,datalineage,ws,ws_runtimes,wml,openscale,db2wh,match360
```

## Pre-requisites
#### 1. Backup of the cluster is done.
Backup your Cloud Pak for Data cluster before the upgrade.
For details, see [Backing up and restoring Cloud Pak for Data](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=administering-backing-up-restoring-software-hub).

**Note:**
Make sure there are no scheduled backups conflicting with the scheduled upgrade.
  
#### 2. The image mirroring completed successfully
If a private container registry is in-use to host the IBM Cloud Pak for Data software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry. <br>
Reference: <br>
[Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=prufpcr-mirroring-images-private-container-registry) 
<br>
#### 3. The permissions required for the upgrade is ready

- Openshift cluster permissions
<br>
An Openshift cluster administrator can complete all of the installation tasks.

<br>

However, if you want to enable users with fewer permissions to complete some of the installation tasks, follow the steps in this documentation and get the roles with required permission prepared.

[Reauthorizing the instance administrator](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=hub-reauthorizing-instance-administrator)

<br>

- Cloud Pak for Data permissions

<br>
The Cloud Pak for Data administrator role or permissions is required for upgrading the service instances.

#### 4. Migrate the Cloud Pak for Data image content source policy to an image digest mirror set.
Starting in Red Hat OpenShift Container Platform Version 4.14, the image content source policy is replaced by the image digest mirror set. If you upgrade to Red Hat OpenShift Container Platform Version 4.14 or later, you must migrate the Cloud Pak for Data image content source policy to an image digest mirror set.
<br>

[Migrating your image content source policy to an image digest mirror set](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=cluster-migrating-image-digest-mirror-set)

#### 5. Migrate environments based on Watson Studio Runtime 22.2 and Runtime 23.1 from IBM Cloud Pak® for Data 4.8 (optional)
The Watson Studio Runtime 22.2 and Runtime 23.1 are not included in IBM® Software Hub. If you want to continue using environments that are based on Runtime 22.2 or Runtime 23.1, you must migrate them.
<br>
[Migrating environments based on Runtime 22.2 and Runtime 23.1 from IBM Cloud Pak® for Data 4.8](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=u-migrating-environments-based-runtime-222-runtime-231-from-cloud-pak-data-48-50)

#### 6. Collect the number of profiling records to be migrated
Collect profiling records information
```
oc project ${PROJECT_CPD_INST_OPERANDS}

oc exec -it $(oc get pods --no-headers | grep -i asset-files | head -n 1 | awk '{print $1}') -- ls -alRt /mnt/asset_file_api | grep -i profiling > /tmp/profiling_records.txt
oc exec -it $(oc get pods --no-headers | grep -i asset-files | head -n 1 | awk '{print $1}') -- ls -alRt /mnt/asset_file_api | grep -i profiling | wc -l > /tmp/profiling_records_number.txt

oc exec -it $(oc get pods --no-headers | grep -i asset-files | head -n 1 | awk '{print $1}') -- ls -alRt /mnt/asset_file_api | egrep -i 'REF_|ARES_' > /tmp/profiling_records_ref.txt
oc exec -it $(oc get pods --no-headers | grep -i asset-files | head -n 1 | awk '{print $1}') -- ls -alRt /mnt/asset_file_api | egrep -i 'REF_|ARES_' | wc -l > /tmp/profiling_records_ref_number.txt

```
#### 7. A pre-upgrade health check is made to ensure the cluster's readiness for upgrade.
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
1.2.2 Update cpd_vars.sh for the upgrade to Version 5.1.0
1.2.3 Obtain the olm-utils-v3 available
1.2.4 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image
1.2.5 Ensure the images were mirrored to the private container registry
1.2.6 Creating a profile for upgrading the service instances
1.3 Health check OCP & CPD

Part 2: Upgrade
2.1 Upgrade CPD to 5.1.0
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
2.2.6 Upgrading Match360

Part 3: Post-upgrade
3.1 Validate the external vault connection setting 
3.2 CCS post-upgrade tasks
3.3 WKC post-upgrade tasks

Part 4: Maintenance
4.1 Migrating from MANTA Automated Data Lineage to IBM Manta Data Lineage
4.2 Changing Db2 configuration settings
4.3 Configure the idle session timeout
4.4 Increasing the number of nginx worker connections
4.5 Increase ephemeral storage for zen-watchdog-serviceability-job
4.6 Upgrade the Backup & Restore service and application

Summarize and close out the upgrade

```

## Part 1: Pre-upgrade
### 1.1 Collect information and review upgrade runbook

#### 1.1.1 Review the upgrade runbook

Review upgrade runbook

#### 1.1.2 Backup before upgrade
Note: Create a folder for 4.8.5 and maintain below created copies in that folder. <br>
Login to the OCP cluster for cpd-cli utility.

```
cpd-cli manage login-to-ocp --username=${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL}
```

Capture data for the CPD 4.8.5 instance. No sensitive information is collected. Only the operational state of the Kubernetes artifacts is collected.The output of the command is stored in a file named collect-state.tar.gz in the cpd-cli-workspace/olm-utils-workspace/work directory.

```
cpd-cli manage collect-state \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Make a copy of existing custom resources (Recommended)

```
oc project ${PROJECT_CPD_INST_OPERANDS}

oc get ibmcpd ibmcpd-cr -o yaml > ibmcpd-cr.yaml

oc get zenservice lite-cr -o yaml > lite-cr.yaml

oc get CCS ccs-cr -o yaml > ccs-cr.yaml

oc get wkc wkc-cr -o yaml > wkc-cr.yaml

oc get analyticsengine analyticsengine-sample -o yaml > analyticsengine-cr.yaml

oc get DataStage datastage -o yaml > datastage-cr.yaml

oc get mantaflow -o yaml > mantaflow-cr.yaml

oc get db2ucluster db2oltp-wkc -o yaml > db2ucluster-db2oltp-wkc.yaml
```

Backup the routes.

```
oc get routes -o yaml > routes.yaml
```

Backup the RSI patches.
```
cpd-cli manage get-rsi-patch-info \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--all
```

Backup the SSO configuration:
```
oc get configmap saml-configmap -o yaml > saml-configmap-cm.yaml
```

Backup deployments and svc that need to be modified after upgrade.
```
oc get deploy asset-files-api -o yaml -n ${PROJECT_CPD_INST_OPERANDS} > asset-files-api-deploy-485.yaml
oc get deploy catalog-api -o yaml -n ${PROJECT_CPD_INST_OPERANDS} > catalog-api-deploy-485.yaml
oc get svc finley-public -o yaml -n ${PROJECT_CPD_INST_OPERANDS} > finley-public-svc-485.yaml
```

#### 1.1.3 Uninstall all hotfixes and apply preventative measures 
Remove the hotfixes by removing the images or configurations from the CRs.
<br>
- 1.Remove useless configuration from the Mantaflow CR.

<br>

1)Edit the Mantaflow CR with below command.
```
oc patch mantaflow mantaflow-wkc --type='json' -p='[{ "op": "remove", "path": "/spec/migrations"}]'
```
2)Remove the Mantaflow deployments 
```
 oc delete deploy manta-admin-gui manta-configuration-service manta-dataflow
```

3)Wait untile the Mantaflow Operator reconcilation completed. 

```
oc get mantaflow mantaflow-wkc -o yaml
```

- 2.Uninstall WKC hot fixes.

<br>

1)Edit the wkc-cr with below command.
```
oc edit WKC wkc-cr
```
2)Remove the hot fix images from the WKC custom resource

```
 wkc_metadata_imports_ui_image:
    name: xxxxxx
    tag: a1997d9a9cde9ecc9f16eb02099a272d7ba2e8d88cb05a9f52f32533e4d633ef
    tag_metadata: xxxxxx

  wdp_profiling_image:
    name: xxxxxx
    tag: d5491cc8c8c8bd45f2605f299ecc96b9461cd5017c7861f22735e3a4d0073abd
    tag_metadata: xxxxxx

  wkc_mde_service_manager_image:
    name: xxxxxx
    tag: 35e6f6ede3383df5c0b2a3d27c146cc121bed32d26ab7fa8870c4aa4fbc6e993
    tag_metadata: xxxxxx

  finley_public_image:
    name: xxxxxx
    tag: e89b59e16c4c10fce5ae07774686d349d17b2d21bf9263c50b49a7a290499c6d
    tag_metadata: xxxxxx
```

3)Remove the `ignoreForMaintenance: true` from the WKC custom resource

```
ignoreForMaintenance: true
```

4)Save and Exit. Wait untile the WKC Operator reconcilation completed and also the wkc-cr in 'Completed' status. 

```
oc get WKC wkc-cr -o yaml
```

- 3.Uninstall the AnalyticsEngine hot fixes.
<br>
1)Edit the analyticsengine-sample with below command.
  
```
oc edit AnalyticsEngine analyticsengine-sample
```

2)Remove the hot fix images from the AnalyticsEngine custom resource if any.

3)Make change for skipping SELinux Relabeling
```
  serviceConfig:
    skipSELinuxRelabeling: true
```

4)Save and Exit. Wait untile the AnalyticsEngine Operator reconcilation completed and also the analyticsengine-sample in 'Completed' status. 

```
oc get AnalyticsEngine analyticsengine-sample -o yaml
```

- 4.Patch the CCS and uninstall the CCS hot fixes.
<br>

1)Edit the CCS cr with below command.
  
```
oc edit CCS ccs-cr
```

2)Remove the hot fix images from the CCS custom resource

```
  portal_projects_image:
    name: xxxxxx
    tag: 93c38bf9870a5e8f9399b1e90e09f32e5f556d5f6e03b4a447a400eddb08dc4e
    tag_metadata: xxxxxx

  asset_files_api_image:
    name: xxxxxx
    tag: bfa820ffebcf55b87f7827037daee7ec074d0435139e57acbb494df19aee0e98
    tag_metadata: xxxxxx

  catalog_api_image:
    name: xxxxxx
    tag: 4ee6645dd5d9160150f3ad21298e85b28bfe45f6bfff3298861552ccf0897903
    tag_metadata: xxxxxx

  wkc_search_image:
    name: xxxxxx
    tag: 3e95e932b2d2a186cab56b5073e2f9d1b70f1ac24a6ff48c1ae322e8727bdcb3
    tag_metadata: xxxxxx

  portal_catalog_image:
    name: xxxxxx
    tag: 33e51a0c7eb16ac4b5dbbcd57b2ebe62313435bab2c0c789a1801a1c2c00c77d
    tag_metadata: xxxxxx
```
3)Apply preventative measures for OpenSearch pvc customization problem
<br>
This step is for applying the preventative measures for OpenSearch problem. Applying the preventative measures in this timing can also help to minimize the number of CCS operator reconcilations.
<br>
List OpenSearch PVC sizes, and make sure to preserve the type, and the size of the largest one (PVC names may be different depending on client environment):
<br>

```
oc get pvc | grep elasticsea
dev                        data-elasticsea-0ac3-ib-6fb9-es-server-esnodes-0           Bound    pvc-ac773a46-0110-48f6-be27-51b6db332945   150Gi      RWO          
dev                        data-elasticsea-0ac3-ib-6fb9-es-server-esnodes-1           Bound    pvc-1e73b25c-c0ca-4bdd-b5ba-b4484e18ade9   150Gi      RWO
dev                        data-elasticsea-0ac3-ib-6fb9-es-server-esnodes-2           Bound    pvc-1a5a6c47-4221-4ecf-96e4-ddef689fd527   150Gi      RWO
dev                        elasticsea-0ac3-ib-6fb9-es-server-snap                     Bound    pvc-eba99ac7-0f9f-481e-883c-56dc8e9ca65c   608Gi      RWX
dev                        elasticsearch-master-backups                               Bound    pvc-8ab9d47d-de90-449c-99b8-89fed818c727   608Gi      RWX
```

In the above example, `150Gi` is the OpenSearch pvc size, `608Gi` is backup/snapshot storage size. 
<br>
**Note** if PVCs are of different sizes, we want to make sure to take the biggest one. 
<br>

In CCS CR make sure to set the following properties, with above values used as example:

```
elasticsearch_persistence_size: "608Gi"
elasticsearch_backups_persistence_size: "608Gi"
```

This will make sure that the Opensearch operator will properly reconcile, - as provided values will match the state of the cluster. 

4)Remove the `ignoreForMaintenance: true` from the CCS custom resource

5)Save and Exit. Wait untile the CCS Operator reconcilation completed and also the ccs-cr in 'Completed' status. 

```
oc get CCS ccs-cr -o yaml
```

6)Wait untile the WKC Operator reconcilation completed and also the wkc-cr in 'Completed' status. 

```
oc get WKC wkc-cr -o yaml
```

- 5.Edit the ZenService custom resource.

```
oc edit ZenService lite-cr
```

1)Remove the hot fix images from the ZenService custom resource
```
  icp4data_nginx_repo:
    name: xxxxxx
    tag: 2ab2c0cfecdf46b072c9b3ec20de80a0c74767321c96409f3825a1f4e7efb788
    tag_metadata: xxxxxx

  icpd_requisite:
    name: xxxxxx
    tag: 5a7082482c1bcf0b23390d36b0800cc04cfaa32acf7e16413f92304c36a51f02
    tag_metadata: xxxxxx

  privatecloud_usermgmt:
    name: xxxxxx
    tag: e7b0dda15fa3905e4f242b66b18bc9cf2d27ea46e267e5a8d6a3d7da011bddb1
    tag_metadata: xxxxxx

  zen_audit:
    name: xxxxxx
    tag: ccf61039298186555fd18f568e715ca9e12f07805f42eb39008f851500c0f024
    tag_metadata: xxxxxx

  zen_core:
    name: xxxxxx
    tag: 67f4d92a6e1f39675856fe3b46b36b34e9f0ae25679f75a1628c9d7d44790bad
    tag_metadata: xxxxxx

  zen_core_api:
    name: xxxxxx
    tag: b3ba3250a228d5f1ba3ea93ccf8b0f018e557f0f4828ed773b57075b842c30e9
    tag_metadata: xxxxxx

  zen_iam_config:
    name: xxxxxx
    tag: 5abf2bf3f29ca28c72c64ab23ee981e8ad122c0de94ca7702980e1d40841d91a
    tag_metadata: xxxxxx

  zen_minio:
    name: xxxxxx
    tag: f66e6c17d1ed9d90a90e9a1280a18aacb9012bbdb604c5230d97db4cffcb4b48
    tag_metadata: xxxxxx

  zen_utils:
    name: xxxxxx
    tag: 6d906104a8bd8b15f3ebcb2c3ae6a5f93c8d88ce6cfcae4b3eed6657562dc9f3
    tag_metadata: xxxxxx

  zen_watchdog:
    name: xxxxxx
    tag: 4f73b382687bd4de6754292670f6281a7944b6b0903396ed78f1de2da54bc8c0
    tag_metadata: xxxxxx
```

2)Remove the `ignoreForMaintenance: true` from the ZenService custom resource.
<br>
Save and Exit. Wait untile the ZenService Operator reconcilation completed and also the lite-cr in 'Completed' status. 
<br>

- 6.Remove stale secret of global search
Check if the elasticsearch-master-ibm-elasticsearch-cred-secret exists.
```
oc get secret -A | grep elasticsearch-master-ibm-elasticsearch-cred-secret
```
If yes, then delete this stale secret.
```
oc delete elasticsearch-master-ibm-elasticsearch-cred-secret -n ${PROJECT_CPD_INST_OPERANDS}
```

#### 1.1.4 Uninstall the old RSI pathch
1.Run the cpd-cli manage login-to-ocp command to log in to the cluster as a user with sufficient permissions.
```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```
2.Delete the finley-public-env-patch-1-may2024 patch
<br>
Inactivate:
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--patch_name=finley-public-env-patch-1-may2024 \
--state=inactive
```

Delete:
```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--patch_name=finley-public-env-patch-1-may2024
```

### 1.2 Set up client workstation

#### 1.2.1 Prepare a client workstation

1. Prepare a RHEL 9 machine with internet

Create a directory for the cpd-cli utility.
```
export CPD510_WORKSPACE=/ibm/cpd/510
mkdir -p ${CPD510_WORKSPACE}
cd ${CPD510_WORKSPACE}
```

Download the cpd-cli for 5.1.0

```
wget https://github.com/IBM/cpd-cli/releases/download/v14.1.0/cpd-cli-linux-EE-14.1.0.tgz
```

2. Install tools.

```
yum install openssl httpd-tools podman skopeo wget -y
```

```
tar xvf cpd-cli-linux-EE-14.1.0.tgz
mv cpd-cli-linux-EE-14.1.0-1189/* .
rm -rf cpd-cli-linux-EE-14.1.0-1189
```

3. Copy the cpd_vars.sh file used by the CPD 4.8.5 to the folder ${CPD510_WORKSPACE}.

```
cd ${CPD510_WORKSPACE}
cp <the file path of the cpd_vars.sh file used by the CPD 4.8.5 > cpd_vars_510.sh
```
4. Make cpd-cli executable anywhere
```
vi cpd_vars_510.sh
```

Add below two lines into the head of cpd_vars_510.sh

```
export CPD510_WORKSPACE=/ibm/cpd/510
export PATH=${CPD510_WORKSPACE}:$PATH
```

Update the CPD_CLI_MANAGE_WORKSPACE variable

```
export CPD_CLI_MANAGE_WORKSPACE=${CPD510_WORKSPACE}
```

Run this command to apply cpd_vars_510.sh

```
source cpd_vars_510.sh
```

Check out with this commands

```
cpd-cli version
```

Output like this

```
cpd-cli
        Version: 14.1.0
        Build Date: 2024-12-05T14:18:50
        Build Number: 1189
        CPD Release Version: 5.1.0
```
5.Update the OpenShift CLI
<br>
Check the OpenShift CLI version.

```
oc version
```

If the version doesn't match the OpenShift cluster version, update it accordingly.

#### 1.2.2 Update environment variables for the upgrade to Version 5.1.0

```
vi cpd_vars_510.sh
```

1.Locate the VERSION entry and update the environment variable for VERSION. 

```
export VERSION=5.1.0
```

2.Locate the COMPONENTS entry and confirm the COMPONENTS entry is accurate.
```
export COMPONENTS=ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,ws,ws_runtimes,wml,wkc,datastage_ent,datastage_ent_plus,analyticsengine,mantaflow,datalineage,openscale,db2wh,match360
```

Save the changes. <br>

Confirm that the script does not contain any errors. 
```
bash ./cpd_vars_510.sh
```

Run this command to apply cpd_vars_510.sh
```
source cpd_vars_510.sh
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
For details please refer to IBM documentation [Obtaining the olm-utils-v3 image](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=pruirn-obtaining-olm-utils-v3-image)

#### 1.2.4 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image
```
cpd-cli manage restart-container
```
**Note:**
<br>Check and confirm the olm-utils-v3 container is up and running.
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
The output is saved to the list_images.csv file in the work/offline/${VERSION} directory.<br>
Check the output for errors:
```
grep "level=fatal" ${CPD_CLI_MANAGE_WORKSPACE}/work/offline/${VERSION}/list_images.csv
```
The command returns images that are missing or that cannot be inspected which needs to be addressed.

#### 1.2.6 Creating a profile for upgrading the service instances
Create a profile on the workstation from which you will upgrade the service instances. <br>

The profile must be associated with a Cloud Pak for Data user who has either the following permissions:

- Create service instances (can_provision)
- Manage service instances (manage_service_instances)

Click this link and follow these steps for getting it done. 

[Creating a profile to use the cpd-cli management commands](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=cli-creating-cpd-profile)


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
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
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
### 2.1 Upgrade CPD to 5.1.0

#### 2.1.1 Upgrading shared cluster components
1.Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```
2.Confirm the project which the License Service is in.
Run the following command:
```
oc get deployment -A |  grep ibm-licensing-operator
```
Make sure the project returned by the command matches the environment variable PROJECT_LICENSE_SERVICE in your environment variables script `cpd_vars_510.sh`.
<br>

3.Upgrade the Certificate manager and License Service.
```
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--cert_manager_ns=${PROJECT_CERT_MANAGER} \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```
**Note**:
<br><br>Monitor the install plan and approved them as needed.

<br>Confirm that the Certificate manager pods in the ${PROJECT_CERT_MANAGER} project are Running:
```
oc get pod -n ${PROJECT_CERT_MANAGER}
```
Confirm that the License Service pods are Running or Completed::
```
oc get pods --namespace=${PROJECT_LICENSE_SERVICE}
```

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

<br>

**Production enironment**
<br>
Apply the IBM Cloud Pak for Data Enterprise Edition for the production environment.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cpd-enterprise
```

Reference: <br>

[Applying your entitlements](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=puish-applying-your-entitlements)

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
3.Upgrade the required operators and custom resources for the instance.
```
cpd-cli manage setup-instance \
--release=${VERSION} \
--license_acceptance=true \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--run_storage_tests=true
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

[Operator and operand versions](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=planning-operator-operand-versions)

<br>
Increase the resource limits of the CCS operator.

```
oc edit csv ibm-cpd-ccs.v10.0.0 -n ${PROJECT_CPD_INST_OPERATORS} 
```

Make changes to the limits like below.

```
    resources:
      limits:
        cpu: 2
        ephemeral-storage: 2Gi
        memory: 4Gi
```

#### 2.1.5 Applying the RSI patches

1).Log the cpd-cli in to the Red Hat OpenShift Container Platform cluster.
```
${CPDM_OC_LOGIN}
```
2).Run the following command to re-apply your existing custom patches.
```
cpd-cli manage apply-rsi-patches \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```
3).Creat new patches required for migrating profiling results
<br>
a).Identify the location of the `work` directory using below command.

```
podman inspect olm-utils-play-v3 | grep -i -A5  mounts
```

The `Source` property value in the output is the location of the `work` directory.

```
  "Mounts": [
       {
            "Type": "bind",
            "Source": "/ibm/cpd/510/work",
            "Destination": "/tmp/work",
            "Driver": "",
```
For example, `/ibm/cpd/510/work` is the location of the `work` directory.

<br>
b).Create a json patch file named `annotation-spec.json` under `cpd-cli-workspace/olm-utils-workspace/work/rsi` with the following content:

```
[{"op":"add","path":"/metadata/annotations/io.kubernetes.cri-o.TrySkipVolumeSELinuxLabel","value":"true"}]
```

c).Create a json patch file named `specpatch.json` under `cpd-cli-workspace/olm-utils-workspace/work/rsi` with the following content:

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

cat cpd-cli-workspace/olm-utils-workspace/work/get_rsi_patch_info.log
```

### 2.2 Upgrade CPD services to 5.1.0
#### 2.2.1 Upgrading IBM Knowledge Catalog service and apply hot fixes
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
    enableDataQuality: True
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

##### 5.Apply the hotfixes (to be updated)
**Combined Zenservice patch command** (including external vault connection for reducing the number of operator reconcilations): <br>

```
oc patch ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge -p '{"spec":{"image_digests":{"xxxxxx":"sha256:yyyyyy"},"vault_bridge_tls_tolerate_private_ca": true}}'
```

**Combined CCS patch command** (Reducing the number of operator reconcilations): <br>

```
oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type=merge -p '{"spec":{"image_digests":{"xxxxxx":"sha256:yyyyyy","aaaaaa":"sha256:bbbbbb"},"asset_files_call_socket_timeout_ms": 60000, "portal_catalog_is_global_search_relationships_enabled": "true","asset_files_api_resources": {"limits": {"cpu": "4", "memory": "32Gi", "ephemeral-storage": "1Gi"}, "requests": {"cpu": "200m", "memory": "256Mi", "ephemeral-storage": "10Mi"}}, "asset_files_api_replicas": 6,"asset_files_api_command":["/bin/bash"], "asset_files_api_args":["-c","cd /home/node/${MICROSERVICENAME}; source /scripts/exportSecrets.sh; export npm_config_cache=~node; node --max-old-space-size=12288 --max-http-header-size=32768 index.js"]}}'
```

If below customization required, then the patch command needs to be updated. Or the patch command can be changed to the command `oc edit ccs ccs-cr` instead.
```
  asset_files_api_replicas: 6
  asset_files_api_resources:
    limits:
      cpu: "4"
      ephemeral-storage: 1Gi
      memory: 32Gi
    requests:
      cpu: 200m
      ephemeral-storage: 10Mi
      memory: 256Mi
```

**Get the file-api-claim pvc size for preparing the postgresql with the proper storage size to accomendate the profiling migration**
<br>
Check the asset-files-api pvc size. Specify the same or a bigger storage size for Postgres.

```
oc get pvc -n ${PROJECT_CPD_INST_OPERANDS} | grep file-api-claim | awk '{print $4}'
```

**Combined WKC patch command** for reducing the number of operator reconcilations:<br>
**Note:** <br>
Change the file-api-claim pvc size accordingly in below command before running it.

```
oc patch wkc wkc-cr -n ${PROJECT_CPD_INST_OPERANDS} --type=merge -p '{"spec":{"image_digests":{"xxxxxx":"sha256:yyyyyy","aaaaaa":"sha256:bbbbbb"},"wkc_term_assignment_rest_retry_config":"cams\\\\.write:400|3|2;cams\\\\.attachment\\\\.markcomplete:400|3|2,404|3|2;finley\\\\.predict:500|18|10,502|18|10,503|18|10,504|1|10,507|18|10;.*:408|3|2,409|3|2,425|3|2,429|3|2,500|6|10,502|6|10,503|6|10,504|6|10,507|6|10,-1|3|2","wkc_term_assignment_finley_page_size_reduction_divisor":"5","finley_public_gunicorn_worker_timeout":"65","wdp_profiling_flight_enabled":"false","wdp_profiling_edb_postgres_storage_size":"500Gi"}}}'

```

#### 2.2.2 Upgrading MANTA service
```
export COMPONENTS=mantaflow
```

- Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
${CPDM_OC_LOGIN}
```

- Remove the `migrations` section from the mantaflow custom resource:
<br>

Run the following command:

```
oc patch mantaflow mantaflow-wkc --type='json' -p='[{ "op": "remove", "path": "/spec/migrations"}]'
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

The Analytics Engine serive should have been upgraded as part of the WKC service upgrade. If the Analytics Engine service version is **not 5.1.0**, then run below commands for the upgrade. <br>

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

### 3.1 Validate the external vault connection setting 
1)Validate and ensure the patch for external vault connection applied.

Found out the following variables set to false
```
oc set env deployment/zen-core-api --list | grep -i vault
```
The values are true like this:
```
VAULT_BRIDGE_TOLERATE_SELF_SIGNED=true
VAULT_BRIDGE_TLS_RENEGOTIATE=true
```

### 3.2 CCS post-upgrade tasks
**1.Check if uploading JDBC drivers enabled**
```
oc get ccs ccs-cr -o yaml | grep -i wdp_connect_connection_jdbc_drivers_repository_mode
```
Make sure the `wdp_connect_connection_jdbc_drivers_repository_mode` parameter set to be enabled.

**2.Change heap size in asset-files-api deployment**
<br>
**Need to consider including this as part of the application of hotfixes **
<br>
1)Patch the CCS cr using below command.
```
oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type=merge -p '{"spec":{"asset_files_api_command":["/bin/bash"], "asset_files_api_args":["-c","cd /home/node/asset-files-api\nsource /scripts/exportSecrets.sh\nexport npm_config_cache=~node\nnode --max-old-space-size=12288 --max-http-header-size=32768 index.js\n"]}}'
```
2)Wait until the CCS operator reconcilation completed.
<br>
2)Double check if the heap size change is set as expected.
```
oc get deployment/catalog-api --list | grep -i "--max-old-space-size=12288" -A 5 -B 5
```

### 3.3 WKC post-upgrade tasks

**1.Enable 'Allow Reporting' settings for Catalogs and Projects**

<br>

1)Put wkc-cr in maintenance mode.
```
oc patch wkc wkc-cr --type=merge --patch='{"spec":{"ignoreForMaintenance":true}}'
```
2)Set environment variable ENFORCE_AUTHORIZE_REPORTING for the wkc-bi-data-service deployment
```
oc set env deployment/wkc-bi-data-service ENFORCE_AUTHORIZE_REPORTING=true
```
3)Double check if ENFORCE_AUTHORIZE_REPORTING is set to be true.
```
oc set env deployment/wkc-bi-data-service --list | grep -i ENFORCE_AUTHORIZE_REPORTING
```

**Note: the above steps may need to be changed by following below documentation. Need to consider including this as part of the application of hotfixes**

[Configuring reporting settings for IBM Knowledge Catalog](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=administering-configuring-reporting-settings)

**2.Enable Relationship Explorer feature**
<br>

[Exploring relationships](https://www.ibm.com/docs/en/cloud-paks/cp-data/5.1.x?topic=catalog-exploring-relationships)

**3.Apply the workaround for the problem - MDE Job failed with error "Deployment not found with given id"**
<br>
1)Put analyticsengine-sample in maintenance mode.
```
oc patch analyticsengine analyticsengine-sample --type=merge --patch='{"spec":{"ignoreForMaintenance":true}}'
```
2)Edit the `spark-hb-deployment-properties` config map and add the property `deploymentStatusRetryCount=6`
```
oc edit cm spark-hb-deployment-properties
```
3)Make sure the property `deploymentStatusRetryCount=6` added successfully
```
oc get cm spark-hb-deployment-properties -o yaml | grep -i deploymentStatusRetryCount
```

**4.Bulk sync relationships for global search**
<br>
To be able to use the global search indexed data for relationships, see [Bulk sync relationships for global search](https://www.ibm.com/docs/en/SSNFH6_5.1.x/wsj/admin/admin-bulk-sync-rel.html).

**5.Bulk sync assets for global search**
<br>
To be able to use the global search indexed data for assets, see [Bulk sync assets for global search](https://www.ibm.com/docs/en/SSNFH6_5.1.x/wsj/admin/admin-bulk-sync.html).

**6.Add potential missing permissions for the pre-defined Data Quality Analyst and Data Steward roles**
<br>

```
oc delete pod $(oc get pod -n ${PROJECT_CPD_INST_OPERANDS} -o custom-columns="Name:metadata.name" -l app.kubernetes.io/component=zen-watcher --no-headers) -n ${PROJECT_CPD_INST_OPERANDS}
```

**7. Migrating profiling results after upgrading**
<br>
In Cloud Pak for Data 5.1.0, profiling results are stored in a PostgreSQL database instead of the asset-files storage. To make existing profiling results available after upgrading from an earlier release, migrate the results following this IBM documentation.
[Migrating profiling results after upgrading](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=administering-migrating-profiling-results)

<br>

Sample override.yaml file:

```
namespace: cpd
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
1.Check the asset-files-api pvc size. Specify the same or a bigger storage size for Postgres.
```
oc patch wkc wkc-cr -n ${PROJECT_CPD_INSTANCE} --type=merge -p '{"spec":{"wdp_profiling_edb_postgres_storage_size":"500Gi"}}'
```

2.The nohup command is recommended for the migration of a large number of records.
```
nohup ansible-playbook /opt/ansible/5.1.0/roles/wkc-core/wdp_profiling_postgres_migration.yaml --extra=@/tmp/override.yaml -vvvv &
```

**8. Monitor the global asset type definition update process**
<br>
Run a metadata import job and then check whether there are multiple wkc-search-reindexing-resource-key-combined-job jobs:
```
oc get pod | grep wkc-search-reindexing-resource-key-combined-job | grep Running | wc -l
```

If there are multiple wkc-search-reindexing-resource-key-combined-job jobs, keep the oldest job and delete all of the newer jobs. To delete a job, run:
```
oc delete job <job-name>
```

[Reference](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=issues-common-core-services#global-asset-type-update)


## Part 4: Maintenance
### 4.1 Migrating from MANTA Automated Data Lineage to IBM Manta Data Lineage
1.Install the IBM Manta Data Lineage
<br>
2.Migrating from MANTA Automated Data Lineage to IBM Manta Data Lineage
[Migrating from MANTA Automated Data Lineage to IBM Manta Data Lineage](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=migrating-from-manta-automated-data-lineage-manta-data-lineage)

### 4.2 Changing Db2 configuration settings
1.Run the following command to edit the Db2uCluster custom resource:
```
oc edit db2ucluster db2oltp-wkc -n ${PROJECT_CPD_INST_OPERANDS}
```
2.Change the following database configuration parameters.
<br>
To find your database configuration parameters, see the yaml path `spec.environment.database.dbConfig`.
Under the key dbConfig, you can add or edit below key-value pairs. Values must be enclosed in quotation marks.
```
spec:
  environment:
    database:
      dbConfig:
        LOGFILSIZ: '30000'
        LOGPRIMARY: '40'
        LOGSECOND: '60'
        CATALOGCACHE_SZ: '567'
```

**Note**
<br>
It's recommended getting this done by following the configuration settings in the Production environment.
<br>
[Changing Db2 configuration settings](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=configuration-changing-db2-settings)

### 4.3 Configure the idle session timeout
Update TOKEN_EXPIRY_TIME and TOKEN_REFRESH_PERIOD variables from 4to 12 hours.
 - oc edit configmap product-configmap ......
 - Restart the usermgmt pods

[Setting the idle session timeout](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=environment-setting-idle-session-timeout)

### 4.4 Increasing the number of nginx worker connections
[Reference](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=platform-increasing-number-nginx-worker-connections)

### 4.5 Increase ephemeral storage for zen-watchdog-serviceability-job
1)Keep a copy of the product-configmap CM.
2)Update the product-configmap for zen_diagnostics with below command.
```
oc edit configmap product-configmap
```
Add the zen_diagnostics sections as follows.
```
data:
  zen_diagnostics: |
    name: zen-diagnostics
    kind: Job
    container: zen-watchdog-serviceability-job-container
    resources:
      limits:
        cpu: 1
        ephemeral-storage: 35000Mi
        memory: 4Gi
      requests:
        cpu: 500m
        ephemeral-storage: 500Mi
        memory: 512Mi
```
3)Restart zen-watchdog pod

### 4.6 Upgrade the Backup & Restore service and application
**Note:** This will be done as a separate task in another maintenance time window.

**1.Updating the cpdbr service**
<br>

If you use IBM Fusion to back up and restore your IBM® Software Hub deployment, you must upgrade the cpdbr service after you upgrade IBM Cloud Pak® for Data Version 4.8 to IBM Software Hub Version 5.1.

[Updating the cpdbr service](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=data-updating-cpdbr-service)

<br>

**2.Upgrade the IBM Fusion application**
<br>
IBM Fusion team can help on this task.

## Summarize and close out the upgrade

1)Schedule a wrap-up meeting and review the upgrade procedure and lessons learned from it.

2)Evaluate the outcome of upgrade with pre-defined goals.

---

End of document
