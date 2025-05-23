# CPD Upgrade Runbook - v.4.8.5 to 5.1.1

---

## Upgrade documentation

[Upgrading from IBM Cloud Pak for Data Version 4.8 to Version 5.1](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=upgrading-from-cloud-pak-data-version-48)

## Upgrade context

From

```
OCP: 4.14
CPD: 4.8.5
Storage: Storage Fusion 2.7.2
Componenets: cpd_platform,wkc,analyticsengine,mantaflow,datalineage,ws,ws_runtimes,wml,openscale,db2wh
```

To

```
OCP: 4.14
CPD: 5.1.1
Storage: Storage Fusion 2.7.2
Componenets: cpd_platform,wkc,analyticsengine,mantaflow,datalineage,ws,ws_runtimes,wml,openscale,db2wh
```

## Pre-requisites

#### 1. Backup of the cluster is done.

Backup your Cloud Pak for Data cluster before the upgrade.
For details, see [Backing up and restoring Cloud Pak for Data](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=administering-backing-up-restoring-software-hub).

**Note:**
Make sure there are no scheduled backups conflicting with the scheduled upgrade.

#### 2. The image mirroring completed successfully

If a private container registry is in-use to host the IBM Cloud Pak for Data software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry. `<br>`
Reference: `<br>`
[Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=prufpcr-mirroring-images-private-container-registry)
`<br>`

#### 3. The permissions required for the upgrade is ready

- Openshift cluster permissions
  `<br>`
  An Openshift cluster administrator can complete all of the installation tasks.

<br>

However, if you want to enable users with fewer permissions to complete some of the installation tasks, follow the steps in this documentation and get the roles with required permission prepared.

[Reauthorizing the instance administrator](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=hub-reauthorizing-instance-administrator)

<br>

- Cloud Pak for Data permissions

<br>
The Cloud Pak for Data administrator role or permissions is required for upgrading the service instances.

- Permission to access the private image registry for pushing or pull images
- Access to the Bastion node for executing the upgrade commands

#### 4. Migrate environments based on Watson Studio Runtime 22.2 and Runtime 23.1 from IBM Cloud Pak® for Data 4.8 (optional)

The Watson Studio Runtime 22.2 and Runtime 23.1 are not included in IBM® Software Hub. If you want to continue using environments that are based on Runtime 22.2 or Runtime 23.1, you must migrate them.
`<br>`
[Migrating environments based on Runtime 22.2 and Runtime 23.1 from IBM Cloud Pak® for Data 4.8](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=u-migrating-environments-based-runtime-222-runtime-231-from-cloud-pak-data-48-50)

#### 5. Collect the number of profiling records to be migrated

Collect profiling records information

```
oc project ${PROJECT_CPD_INST_OPERANDS}

oc rsh $(oc get pods --no-headers | grep -i asset-files | head -n 1 | awk '{print $1}')

nohup ls -alRt /mnt/asset_file_api | egrep -i 'REF_|ARES_' | wc -l > /tmp/profiling_records_ref_number.txt &

```

#### 6. Collect the number about Neo4J database size of Manta Automated Data Lineage

```
oc rsh $(oc get pods --no-headers | grep -i manta-dataflow | head -n 1 | awk '{print $1}')
nohup du -hs /opt/mantaflow/server/manta-dataflow-server-dir/data/neo4j/data > /tmp/neo4j_db_size.txt &

```

#### 7. A pre-upgrade health check is made to ensure the cluster's readiness for upgrade.

- The OpenShift cluster, persistent storage and Cloud Pak for Data platform and services are in healthy status.

#### 8. Download the MDI Lineage Migration Toolkit Patch and images

- Download the MDI Lineage Migration Toolkit Patch

<br>

[Download the patch from the Fix Central](https://www.ibm.com/support/fixcentral/quickorder?product=ibm%2FInformation+Management%2FIBM+InfoSphere+Information+Server&fixids=mdi-lineage-migration-patch_5112&source=SAR)

<br>
Copy the MDI Lineage Migration Toolkit Patch to below `${MDIWORK_DIR}` on the Cloud Pak for Data cluster.

```
MDIWORK_DIR=/tmp/mdiwork
mkdir -p ${MDIWORK_DIR}
```

- Mirror the images

<br>

You need to change the path to the `auth.json` (credentials) and the `PRIVATE_REGISTRY_LOCATION` in below commands.

```
DOCKER_IMAGE_ID=6e6b800e8ca2d096e8b36767656ec2f335b7df009861e66355bb8f67b27713cf
PRIVATE_REGISTRY_LOCATION=<your private image registry URL>

skopeo copy --all --authfile "<folder path>/auth.json" \
    --dest-tls-verify=false --src-tls-verify=false \
    docker://cp.icr.io/cp/cpd/mdi-lineage-migration@sha256:${DOCKER_IMAGE_ID} \
    docker://${PRIVATE_REGISTRY_LOCATION}/cp/cpd/mdi-lineage-migration@sha256:${DOCKER_IMAGE_ID}


skopeo copy --all --authfile "<folder path>/auth.json" \
    --dest-tls-verify=false --src-tls-verify=false \
    docker://icr.io/cpopen/cpd/cpdtool:5.1.1 \
    docker://${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/cpdtool:5.1.1
```

#### 9. Check and ensure there's no high storage utilization alerts

<br>
This check should be done both from the storge cluster and PVC level.
<br>
The storage utilization should be less than 70%.

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

### 1.1 Collect information and review upgrade runbook

#### 1.1.1 Review the upgrade runbook

Review upgrade runbook

#### 1.1.2 Backup before upgrade

Note: Create a folder for 4.8.5 and maintain below created copies in that folder. `<br>`
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

for i in $(oc get crd | grep cpd.ibm.com | awk '{ print $1 }'); do echo "---------$i------------"; oc get $i $(oc get $i | grep -v "NAME" | awk '{ print $1 }') -o yaml > cr-$i.txt; if grep -q "image_digests" cr-$i.txt; then echo "Hot fix detected in cr-$i"; fi; done

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

#### 1.1.3 Uninstall all hotfixes and apply preventative measures

Remove the hotfixes by removing the images or configurations from the CRs.
`<br>`

- 1.Uninstall WKC hot fixes.

<br>

1)Edit the wkc-cr with below command.

```
oc edit WKC wkc-cr
```

2)Remove the hot fix images from the WKC custom resource

```
  image_digests:
    finley_public_image: sha256:e89b59e16c4c10fce5ae07774686d349d17b2d21bf9263c50b49a7a290499c6d
    metadata_discovery_image: sha256:9b1811703c2639a822e0d3a2c466ee2a6378bbd4f706da66ed1b61cb96971610
    wdp_kg_ingestion_service_image: sha256:bb6c382edf1da01335152da98b803f44b424e8651c862eaf96b6394d8e735e6f
    wdp_lineage_image: sha256:9eda6aaf3dd2581fc7a85746e4b559192b4b00aa5cd7bc227cb29e7873a3cea1
    wdp_profiling_image: sha256:d5491cc8c8c8bd45f2605f299ecc96b9461cd5017c7861f22735e3a4d0073abd
    wkc_mde_service_manager_image: sha256:35e6f6ede3383df5c0b2a3d27c146cc121bed32d26ab7fa8870c4aa4fbc6e993
    wkc_metadata_imports_ui_image: sha256:a1997d9a9cde9ecc9f16eb02099a272d7ba2e8d88cb05a9f52f32533e4d633ef
```

3)Remove the `ignoreForMaintenance: true` from the WKC custom resource

```
ignoreForMaintenance: true
```

4)Save and Exit. Wait untile the WKC Operator reconcilation completed and also the wkc-cr in 'Completed' status.

```
oc get WKC wkc-cr -o yaml
```

- 2.Uninstall the AnalyticsEngine hot fixes if any.
  `<br>`
  1)Edit the analyticsengine-sample with below command.

```
oc edit AnalyticsEngine analyticsengine-sample
```

2)Remove the hot fix images from the AnalyticsEngine custom resource if any.

3)Check the setting for skipping SELinux Relabeling and make change if needed.

```
  serviceConfig:
    skipSELinuxRelabeling: true
```

4)Save and Exit. Wait untile the AnalyticsEngine Operator reconcilation completed and also the analyticsengine-sample in 'Completed' status.

```
oc get AnalyticsEngine analyticsengine-sample -o yaml
```

- 3.Patch the CCS and uninstall the CCS hot fixes.
  `<br>`

1)Apply preventative measures for potential time-consuming CCS reconcilation caused by large number of cronjobs.

Create a directory for the backup of cronjobs.

```
mkdir cronjob_bak
cd cronjob_bak
```

**Important:**

<br>

Backup of all cronjob

```
oc project ${PROJECT_CPD_INST_OPERANDS} 
for cj in $(oc get cronjob -l runtimeAssembly --no-headers | awk '{print $1}'); do oc get cronjob $cj -oyaml >  $cj.yaml;done
```

Deleting label from all cronjobs

```
for cj in $(oc get cronjob -l runtimeAssembly --no-headers | awk '{print $1}'); do oc label cronjob $cj created-by- 2>/dev/null; done
```

Return to the parent directory.

```
cd ..
```

2)Edit the CCS cr with below command.

```
oc edit CCS ccs-cr
```

3)Remove the hot fix images from the CCS custom resource

```
  image_digests:
    asset_files_api_image: sha256:bfa820ffebcf55b87f7827037daee7ec074d0435139e57acbb494df19aee0e98
    catalog_api_image: sha256:d64c61cbc010d7535b33457439b5cb65c276346d4533963a9a5165471840beb4
    portal_catalog_image: sha256:33e51a0c7eb16ac4b5dbbcd57b2ebe62313435bab2c0c789a1801a1c2c00c77d
    portal_projects_image: sha256:93c38bf9870a5e8f9399b1e90e09f32e5f556d5f6e03b4a447a400eddb08dc4e
    wkc_search_image: sha256:64e59002617d48428cd59a55bbad5ebf0ccf68644fd627fd1e33f6558dbc8b68
```

4)Apply preventative measures for OpenSearch pvc customization problem
`<br>`
This step is for applying the preventative measures for OpenSearch problem. Applying the preventative measures in this timing can also help to minimize the number of CCS operator reconcilations.
`<br>`
List OpenSearch PVC sizes, and make sure to preserve the type, and the size of the largest one (PVC names may be different depending on client environment):
`<br>`

```
oc get pvc | grep elasticsea

hptv-prodcloudpak          data-elasticsea-0ac3-ib-6fb9-es-server-esnodes-0           Bound    pvc-f9b87faf-f288-4f08-a290-585ab7d4f9bc   1407Gi     RWO            ocs-storagecluster-ceph-rbd   236d
hptv-prodcloudpak          data-elasticsea-0ac3-ib-6fb9-es-server-esnodes-1           Bound    pvc-d91991a0-9ffc-46b6-8578-1f0826092caa   1407Gi     RWO            ocs-storagecluster-ceph-rbd   236d
hptv-prodcloudpak          data-elasticsea-0ac3-ib-6fb9-es-server-esnodes-2           Bound    pvc-28839878-c327-4b2b-9b9b-be17ea8fa7ff   1407Gi     RWO            ocs-storagecluster-ceph-rbd   236d
hptv-prodcloudpak          elasticsea-0ac3-ib-6fb9-es-server-snap                     Bound    pvc-856121a4-fdbc-4d24-a25a-9b73ba1129ab   3078Gi     RWX            ocs-storagecluster-cephfs     236d
hptv-prodcloudpak          elasticsearch-master-backups                               Bound    pvc-11bc9f8e-65de-4d3c-a855-ba1358bb3a77   3078Gi     RWX            ocs-storagecluster-cephfs     579d  

```

In the above example, `1407Gi` is the OpenSearch pvc size. `3078Gi` is the backup/snapshot storage size.
`<br>`
**Note** if PVCs are of different sizes, we want to make sure to take the biggest one.
`<br>`

In CCS CR make sure to set the following properties, with above values used as example:

```
elasticsearch_persistence_size: "3078Gi"
elasticsearch_backups_persistence_size: "3078Gi"
```

This will make sure that the Opensearch operator will properly reconcile, - as provided values will match the state of the cluster.

5)Disable bulk resync during the upgrade. This job can be run separately (if its needed) after upgrade has completed. Set the following properties in the spec section of CCS CR.

```
run_reindexer_with_resource_key: false
```

6)Increasing the resource limits for the `search` container of the CouchDb. This can help accelerate the CCS upgrade. Set the following property in the spec section of CCS CR. The changes can be reverted after the upgrade.

```
couchdb_search_resources:
  limits:
    cpu: "6"
    memory: 12Gi
  requests:
    cpu: 250m
    memory: 256Mi
```

7)Remove the `ignoreForMaintenance: true` from the CCS custom resource

8)Save and Exit. Wait untile the CCS Operator reconcilation completed and also the ccs-cr in 'Completed' status.

```
oc get CCS ccs-cr -o yaml
```

9)Wait untile the WKC Operator reconcilation completed and also the wkc-cr in 'Completed' status.

```
oc get WKC wkc-cr -o yaml
```

- 4.Edit the ZenService custom resource.

```
oc edit ZenService lite-cr
```

1)Remove the hot fix images from the ZenService custom resource

```
  image_digests:
    icp4data_nginx_repo: sha256:2ab2c0cfecdf46b072c9b3ec20de80a0c74767321c96409f3825a1f4e7efb788
    icpd_requisite: sha256:5a7082482c1bcf0b23390d36b0800cc04cfaa32acf7e16413f92304c36a51f02
    privatecloud_usermgmt: sha256:e7b0dda15fa3905e4f242b66b18bc9cf2d27ea46e267e5a8d6a3d7da011bddb1
    zen_audit: sha256:ccf61039298186555fd18f568e715ca9e12f07805f42eb39008f851500c0f024
    zen_core: sha256:67f4d92a6e1f39675856fe3b46b36b34e9f0ae25679f75a1628c9d7d44790bad
    zen_core_api: sha256:b3ba3250a228d5f1ba3ea93ccf8b0f018e557f0f4828ed773b57075b842c30e9
    zen_iam_config: sha256:5abf2bf3f29ca28c72c64ab23ee981e8ad122c0de94ca7702980e1d40841d91a
    zen_minio: sha256:f66e6c17d1ed9d90a90e9a1280a18aacb9012bbdb604c5230d97db4cffcb4b48
    zen_utils: sha256:6d906104a8bd8b15f3ebcb2c3ae6a5f93c8d88ce6cfcae4b3eed6657562dc9f3
    zen_watchdog: sha256:4f73b382687bd4de6754292670f6281a7944b6b0903396ed78f1de2da54bc8c0
```

2)Add the `gcMemoryLimit` configurtion under the ZenMinio in the spec which looks like below.
`<br>`

```
  ZenMinio:
    name: zen-minio
    gcMemoryLimit: 2000MiB
```

3)Add the `GATEWAY_WORKER_CONNECTIONS` in the spec

```
GATEWAY_WORKER_CONNECTIONS: "4096"
```

4)Change the `refresh_install` parameter to be `false`

```
refresh_install: false
```

5)Save and Exit. Wait untile the ZenService Operator reconcilation completed and also the lite-cr in 'Completed' status.
`<br>`

- 5.Edit the mantaflow custom resource.

```
oc edit mantaflow mantaflow-wkc
```

1)Remove the migration section from the cr.

```
 migrations: 
     h2-format-3: true
```

2)Remove the `ignoreForMaintenance: true` from the CCS custom resource.

<br>

3)Save and Exit. Wait untile the Mantaflow Operator reconcilation completed and also the mantaflow-wkc in 'Completed' status.

<br>

Restart the deployment.

```
oc delete deploy manta-admin-gui manta-configuration-service manta-dataflow -n ${PROJECT_CPD_INST_OPERANDS}
```

- 6.Remove stale secret of global search
  Check if the elasticsearch-master-ibm-elasticsearch-cred-secret exists.

```
oc get secret -n ${PROJECT_CPD_INST_OPERANDS} | grep elasticsearch-master-ibm-elasticsearch-cred-secret
```

If yes, then delete this stale secret.

```
oc delete secret elasticsearch-master-ibm-elasticsearch-cred-secret -n ${PROJECT_CPD_INST_OPERANDS}
```

#### 1.1.4 Uninstall the old RSI patch

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster as a user with sufficient permissions.

```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```

2.Delete the finley-public-env-patch-1-may2024 patch
`<br>`
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
export CPD511_WORKSPACE=/ibm/cpd/511
mkdir -p ${CPD511_WORKSPACE}
cd ${CPD511_WORKSPACE}

```

Download the cpd-cli for 5.1.1

```
wget https://github.com/IBM/cpd-cli/releases/download/v14.1.1/cpd-cli-linux-EE-14.1.1.tgz
```

2. Install tools.

```
yum install openssl httpd-tools podman skopeo wget -y
```

The version in below commands may need to be updated accordingly.

```
tar xvf cpd-cli-linux-EE-14.1.1.tgz
mv cpd-cli-linux-EE-14.1.1-1650/* .
rm -rf cpd-cli-linux-EE-14.1.1-1650
```

3. Copy the cpd_vars.sh file used by the CPD 4.8.5 to the folder ${CPD511_WORKSPACE}.

```
cd ${CPD511_WORKSPACE}
cp <the file path of the cpd_vars.sh file used by the CPD 4.8.5 > cpd_vars_511.sh
```

4. Make cpd-cli executable anywhere

```
vi cpd_vars_511.sh
```

Add below two lines into the head of cpd_vars_511.sh

```
export CPD511_WORKSPACE=/ibm/cpd/511
export PATH=${CPD511_WORKSPACE}:$PATH
```

Update the CPD_CLI_MANAGE_WORKSPACE variable

```
export CPD_CLI_MANAGE_WORKSPACE=${CPD511_WORKSPACE}
```

Run this command to apply cpd_vars_511.sh

```
source cpd_vars_511.sh
```

Check out with this commands

```
cpd-cli version
```

Output like this

```
cpd-cli
	Version: 14.1.1
	Build Date: 2025-02-20T18:45:49
	Build Number: 1650
	CPD Release Version: 5.1.1
```

5.Update the OpenShift CLI
`<br>`
Check the OpenShift CLI version.

```
oc version
```

If the version doesn't match the OpenShift cluster version, update it accordingly.

#### 1.2.2 Update environment variables for the upgrade to Version 5.1.1

```
vi cpd_vars_511.sh
```

1.Locate the VERSION entry and update the environment variable for VERSION.

```
export VERSION=5.1.1
```

2.Locate the COMPONENTS entry and confirm the COMPONENTS entry is accurate.

```
export COMPONENTS=ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,ws,ws_runtimes,wml,wkc,analyticsengine,mantaflow,datalineage,openscale,db2wh
```

Save the changes. `<br>`

Confirm that the script does not contain any errors.

```
bash ./cpd_vars_511.sh
```

Run this command to apply cpd_vars_511.sh

```
source cpd_vars_511.sh
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

The output is saved to the list_images.csv file in the work/offline/${VERSION} directory.`<br>`
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

### 2.1 Upgrade CPD to 5.1.1

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

Make sure the project returned by the command matches the environment variable PROJECT_LICENSE_SERVICE in your environment variables script `cpd_vars_511.sh`.
`<br>`

3.Upgrade the Certificate manager and License Service.

```
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--cert_manager_ns=${PROJECT_CERT_MANAGER} \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```

**Note**:
`<br><br>`Monitor the install plan and approved them as needed.
`<br>`
In another terminal, keep running below command and monitoring "InstallPlan" to find which one need manual approval.

```
watch "oc get ip -n ${PROJECT_CPD_INST_OPERATORS} -o=jsonpath='{.items[?(@.spec.approved==false)].metadata.name}'"
```

Approve the upgrade request and run below command as soon as we find it.

```
oc patch installplan $(oc get ip -n ${PROJECT_CPD_INST_OPERATORS} -o=jsonpath='{.items[?(@.spec.approved==false)].metadata.name}') -n ${PROJECT_CPD_INST_OPERATORS} --type merge --patch '{"spec":{"approved":true}}'
```

`<br>`Confirm that the Certificate manager pods in the ${PROJECT_CERT_MANAGER} project are Running:

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
`<br>`
**Non-Production enironment**
`<br>`
Apply the IBM Cloud Pak for Data Enterprise Edition for the non-production environment.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cpd-enterprise
```

Apply the IBM Manta Data Lineage license for the non-production environment.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=data-lineage
```

Reference: `<br>`

[Applying your entitlements](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=puish-applying-your-entitlements)

#### 2.1.3 Upgrading to IBM Software Hub

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
${CPDM_OC_LOGIN}
```

2.Review the license terms for the software that is installed on this instance of IBM Software Hub.
`<br>`
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

4.Create a custom route

<br>

- Changing the hostname of the route

<br>

Ensure the `custom_hostname` and `route_secret` are set properly before running belwo command.

```
cpd-cli manage setup-route \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--custom_hostname=<The route of your production environment> \
--route_type=passthrough \
--route_secret=<your-external-tls-secret-name>
```

[Create a custom route using cpd-cli](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=manage-setup-route)

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

Increase the resource limits of the CCS operator for avoiding potention problems when dealing with large data volume.

<br>

Have a backup of the CCS CSV yaml file.

```
oc get csv ibm-cpd-ccs.v10.1.0 -n ${PROJECT_CPD_INST_OPERATORS} -o yaml > ibm-cpd-ccs-csv-511.yaml
```

Edit the CCS CSV:

```
oc edit csv ibm-cpd-ccs.v10.1.0 -n ${PROJECT_CPD_INST_OPERATORS} 
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

##### Hotfix for WKC Operator to handle useFDB properly

Back up the Operator image digest value:

```bash
oc get csv -n ${PROJECT_CPD_INST_OPERATORS} ibm-cpd-wkc.v2.1.1 -o yaml | grep "image: icr.io/cpopen/ibm-cpd-wkc-operator@sha256" > 5.1.1_wkc_operator_image.yaml
```

Patch and label the CSV:

```bash
oc patch csv -n ${PROJECT_CPD_INST_OPERATORS}  ibm-cpd-wkc.v2.1.1 --type='json' -p='[{"op":"replace","path":"/spec/install/spec/deployments/0/spec/template/spec/containers/0/image","value":"icr.io/cpopen/ibm-cpd-wkc-operator@sha256:408aa1070613b7be3c4515a657dd48c81d4f09f613e94664b4eaa43ddb8d0632"}]'

oc label csv ibm-cpd-wkc.v2.1.1 support.operator.ibm.com/hotfix=true -n ${PROJECT_CPD_INST_OPERATORS} 
```

Ensure the Operator is up and running with the new image :

```
oc get pods -n ${PROJECT_CPD_INST_OPERATORS} | grep -i ibm-cpd-wkc-operator | grep -v "catalog"

oc describe pod $(oc get pods -n ${PROJECT_CPD_INST_OPERATORS} | grep -i ibm-cpd-wkc-operator | grep -v "catalog" | awk '{print $1}') -n ${PROJECT_CPD_INST_OPERATORS} | grep "Image ID" 
```

Wait for WKC Operator to fully reconcile:

```bash
oc get wkc wkc-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

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
`<br>`
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
`<br>`
1)Make sure you edit or create the `install-options.yml` file in the right `work` folder.

<br>

Identify the location of the `work` folder using below command.

```
podman inspect olm-utils-play-v3 | grep -i -A5  mounts
```

The `Source` property value in the output is the location of the `work` folder.

<br>

2)Make sure the `useFDB` is set to be `True` in the install-options.yml file.
`<br>`

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

**1).Apply the change for supporting CyberArk Vault with a private CA signed certificate**: `<br>`

```
oc patch ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge -p '{"spec":{"vault_bridge_tls_tolerate_private_ca": true}}'
```

**2)Resolve Mismatch from Catalog API**

```
oc project ${PROJECT_CPD_INST_OPERANDS}
```

[Run the script for resolving the mismatch from Catalog API](https://github.com/jhwhenry/cpd/blob/main/delete_rabbitmq_queues.sh.zip)

**3)Combined CCS patch command** (Reducing the number of operator reconcilations): `<br>`

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

**4)Combined WKC patch command** (Reducing the number of operator reconcilations): `<br>`

- Figure out a proper PVC size for the PostgreSQL used by profiling migration.
  `<br>`
  Check the asset-files-api pvc size. Specify the same or a bigger storage size for preparing the postgresql with the proper storage size to accomendate the profiling migration.
  `<br>`
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

The Analytics Engine serive should have been upgraded as part of the WKC service upgrade. If the Analytics Engine service version is **not 5.1.1**, then run below commands for the upgrade. `<br>`

Check if the Analytics Engine service was installed with the custom install options. `<br>`

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
`<br>`
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
`<br>`
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
`<br>`
Check if the heap size 12288 is set as expected.

```
oc get deployment asset-files-api -o yaml | grep -i -A5 'max-old-space-size=12288'
```

### 3.3 WKC post-upgrade tasks

**1.Validate the 'Allow ' settings for Catalogs and Projects**
`<br>`
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
`<br>`
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
`<br>`

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
`<br>`

```
oc delete pod $(oc get pod -n ${PROJECT_CPD_INST_OPERANDS} -o custom-columns="Name:metadata.name" -l app.kubernetes.io/component=zen-watcher --no-headers) -n ${PROJECT_CPD_INST_OPERANDS}
```

**4. Migrating profiling results after upgrading**
`<br>`
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
`<br>`
1).The nohup command is recommended for the migration of a large number of records.

```
nohup ansible-playbook /opt/ansible/5.1.1/roles/wkc-core/wdp_profiling_postgres_migration.yaml --extra=@/tmp/override.yaml -vvvv &
```

2).Validate the job log for successful migration of profiling data; then run the `CLEAN` option.
`<br>`
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
`<br>`

- Migration needs to be run as root or by a user with sudo access.

[Migrating from MANTA Automated Data Lineage to IBM Manta Data Lineage](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=lineage-migrating)

#### 4.1.5 Post-migration tasks

**1. Resync glossary assets**
`<br>`
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

## Summarize and close out the upgrade

1)Schedule a wrap-up meeting and review the upgrade procedure and lessons learned from it.

2)Evaluate the outcome of upgrade with pre-defined goals.

---

End of document
