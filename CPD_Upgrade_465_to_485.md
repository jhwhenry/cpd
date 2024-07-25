# CPD Upgrade Runbook - v.4.6.5 to 4.8.5

---
## Upgrade documentation
[Upgrading from IBM Cloud Pak for Data Version 4.6 to Version 4.8](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=upgrading-from-cloud-pak-data-version-46)

## Upgrade context
From

```
OCP: 4.12
CPD: 4.6.5
Storage: Storage Fusion 2.7.2
Componenets: cpfs,cpd_platform,ws,ws_runtimes,wml,wkc,analyticsengine,openscale,db2wh
```

To

```
OCP: 4.12
CPD: 4.8.5
Storage: Storage Fusion 2.7.2
Componenets: cpfs,cpd_platform,ws,ws_runtimes,wml,wkc,analyticsengine,openscale,db2wh
```

## Pre-requisites
#### 1. Backup of the cluster is done.
Backup your Cloud Pak for Data installation before you upgrade.
For details, see Backing up and restoring Cloud Pak for Data (https://www.ibm.com/docs/en/SSQNUZ_4.8.x/cpd/admin/backup_restore.html).

**Note:**
Make sure there are no scheduled backups conflicting with the scheduled upgrade.
  
#### 2. The image mirroring completed successfully
If a private container registry is in-use to host the IBM Cloud Pak for Data software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry. <br>
Reference: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=48-preparing-run-upgrades-from-private-container-registry <br>

**Note:**
There are some special images required to be mirrored. Please follow below steps for the image mirroring. <br>

1) Log in to the IBM Entitled Registry entitled registry:
```
cpd-cli manage login-entitled-registry \
${IBM_ENTITLEMENT_KEY}
```

2) Log in to the private image registry :
```
cpd-cli manage login-private-registry \
${PRIVATE_REGISTRY_LOCATION} \
${PRIVATE_REGISTRY_PULL_USER} \
${PRIVATE_REGISTRY_PULL_PASSWORD}
```
3) Mirror image for RSI adm controller
```
cpd-cli manage copy-image \
--from=icr.io/cpopen/cpd/zen-rsi-adm-controller:4.8.5-x86_64 \
--to=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/zen-rsi-adm-controller:4.8.5-x86_64
```
4) Mirror image for Manta
```
cpd-cli manage copy-image \
--from=cp.icr.io/cp/cpd/manta-init-migrate-h2@sha256:0bb84e3f2ebd2219afa860e4bd3d3aa3a3c642b3b58685880df2cff121d43583 \
--to=${PRIVATE_REGISTRY_LOCATION}/cp/cpd/manta-init-migrate-h2
```

#### 3. The permissions required for the upgrade is ready
- Openshift cluster permissions
An Openshift cluster administrator can complete all of the installation tasks.<br>
However, if you want to enable users with fewer permissions to complete some of the installation tasks, follow this link https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=planning-installation-roles-personas and get the roles with required permission prepared.
- Cloud Pak for Data permissions
The Cloud Pak for Data administrator role or permissions is required for upgrading the service instances.

#### 4. A pre-upgrade health check is made to ensure the cluster's readiness for upgrade.
- The OpenShift cluster, persistent storage and Cloud Pak for Data platform and services are in healthy status.

## Table of Content

```
Part 1: Pre-upgrade
1.1 Collect information and review upgrade runbook
1.1.1 Review the upgrade runbook
1.1.2 Backup before upgrade
1.1.3 Uninstall all hotfixes and apply preventative measures
1.1.4 Uninstall the RSI patches and the cluster-scoped webhook
1.1.5 If use SAML SSO, export SSO configuration
1.2 Set up client workstation 
1.2.1 Prepare a client workstation
1.2.2 Update cpd_vars.sh for the upgrade to Version 4.8.5
1.2.3 Make olm-utils available
1.2.4 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image
1.2.5 Ensure the images were mirrored to the private container registry
1.2.6 Creating a profile for upgrading the service instances
1.3 Health check OCP & CPD

Part 2: Upgrade
2.1 Upgrade CPD to 4.8.5
2.1.1 Migrate to private topology
2.1.2 Preparing to upgrade the CPD instance
2.1.3 Upgrade foundation service and CPD platform to 4.8.5
2.1.4 Install the RSI patches
2.2 Upgrade CPD services
2.2.1 Upgrade IBM Knowledge Catalog service and apply hot fixes
2.2.2 Upgrade MANTA service
2.2.3 Upgrade Analytics Engine service
2.2.4 Upgrade Watson Studio, Watson Studio Runtimes, Watson Machine Learning and OpenScale
2.2.5 Upgrade Db2 Warehouse

Part 3: Post-upgrade
3.1 Configuring single sign-on
3.2 Validate CPD & CPD services
3.3 Removing the shared operators
3.4 CCS post-upgrade tasks
3.5 WKC post-upgrade tasks
3.6 Handling embedded Postgres license expiry for CPD 4.8.5
3.7 Summarize and close out the upgrade
```

## Part 1: Pre-upgrade
### 1.1 Collect information and review upgrade runbook

#### 1.1.1 Review the upgrade runbook

Review upgrade runbook

#### 1.1.2 Backup before upgrade
Note: Create a folder for 4.6.5 and maintain below created copies in that folder. <br>
Login to the OCP cluster for cpd-cli utility.

```
cpd-cli manage login-to-ocp --username=${OCP_USERNAME} --password=${OCP_PASSWORD} --server=${OCP_URL}
```

Capture data for the CPD 4.6.5 instance. No sensitive information is collected. Only the operational state of the Kubernetes artifacts is collected.The output of the command is stored in a file named collect-state.tar.gz in the cpd-cli-workspace/olm-utils-workspace/work directory.

```
cpd-cli manage collect-state \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE}
```

Make a copy of existing custom resources (Recommended)

```
oc project ${PROJECT_CPD_INSTANCE}

oc get ibmcpd ibmcpd-cr -o yaml > ibmcpd-cr.yaml

oc get zenservice lite-cr -o yaml > lite-cr.yaml

oc get CCS ccs-cr -o yaml > ccs-cr.yaml

oc get wkc wkc-cr -o yaml > wkc-cr.yaml

oc get analyticsengine analyticsengine-sample -o yaml > analyticsengine-cr.yaml

oc get DataStage datastage -o yaml > datastage-cr.yaml

 oc get route -o yaml > cpd_routes.yaml

```

Backup the routes.

```
oc get routes -o yaml > routes.yaml
```

Backup the RSI patches.
```
cpd-cli manage get-rsi-patch-info \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--all
```

Backup deployments and svc that need to be modified after upgrade.
```
oc get deploy asset-files-api -o yaml -n ${PROJECT_CPD_INSTANCE} > asset-files-api-deploy-465.yaml
oc get deploy catalog-api -o yaml -n ${PROJECT_CPD_INSTANCE} > catalog-api-deploy-465.yaml
oc get svc finley-public -o yaml -n ${PROJECT_CPD_INSTANCE} > finley-public-svc-465.yaml
```

Collect wkc db2u information.
```
oc exec -it c-db2oltp-wkc-db2u-0 bash
```

Collect the c-db2oltp-wkc-db2u-0 databases' buffer pool information.
```
for db in LINEAGE BGDB ILGDB WFDB; do echo "(*) dbname: $db"; db2top -d $db -b b -s 1 | awk -F';' '{print $2 ":" $14}'; echo "--------------------------"; done
```
Save the output to a file named wkc-db2u-dbs-bp.txt .


Collect the c-db2oltp-wkc-db2u-0 databases' log utilization information.
```
for db in LINEAGE BGDB ILGDB WFDB; do echo "(*) dbname: $db"; db2 connect to $db; db2 "select * from SYSIBMADM.LOG_UTILIZATION"; echo "--------------------------"; done
```
Save the output to a file named wkc-db2u-log-utilization.txt .

Collect the c-db2oltp-wkc-db2u-0 databases' snapshot information.
```
 for db in LINEAGE BGDB ILGDB WFDB; do echo "(*) dbname: $db"; db2 "get snapshot for database on $db" | grep -i log; echo "--------------------------"; done
```
Save the output to a file named wkc-db2u-log-snapshot.txt .

Collect the c-db2oltp-wkc-db2u-0 databases' log configuration information.
```
for db in LINEAGE BGDB ILGDB WFDB; do echo "(*) dbname: $db"; db2 "get db cfg for $db" | grep -i log; echo "--------------------------"; done
```
Save the output to a file named wkc-db2u-log-conf.txt .

#### 1.1.3 Uninstall all hotfixes and apply preventative measures 
Remove the hotfixes by removing the images from the CRs.
<br>

- 1.Uninstall WKC hot fixes.

<br>

1)Edit the wkc-cr with below command.
```
oc edit WKC wkc-cr
```
2)Remove the hot fix images from the WKC custom resource

```
 finley_public_image:
    name: finley-public@sha256
    tag: 9b5b907b054ca4bd355bc7180216d14fb40dc401ade120875e8ff6bc9d0f354a
    tag_metadata: 2.5.8-amd64

  metadata_discovery_image:
    name: metadata-discovery@sha256
    tag: 02a1923656678cd74f32329ff18bfe0f1b7e7716eae5a9cb25203fcfd23fcc35
    tag_metadata: 4.6.519

  wdp_profiling_image:
    name: wdp-profiling@sha256
    tag: ecc845503e45b4f8a0c83dce077d41c9a816cb9116d3aa411b000ec0eb916620
    tag_metadata: 4.6.5031-amd64

  wdp_profiling_ui_image:
    name: wdp-profiling-ui@sha256
    tag: 85e36bf943bc4ccd7cb2af0c524d5430ceabc90f2d5a5fb7e1696dbc251e5cc0
    tag_metadata: 4.6.1203

  wkc_bi_data_service_image:
    name: wkc-bi-data-service@sha256
    tag: 90837d26d108d3d086f71d6a9e36fbf7999caa4563404ee0e03d5735dfa2f3d3
    tag_metadata: 4.6.120

  wkc_mde_service_manager_image:
    name: wkc-mde-service-manager@sha256
    tag: 713684c36db568e0c9d5a3be40010b0f732fa73ede7177d9613bc040c53d6ab9
    tag_metadata: 1.2.55
  wkc_metadata_imports_ui_image:
    name: wkc-metadata-imports-ui@sha256
    tag: 53c8e2a0def2aa48c11bc702fc1ddd0dda089585f65597d0e64ec6cfba3a103e
    tag_metadata: 4.6.5511

  wkc_term_assignment_image:
    name: term-assignment-service@sha256
    tag: 80df5ba17fe08be48da4089d165bc29205009c79dde7f3ae3832db2edb7c54ce
    tag_metadata: 2.5.20-amd64
```

3)Add or update below entry for setting the number of wkc_bi_data_service replicas to be 3

```
wkc_bi_data_service_max_replicas: 3
wkc_bi_data_service_min_replicas: 3
```

4)Add or update the `kg_resources` section under spec in the wkc-cr

```
kg_resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 6
    memory: 8000Mi
```

5)Remove the `wdp-kg-ingestion-service` section from the wkc-cr spec

```
  wdp-kg-ingestion-service:
    limits:
      cpu: 6
      memory: 8000Mi
    requests:
      cpu: 250m
      memory: 512Mi
```

6)Remove the `ignoreForMaintenance: true` from the WKC custom resource

```
ignoreForMaintenance: true
```

7)Save and Exit. Wait untile the WKC Operator reconcilation completed and also the wkc-cr in 'Completed' status. 

```
oc get WKC wkc-cr -o yaml
```

- 2.Uninstall the AnalyticsEngine hot fixes.
<br>
1)Edit the analyticsengine-sample with below command.
  
```
oc edit AnalyticsEngine analyticsengine-sample
```

2)Remove the hot fix images from the AnalyticsEngine custom resource

```
 image_digests:
    spark-hb-control-plane: sha256:ef46de7224c6c37b2eadf2bfbbbaeef5be7b2e7e7c05d55c4f8b0eba1fb4e9e4
    spark-hb-jkg-v33: sha256:4b4eefb10d2a45ed1acab708a28f2c9d3619432f4417cfbfdc056f2ca3c085f7
```

3)Save and Exit. Wait untile the AnalyticsEngine Operator reconcilation completed and also the analyticsengine-sample in 'Completed' status. 

```
oc get AnalyticsEngine analyticsengine-sample -o yaml
```

- 3.Patch the CCS and uninstall the CCS hot fixes.
<br>
1)Increase the capacity of elasticsearch cluster for helping with indexing and searching

```
oc patch CCS ccs-cr --type merge --patch '{ "spec": { "elasticsearch_java_opts": "-Xmx8g -Xms8g", "elasticsearch_resources": { "requests": { "cpu": "200m", "ephemeral-storage": "10Mi", "memory": "1Gi" }, "limits": { "cpu": "4", "ephemeral-storage": "1Gi", "memory": "16Gi" } } } }'
```
<br>
2)Edit the CCS cr with below command.
  
```
oc edit CCS ccs-cr
```

3)Remove the hot fix images from the CCS custom resource

```
  asset_files_api_image:
    name: asset-files-api@sha256
    tag: a1525c29bebed6e9a982f3a06b3190654df7cf6028438f58c96d0c8f69e674c1
    tag_metadata: 4.6.5.4.155-amd64

  catalog_api_aux_image:
    name: catalog-api-aux_master@sha256
    tag: e221df32209340617763897d6acbdcbae6a29d75bd0fd2af65ba6448c430534d
    tag_metadata: 2.0.0-20240311173712-babe16ea94

  catalog_api_image:
    name: catalog_master@sha256
    tag: a5f2b44fbe532b9fecd4f67b00c937becde897e1030b7aa48087cbc2c8505707
    tag_metadata: 2.0.0-20240311173712-babe16ea94

  jobs_ui_image:
    name: jobs-ui@sha256
    tag: 7758afa382ce3302fb2d8fb020cfe7baab5d960da3896ef4eb4bb2187cb477e3
    tag_metadata: 4.6.5.2.167

  portal_catalog_image:
    name: portal-catalog@sha256
    tag: 4646053d470dbb7edc90069f1d7e0b1d26da76edd7325d22af50535a61e42fed
    tag_metadata: 0.4.2817

  portal_projects_image:
    name: portal-projects@sha256
    tag: d3722fb9a7e4a97f6f6de7d2b92837475e62cd064aa6d7590342e05620b16a6a
    tag_metadata: 4.6.5.4.2504-amd64

  wdp_connect_connection_image:
    name: wdp-connect-connection@sha256
    tag: 3d5fadf3ec1645dae10136226d37542a9d087782663344a1f78e0ee3af7b5aa6
    tag_metadata: 6.3.325

  wdp_connect_connector_image:
    name: wdp-connect-connector@sha256
    tag: 1b7ecb102c8461b1b9b0df9a377695b71164b00ab72391ddf4b063bd45da670c
    tag_metadata: 6.3.325

  wdp_connect_flight_image:
    name: wdp-connect-flight@sha256
    tag: a1558a88258719da7414e345550210ab6e013c45af54c22bf01d37851f94dc9f
    tag_metadata: 6.3.324

  wkc_search_image:
    name: wkc-search_master@sha256
    tag: 08105e65f1b0091499366d8f15b6a6d045bc1319bbae463619737172afed1dc1
    tag_metadata: 4.6.194
```
4)Apply preventative measures for Elastic Search pvc customization problem
<br>
This step is for applying the preventative measures for Elastic Search problem. Applying the preventative measures in this timing can also help to minimize the number of CCS operator reconcilations.
<br>
List Elasticsearch PVC sizes, and make sure to preserve the type, and the size of the largest one (PVC names may be different depending on client environment):
<br>

```
oc get pvc | grep elastic | grep RWO

hptv-prodcloudpak              elasticsearch-master-elasticsearch-master-0        Bound    pvc-b7e7db35-2deb-4ebb-949e-8f01abc6649f   1126Gi     RWO            ocs-storagecluster-ceph-rbd   329d
hptv-prodcloudpak              elasticsearch-master-elasticsearch-master-1        Bound    pvc-28652bea-503f-4034-a29e-00e7bb5cf63e   1126Gi     RWO            ocs-storagecluster-ceph-rbd   329d
hptv-prodcloudpak              elasticsearch-master-elasticsearch-master-2        Bound    pvc-03258821-8839-4600-8529-853125f99195   1126Gi     RWO            ocs-storagecluster-ceph-rbd   329d

```

In the above example, block storage `ocs-storagecluster-ceph-rbd` is the storage type, and `1126Gi` is the largest size. 
<br>
**Note** if PVCs are of different sizes, we want to make sure to take the biggest one. 
<br>

In CCS CR make sure to set the following properties, with above values used as example:

```
elasticsearch_persistence_size: "1126Gi"
elasticsearch_storage_class_name: "ocs-storagecluster-ceph-rbd"
```

This will make sure that the Opensearch operator will properly reconcile, - as provided values will match the state of the cluster. 

5)Apply preventative measures for Elastic Search backup time out problem
<br>

The time out issue that may occur during the backup operation. This problem can be avoided by setting the following property in CCS CR:

```
elasticsearch_cpdbr_timeout_seconds: 100000
```

6)Remove the `ignoreForMaintenance: true` from the CCS custom resource

7)Save and Exit. Wait untile the CCS Operator reconcilation completed and also the ccs-cr in 'Completed' status. 

```
oc get CCS ccs-cr -o yaml
```

8)Wait untile the WKC Operator reconcilation completed and also the wkc-cr in 'Completed' status. 

```
oc get WKC wkc-cr -o yaml
```

- 4.Remove the `ignoreForMaintenance: true` from the ZenService custom resource.

```
oc edit ZenService lite-cr
```

Save and Exit. Wait untile the ZenService Operator reconcilation completed and also the lite-cr in 'Completed' status. 
<br>

- 5.Apply preventative measures for potential time-consuming CCS reconcilation caused by large number of cronjobs.
<br>

Create a directory for the backup of cronjobs.

```
mkdir cronjob_bak
cd cronjob_bak
```

**Important:**
<br>

Backup of all cronjob

```
for cj in $(oc get cronjob -l runtimeAssembly --no-headers | grep "<none>" | awk '{print $1}'); do oc get cronjob $cj -oyaml >  $cj.yaml;done
```

Deleting label from all cronjobs

```
for cj in $(oc get cronjob -l runtimeAssembly --no-headers | grep "<none>" | awk '{print $1}'); do oc label cronjob $cj created-by- 2>/dev/null; done

```

Return to the parent directory.

```
cd ..
```

#### 1.1.4 Uninstall the RSI patches and the cluster-scoped webhook
1.Run the cpd-cli manage login-to-ocp command to log in to the cluster as a user with sufficient permissions.
```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```
2.Delete all RSI patches.
<br>
- Delete asset-files-api-annotation-selinux.
<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=asset-files-api-annotation-selinux \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=asset-files-api-annotation-selinux
```

- Delete asset-files-api-pod-spec-selinux.
<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=asset-files-api-pod-spec-selinux \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=asset-files-api-pod-spec-selinux
```

- Delete create-dap-directories-annotation-selinux.

<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=create-dap-directories-annotation-selinux \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=create-dap-directories-annotation-selinux
```

- Delete create-dap-directories-pod-spec-selinux.

<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=create-dap-directories-pod-spec-selinux \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=create-dap-directories-pod-spec-selinux
```

- Delete event-logger-api-annotation-selinux.

<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=event-logger-api-annotation-selinux \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=event-logger-api-annotation-selinux
```

- Delete event-logger-api-pod-spec-selinux.

<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=event-logger-api-pod-spec-selinux \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=event-logger-api-pod-spec-selinux
```

- Delete finley-public-service-patch.

<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=finley-public-service-patch \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=finley-public-service-patch
```

- Delete iae-nginx-ephemeral-patch.

<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=iae-nginx-ephemeral-patch \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=iae-nginx-ephemeral-patch
```

- Delete mde-service-manager-patch.

<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=mde-service-manager-patch \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=mde-service-manager-patch
```

- Delete mde-service-manager-patch-2.
  
<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=mde-service-manager-patch-2 \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=mde-service-manager-patch-2
```

- Delete mde-service-manager-env-patch-publish-batch-size.

<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=mde-service-manager-env-patch-publish-batch-size \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=mde-service-manager-env-patch-publish-batch-size
```

- Delete rsi-env-term-assignment-4.6.5-patch-2-april2024.

<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=env-term-assignment-4.6.5-patch-2-april2024 \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=env-term-assignment-4.6.5-patch-2-april2024
```

- Activate term-assignment-env-patch-1-march2024.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=term-assignment-env-patch-1-march2024 \
--state=inactive
```

- Delete spark-runtimes-annotation-selinux.

<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=spark-runtimes-annotation-selinux \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=spark-runtimes-annotation-selinux
```

- Delete spark-runtimes-pod-spec-selinux.

<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=spark-runtimes-pod-spec-selinux \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=spark-runtimes-pod-spec-selinux
```

- Delete finley-public-env-patch-1-may2024.
  
<br>
Inactivate:

```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=finley-public-env-patch-1-may2024 \
--state=inactive
```

Delete:

```
cpd-cli manage delete-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=finley-public-env-patch-1-may2024
```

4.Check the RSI patches status again:
<br>
Get all RSI patches' status
```
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --all
```
Check the RSI patches' status

```
cat cpd-cli-workspace/olm-utils-workspace/work/get_rsi_patch_info.log
```

Check whether all the RSI patches' zenextensions are removed.

```
oc get zenextension -n ${PROJECT_CPD_INSTANCE} | grep -i rsi
```

5.Disable the RSI feature in the project
If IBM Cloud Pak foundational services is installed in ibm-common-services
```
cpd-cli manage disable-rsi \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE}
```
6.Uninstall the webhook
```
cpd-cli manage uninstall-rsi \
--cs_ns=${PROJECT_CPFS_OPS} \
--rsi_image=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/zen-rsi-adm-controller:4.6.5-x86_64
```

#### 1.1.5 If use SAML SSO, export SSO configuration

If you use SAML SSO, export your SSO configuration. You will need to reapply your SAML SSO configuration after you upgrade to Version 4.8. Skip this step if you use the IBM Cloud Pak foundational services Identity Management Service

```
oc cp -n=${PROJECT_CPD_INSTANCE} \
$(oc get pods -l component=usermgmt -n ${PROJECT_CPD_INSTANCE} \
-o jsonpath='{.items[0].metadata.name}'):/user-home/_global_/config/saml ./samlConfig
```

### 1.2 Set up client workstation

#### 1.2.1 Prepare a client workstation

1. Prepare a RHEL 8 machine with internet

Create a directory for the cpd-cli utility.
```
export CPD485_WORKSPACE=/ibm/cpd/485
mkdir -p ${CPD485_WORKSPACE}
cd ${CPD485_WORKSPACE}
```

Download the cpd-cli for 4.8.5.

```
wget https://github.com/IBM/cpd-cli/releases/download/v13.1.5/cpd-cli-linux-EE-13.1.5.tgz
```

2. Install tools.

```
yum install openssl httpd-tools podman skopeo wget -y
```

```
tar xvf cpd-cli-linux-EE-13.1.5.tgz
mv cpd-cli-linux-EE-13.1.5-176/* .
rm -rf cpd-cli-linux-EE-13.1.5-176
```

3. Copy the cpd_vars.sh file used by the CPD 4.8.2 to the folder ${CPD485_WORKSPACE}.

```
cd ${CPD485_WORKSPACE}
cp <the file path of the cpd_vars.sh file used by the CPD 4.8.2 > cpd_vars_485.sh
```
4. Make cpd-cli executable anywhere
```
vi cpd_vars_485.sh
```

Add below two lines into the head of cpd_vars_485.sh

```
export CPD485_WORKSPACE=/ibm/cpd/485
export PATH=${CPD485_WORKSPACE}:$PATH
```

Update the CPD_CLI_MANAGE_WORKSPACE variable

```
export CPD_CLI_MANAGE_WORKSPACE=${CPD485_WORKSPACE}
```

Run this command to apply cpd_vars_485.sh

```
source cpd_vars_485.sh
```

Check out with this commands

```
cpd-cli version
```

Output like this

```
cpd-cli
	Version: 13.1.5
	Build Date: 
	Build Number: nn
	CPD Release Version: 4.8.5
```
#### 1.2.2 Update cpd_vars.sh for the upgrade to Version 4.8.5

```
vi cpd_vars_485.sh
```

1.Locate the VERSION entry and update the environment variable for VERSION. 

```
export VERSION=4.8.5
```
2.Locate the Projects section of the script and add the following environment variables. 
<br>
**Note:** 
<br>When adding the following environment variables, The value of PROJECT_CPD_INST_OPERANDS is the same as that of PROJECT_CPD_INSTANCE.
```
export PROJECT_CERT_MANAGER=ibm-cert-manager
export PROJECT_LICENSE_SERVICE=ibm-licensing
export PROJECT_CS_CONTROL=ibm-licensing
export PROJECT_CPD_INST_OPERATORS=cpd-operators
export PROJECT_CPD_INST_OPERANDS=hptv-prodcloudpak
```
3.Remove the PROJECT_CATSRC entry from the Projects section of the script.

4.Locate the COMPONENTS entry and upate the COMPONENTS entry.
If the advanced metadata import feature in IBM® Knowledge Catalog is used, add the mantaflow component to the COMPONENTS variable.
```
export COMPONENTS=ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,ws,ws_runtimes,wml,wkc,analyticsengine,mantaflow,openscale,db2wh
```

Save the changes. <br>

Confirm that the script does not contain any errors. For example, if you named the script cpd_vars.sh, run:
```
bash ./cpd_vars.sh
```

Run this command to apply cpd_vars_485.sh
```
source cpd_vars_485.sh
```
5.Locate the Cluster section of the script and add the following environment variables.
```
export SERVER_ARGUMENTS="--server=${OCP_URL}"
export LOGIN_ARGUMENTS="--username=${OCP_USERNAME} --password=${OCP_PASSWORD}"
export CPDM_OC_LOGIN="cpd-cli manage login-to-ocp ${SERVER_ARGUMENTS} ${LOGIN_ARGUMENTS}"
export OC_LOGIN="oc login ${OCP_URL} ${LOGIN_ARGUMENTS}"
```

#### 1.2.3 Make olm-utils available
**Note:** If the bastion node is internet connected, then you can ignore below steps in this section.

```
podman pull icr.io/cpopen/cpd/olm-utils-v2:latest --tls-verify=false

podman login ${PRIVATE_REGISTRY_LOCATION} -u ${PRIVATE_REGISTRY_PULL_USER} -p ${PRIVATE_REGISTRY_PULL_PASSWORD}

podman tag icr.io/cpopen/cpd/olm-utils-v2:latest ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v2:latest

podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v2:latest --remove-signatures 

export OLM_UTILS_IMAGE=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v2:latest
export OLM_UTILS_LAUNCH_ARGS=" --network=host"

```
For details please refer to 4.8 doc (https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=46-updating-client-workstations)

#### 1.2.4 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image
```
podman stop olm-utils-play-v2
cpd-cli manage restart-container
```
**Note:**
<br>Check the olm-utils-v2 image ID and ensure it's the latest one.
```
podman images | grep olm-utils-v2
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

Click this link and follow these steps for getting it done. https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=cli-creating-cpd-profile#taskcpd-profile-mgmt__steps__1


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

4. Check if there are any customization of the PVC size.

Check the pvc size of the zen-metastoredb. Default size is 10Gi.
```
oc get pvc | grep zen-metastore
```

Check the pvc size of the OpenScale payload pvc. Default size is 4Gi.
```
oc get pvc | grep aiopenscale-ibm-aios-payload-pvc
```

If there are any customization of the PVC, we'll need to evaluate and make sure this is addressed before the upgrade.

5. Check for wkc-search pod.

```
oc get pods -n ${PROJECT_CPD_INSTANCE} | grep wkc-search
```

Remove the wkc-search pods which is in completed state ( if there are any ). We should have wkc-search pod only in Running state.

## Part 2: Upgrade
### 2.1 Upgrade CPD to 4.8.5

#### 2.1.1 Migrate to private topology
1.Create new projects
```
${OC_LOGIN}
oc new-project ${PROJECT_CS_CONTROL}             # This is for ibm-licensing operator and instance
oc new-project ${PROJECT_CERT_MANAGER}           # This is for ibm-cert-manager operator and instance
oc new-project ${PROJECT_LICENSE_SERVICE}        # This is for the License Service
```
2.Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```
3.Move the Certificate manager and License Service from the shared operators project to the cs-control project.
```
cpd-cli manage detach-cpd-instance \
--cpfs_operator_ns=${PROJECT_CPFS_OPS} \
--control_ns=${PROJECT_CS_CONTROL} \
--specialized_operator_ns=${PROJECT_CPD_OPS}
```
**Note**:
<br>WARNING: This step will ask you to validated that your WKC legacy data can be migrated If you want to continue with the migration, please type: 'I have validated that I can migrate my metadata and I want to continue'
<br>Monitor the install plan and approved them as needed.
<br><br>Wait for the detach-cpd-instance command ran successfully before proceeding to the next step:
<br>Confirm that the Certificate manager and License Service pods in the cs-control project are Running :
```
oc get pods --namespace=${PROJECT_CS_CONTROL}
```
4.Upgrade the Certificate manager and License Service
```
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--migrate_from_cs_ns=${PROJECT_CPFS_OPS} \
--cert_manager_ns=${PROJECT_CERT_MANAGER} \
--licensing_ns=${PROJECT_CS_CONTROL}
```
The Certificate manager will be moved to the {PROJECT_CERT_MANAGER} project. The License Service will remain in the cs-control {PROJECT_CS_CONTROL} project.
<br>Confirm that the Certificate manager pods in the ${PROJECT_CERT_MANAGER} project are Running:
```
oc get pod -n ${PROJECT_CERT_MANAGER}
```
Confirm that the License Service pods in the ${PROJECT_CS_CONTROL} project are Running:
```
oc get pods --namespace=${PROJECT_CS_CONTROL}
```

#### 2.1.2 Preparing to upgrade an CPD instance
1.Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```

2.Detache CPD instance from the shared operators
```
cpd-cli manage detach-cpd-instance \
--cpfs_operator_ns=${PROJECT_CPFS_OPS} \
--specialized_operator_ns=${PROJECT_CPD_OPS} \
--control_ns=${PROJECT_CS_CONTROL} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```
<br>WARNING: This step will ask you to validated that your WKC legacy data can be migrated If you want to continue with the migration, please type: 'I have validated that I can migrate my metadata and I want to continue'
<br>
Confirm ${PROJECT_CPD_INST_OPERANDS} has been isolated from the previous nss
```
oc get cm -n $PROJECT_CPFS_OPS namespace-scope -o yaml
```

Result example:
```
Name:         namespace-scope
Namespace:    ibm-common-services
Labels:       <none>
Annotations:  <none>

Data
====
namespaces:
----
ibm-common-services #<- original $PROJECT_CPD_INSTANCE (cpd-instance) should NOT appear here
...
```

3.Manually creating the operators project
Create the operators project for the instance:
```
oc new-project ${PROJECT_CPD_INST_OPERATORS}
```

4.Apply the required permissions to the projects
```
cpd-cli manage authorize-instance-topology \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

#### 2.1.3 Upgrade foundation service and CPD platform to 4.8.5

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster.
```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```
2.Upgrade IBM Cloud Pak foundational services and create the required ConfigMap.
```
cpd-cli manage setup-instance-topology \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--block_storage_class=${STG_CLASS_BLOCK}
```
 
Confirm common-service, namespace-scope, opencloud and odlm operator migrated to ${PROJECT_CPD_INST_OPERATORS} namespace
```
oc get pod -n ${PROJECT_CPD_INST_OPERATORS}
```

3.Upgrade the operators in the operators project for CPD instance
```
cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--upgrade=true
```

In another terminal, keep running below command and monitoring "InstallPlan" to find which one need manual approval.
```
watch "oc get ip -n ibm-cpd-operators -o=jsonpath='{.items[?(@.spec.approved==false)].metadata.name}'"
```
Approve the upgrade request and run below command as soon as we find it.
```
oc patch installplan $(oc get ip -n ibm-cpd-operators -o=jsonpath='{.items[?(@.spec.approved==false)].metadata.name}') -n ibm-cpd-operators --type merge --patch '{"spec":{"approved":true}}'
```

Confirm that the operator pods are Running or Copmleted:
```
oc get pods --namespace=${PROJECT_CPD_INST_OPERATORS}
```

**WARNING:**
<br>This step will ask you to validated that your WKC legacy data can be migrated If you want to continue with the migration, please type: 'I have validated that I can migrate my metadata and I want to continue'
<br><br>
Check the version for both CSV and Subscription and ensure the CPD Operators have been upgraded successfully.
```
oc get csv,sub -n ${PROJECT_CPD_INST_OPERATORS}
```
Operator and operand versions: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=planning-operator-operand-versions

4. Upgrade the operands in the operands project for CPD instance

```
cpd-cli manage apply-cr \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=cpd_platform \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true \
--upgrade=true
```
Confirm that the status of the operands is Completed:
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

5.Clean up any failed operand requests in the operands project:
Get the list of operand requests with the format <component>-requests-<component>:
```
oc get operandrequest --namespace=${PROJECT_CPD_INST_OPERANDS} | grep requests
```
Delete each operand request in the Failed state: Replace with the name of operand request to delete.
```
oc delete operandrequest <operand-request-name> \
--namespace=${PROJECT_CPD_INST_OPERANDS}
```
Remove the instance project from the sharewith list in the ibm-cpp-config SecretShare in the shared IBM Cloud Pak foundational services operators project:

<br>Confirm the name of the instance project:
```
echo $PROJECT_CPD_INST_OPERANDS
```

Check whether the instance project is listed in the sharewith list in the ibm-cpp-config SecretShare:
```
oc get secretshare ibm-cpp-config \
--namespace=${PROJECT_CPFS_OPS} \
-o yaml
```
The command returns output with the following format:
```
apiVersion: ibmcpcs.ibm.com/v1
kind: SecretShare
metadata:
  name: ibm-cpp-config
  namespace: ibm-common-services
spec:
  configmapshares:
  - configmapname: ibm-cpp-config
    sharewith:
    - namespace: cpd-instance-x
    - namespace: ibm-common-services
    - namespace: cpd-operators
    - namespace: cpd-instance-y
```
If the instance project is in the list, proceed to the next step. If the instance is not in the list, no further action is required.
<br>
Open the ibm-cpp-config SecretShare in the editor:
```
oc edit secretshare ibm-cpp-config \
--namespace=${PROJECT_CPFS_OPS}
```
Remove the entry for the instance project from the sharewith list and save your changes to the SecretShare.

#### 2.1.4 Install the RSI patches

1).Log the cpd-cli in to the Red Hat OpenShift Container Platform cluster.
```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```
2).Install the RSI webhook.
```
cpd-cli manage install-rsi \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
--rsi_image=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/zen-rsi-adm-controller:${VERSION}-x86_64
```
3).Enable RSI for the instance
```
cpd-cli manage enable-rsi \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```
4).Install the RSI patches
<br>
Create a json patch file named `annotation-spec.json` under `cpd-cli-workspace/olm-utils-workspace/work/rsi` with the following content:

```
[{"op":"add","path":"/metadata/annotations/io.kubernetes.cri-o.TrySkipVolumeSELinuxLabel","value":"true"}]
```

Create a json patch file named `specpatch.json` under `cpd-cli-workspace/olm-utils-workspace/work/rsi` with the following content:

```
[{"op":"add","path":"/spec/runtimeClassName","value":"selinux"}]
```

<br>
- Reinstall the asset-files-api-annotation-selinux.

```
cpd-cli manage create-rsi-patch --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --patch_type=rsi_pod_annotation --patch_name=asset-files-api-annotation-selinux --description="This is annotation patch is for selinux relabeling disabling on CSI based storages for asset-files-api" --include_labels=app:asset-files-api --state=active --spec_format=json --patch_spec=/tmp/work/rsi/annotation-spec.json
```
- Reinstall asset-files-api-pod-spec-selinux.
  
```
cpd-cli manage create-rsi-patch --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --patch_type=rsi_pod_spec --patch_name=asset-files-api-pod-spec-selinux --description="This is spec patch is for selinux relabeling disabling on CSI based storages for asset-files-api" --include_labels=app:asset-files-api --state=active --spec_format=json --patch_spec=/tmp/work/rsi/specpatch.json
```

- Reinstall create-dap-directories-annotation-selinux.
  
```
cpd-cli manage create-rsi-patch --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --patch_type=rsi_pod_annotation --patch_name=create-dap-directories-annotation-selinux --description="This is annotation patch is for selinux relabeling disabling on CSI based storages for create-dap-directories-job" --include_labels=app:create-dap-directories --state=active --spec_format=json --patch_spec=/tmp/work/rsi/annotation-spec.json
```

- Reinstall create-dap-directories-pod-spec-selinux.

```
cpd-cli manage create-rsi-patch --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --patch_type=rsi_pod_spec --patch_name=create-dap-directories-pod-spec-selinux --description="This is spec patch is for selinux relabeling disabling on CSI based storages for create-dap-directories job" --include_labels=app:create-dap-directories --state=active --spec_format=json --patch_spec=/tmp/work/rsi/specpatch.json
```

- Reinstall event-logger-api-annotation-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--patch_type=rsi_pod_annotation \
--patch_name=event-logger-api-annotation-selinux \
--description="This is annotation patch is for selinux relabeling disabling on CSI based storages for event-logger-api" \
--include_labels=app:event-logger-api \
--state=active \
--spec_format=json \
--patch_spec=/tmp/work/rsi/annotation-spec.json
```

- Reinstall event-logger-api-pod-spec-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--patch_type=rsi_pod_spec \
--patch_name=event-logger-api-pod-spec-selinux \
--description="This is spec patch is for selinux relabeling disabling on CSI based storages for event-logger-api" --include_labels=app:event-logger-api \
--state=active \
--spec_format=json \
--patch_spec=/tmp/work/rsi/specpatch.json
```

- Reinstall spark-runtimes-annotation-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--patch_type=rsi_pod_annotation \
--patch_name=spark-runtimes-annotation-selinux \
--description="This is annotation patch is for selinux relabeling disabling on CSI based storages for spark worker and master pods" \
--include_labels=spark/exclude-from-backup:true \
--state=active --spec_format=json --patch_spec=/tmp/work/rsi/annotation-spec.json
```

- Reinstall spark-runtimes-pod-spec-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--patch_type=rsi_pod_spec \
--patch_name=spark-runtimes-pod-spec-selinux \
--description="This is spec patch is for selinux relabeling disabling on CSI based storages for spark worker and master pods" \
--include_labels=spark/exclude-from-backup:true \
--state=active \
--spec_format=json \
--patch_spec=/tmp/work/rsi/specpatch.json
```

- Install the RSI patch for the couchdb pvc mounting issue.
<br>

Create a patch file named `couch-fsGroupChangePolicy.json` under `cpd-cli-workspace/olm-utils-workspace/work/rsi` with the following content:

```
[{"op":"add","path":"/spec/securityContext/fsGroupChangePolicy","value":"OnRootMismatch"}]
```

Apply the RSI patch about fsGroupChangePolicy for CouchDB.

```
cpd-cli manage create-rsi-patch --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --patch_type=rsi_pod_spec \
  --patch_name=couchdb-pod-spec-fsgroupchangepolicy \
  --description="This is spec patch for couchdb fsGroupChangePolicy" \
  --include_labels=app:couchdb \
  --state=active \
  --spec_format=json \
  --patch_spec=/tmp/work/rsi/couch-fsGroupChangePolicy.json
```

Apply the RSI patch about annotation selinux for CouchDB.

```
cpd-cli manage create-rsi-patch --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --patch_type=rsi_pod_annotation \
  --patch_name=couchdb-pod-annotation-selinux \
  --description="This annotation patch is for selinux relabeling disabling on CSI based storages for couchdb" \
  --include_labels=app:couchdb \
  --state=active \
  --spec_format=json \
  --patch_spec=/tmp/work/rsi/annotation-spec.json
```

Apply the RSI patch about spec selinux for CouchDB.

```
cpd-cli manage create-rsi-patch --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --patch_type=rsi_pod_spec \
  --patch_name=couchdb-pod-spec-selinux \
  --description="This spec patch is for selinux relabeling disabling on CSI based storages for couchdb" \
  --include_labels=app:couchdb \
  --state=active \
  --spec_format=json \
  --patch_spec=/tmp/work/rsi/specpatch.json
```

- Install the patch for finley-public.

Create a patch file named `patch-rsi-env-finley-public-4.8.5-patch-1-may2024.json` under `cpd-cli-workspace/olm-utils-workspace/work/rsi` with the following content:

```
[
  {
    "name": "ENABLE_SLIDING_WINDOW_FOR_WORD_BREAK_PRE_PROCESS",
    "value": "true"
  },
  {
    "name": "FINAL_PREDICTION_SIZE_LIMIT",
    "value": "1000"
  }
]
```
Apply the RSI patch about finley-public.

```
cpd-cli manage create-rsi-patch --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--patch_type=rsi_pod_env_var \
--patch_name=finley-public-env-patch-1-may2024 \
--description=Finley_public_patch_1_May_2024 \
--include_labels=app:finley-public \
--state=active \
--spec_format=set-env \
--patch_spec=/tmp/work/rsi/patch-rsi-env-finley-public-4.8.5-patch-1-may2024.json
```

5).Check the RSI patches status again:
```
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --all

cat cpd-cli-workspace/olm-utils-workspace/work/get_rsi_patch_info.log
```

### 2.2 Upgrade CPD services to 4.8.5
#### 2.2.1 Upgrade IBM Knowledge Catalog service and apply hot fixes
Check if the IBM Knowledge Catalog service was installed with the custom install options. 
##### 1. For custom installation, check the previous install-options.yaml or wkc-cr yaml, make sure to keep original custom settings
```
vim cpd-cli-workspace/olm-utils-workspace/work/install-options.yml

################################################################################
# IBM Knowledge Catalog parameters
################################################################################
custom_spec:
  wkc:
    enableKnowledgeGraph: True
    enableDataQuality: False
```

##### 2.Apply the timeout settings in CCS Operator for avoiding the elstic search timeout issue

Considering the large volume of data in the elsticsearch pvc, the backup job may take longer than the default `100` status checks that the CCS operator alots it by default. Unfortunately this is not a setting that is exposed directly as a CCS parameter and a manual change has to be carried out before upgrade to ensure that enough time is give to the backup process:

1). Copy `roles/wkc-base/tasks/elasticsearch_migration.yml` out of the CCS operator pod for further modification. 

```bash
# get the name of the operator pod in the operator namespace
oc get pod -n ${PROJECT_CPD_INST_OPERATORS} | grep ibm-cpd-ccs-operator | grep -v catalog | grep Running

ibm-cpd-ccs-operator-77568d6655-qvbj8                             1/1     Running                  0              5d4h

# use the name of the operator pod to copy out the elasticsearch_migration.yml to a temporary location
oc cp -n ${PROJECT_CPD_INST_OPERATORS} ibm-cpd-ccs-operator-77568d6655-qvbj8:/opt/ansible/8.5.0/roles/wkc-base/tasks/elasticsearch_migration.yml elasticsearch_migration.yml
```

2). Create a backup of the file.
```
cp elasticsearch_migration.yml elasticsearch_migration_bak.yml
```

4. Modify line `238` replacing value `100` by `10000`. This should give sufficient amount of time for event the largest backup sizes. 

5. Replace the version insided of the CCS operator pod with the updated one - using the following command:
```bash
oc cp -n ${PROJECT_CPD_INST_OPERATORS} elasticsearch_migration.yml ibm-cpd-ccs-operator-77568d6655-qvbj8:/opt/ansible/8.5.0/roles/wkc-base/tasks/elasticsearch_migration.yml
```


##### 3.Upgrade WKC with custom installation

```
export COMPONENTS=wkc
```

Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```

Run custom upgrade with installation options.
```
cpd-cli manage apply-cr \
--components=wkc \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--param-file=/tmp/work/install-options.yml \
--license_acceptance=true \
--upgrade=true
```

##### 4.Validate the upgrade
```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

##### 5.Apply the hotfixes
June Zenservice patch command (including external vault connection for reducing the number of operator reconcilations): <br>

```
oc patch ZenService lite-cr -n ${PROJECT_CPD_INSTANCE} --type merge -p '{"spec":{"image_digests":{"icp4data_nginx_repo":"sha256:2ab2c0cfecdf46b072c9b3ec20de80a0c74767321c96409f3825a1f4e7efb788","icpd_requisite":"sha256:5a7082482c1bcf0b23390d36b0800cc04cfaa32acf7e16413f92304c36a51f02","privatecloud_usermgmt":"sha256:e7b0dda15fa3905e4f242b66b18bc9cf2d27ea46e267e5a8d6a3d7da011bddb1","zen_audit":"sha256:ccf61039298186555fd18f568e715ca9e12f07805f42eb39008f851500c0f024","zen_core":"sha256:67f4d92a6e1f39675856fe3b46b36b34e9f0ae25679f75a1628c9d7d44790bad","zen_core_api":"sha256:b3ba3250a228d5f1ba3ea93ccf8b0f018e557f0f4828ed773b57075b842c30e9","zen_iam_config":"sha256:5abf2bf3f29ca28c72c64ab23ee981e8ad122c0de94ca7702980e1d40841d91a","zen_minio":"sha256:f66e6c17d1ed9d90a90e9a1280a18aacb9012bbdb604c5230d97db4cffcb4b48","zen_utils":"sha256:6d906104a8bd8b15f3ebcb2c3ae6a5f93c8d88ce6cfcae4b3eed6657562dc9f3","zen_watchdog":"sha256:4f73b382687bd4de6754292670f6281a7944b6b0903396ed78f1de2da54bc8c0"},"vault_bridge_tls_tolerate_private_ca": true}}'
```

April & May combined CCS patch command (including the jdbc driver uploading for reducing the number of operator reconcilations): <br>

```
oc patch ccs ccs-cr -n ${PROJECT_CPD_INSTANCE} --type=merge -p '{"spec":{"image_digests":{"portal_projects_image":"sha256:93c38bf9870a5e8f9399b1e90e09f32e5f556d5f6e03b4a447a400eddb08dc4e","asset_files_api_image":"sha256:bfa820ffebcf55b87f7827037daee7ec074d0435139e57acbb494df19aee0e98","catalog_api_image":"sha256:4ee6645dd5d9160150f3ad21298e85b28bfe45f6bfff3298861552ccf0897903","wkc_search_image":"sha256:3e95e932b2d2a186cab56b5073e2f9d1b70f1ac24a6ff48c1ae322e8727bdcb3","portal_catalog_image":"sha256:33e51a0c7eb16ac4b5dbbcd57b2ebe62313435bab2c0c789a1801a1c2c00c77d"},"wdp_connect_connection_jdbc_drivers_repository_mode": "enabled","asset_files_call_socket_timeout_ms": 60000}}'
```

April & May combined WKC patch command (including legacyCleanup for reducing the number of operator reconcilations):<br>

```
oc patch wkc wkc-cr -n ${PROJECT_CPD_INSTANCE} --type=merge -p '{"spec":{"image_digests":{"wkc_metadata_imports_ui_image":"sha256:a1997d9a9cde9ecc9f16eb02099a272d7ba2e8d88cb05a9f52f32533e4d633ef","wdp_profiling_image":"sha256:d5491cc8c8c8bd45f2605f299ecc96b9461cd5017c7861f22735e3a4d0073abd","wkc_mde_service_manager_image":"sha256:35e6f6ede3383df5c0b2a3d27c146cc121bed32d26ab7fa8870c4aa4fbc6e993","finley_public_image":"sha256:e89b59e16c4c10fce5ae07774686d349d17b2d21bf9263c50b49a7a290499c6d"},"wkc_term_assignment_rest_retry_config":"cams\\\\.write:400|3|2;cams\\\\.attachment\\\\.markcomplete:400|3|2,404|3|2;finley\\\\.predict:500|18|10,502|18|10,503|18|10,504|1|10,507|18|10;.*:408|3|2,409|3|2,425|3|2,429|3|2,500|6|10,502|6|10,503|6|10,504|6|10,507|6|10,-1|3|2","wkc_term_assignment_finley_page_size_reduction_divisor":"5","finley_public_gunicorn_worker_timeout":"65","wdp_profiling_flight_enabled":"false","legacyCleanup":true}}'

```

#### 2.2.2 Upgrade MANTA service
```
export COMPONENTS=mantaflow
```

Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```

Patch the mantaflow custom resource (CR) by running the following command:

```
oc patch mantaflow mantaflow-wkc --type merge -p '{ "spec": { "migrations": { "h2-format-3": "true" }}}'
```

Recreate the deployments by running:
```
oc delete deploy manta-admin-gui manta-configuration-service manta-dataflow
```

Run the command for upgrading MANTA service.

```
cpd-cli manage apply-cr \
--components=mantaflow \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true \
--upgrade=true
```

Validating the upgrade.
```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=${COMPONENTS}
```

#### 2.2.3 Upgrade Analytics Engine service
##### 2.2.3.1 Upgrade the service

Check the Analytics Engine service version and status. 
```
export COMPONENTS=analyticsengine

cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=${COMPONENTS}
```

The Analytics Engine serive should have been upgraded as part of the WKC service upgrade. If the Analytics Engine service version is **not 4.8.5**, then run below commands for the upgrade. <br>

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

##### 2.2.3.2 Upgrade the service instances

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
#### 2.2.4 Upgrade Watson Studio, Watson Studio Runtimes and Watson Machine Learning 
```
export COMPONENTS=ws,ws_runtimes,wml,openscale
```
Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```

Run the upgrade command.
```
cpd-cli manage apply-cr \
--components=${COMPONENTS}  \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true \
--upgrade=true
```

Validate the service upgrade status.
```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=${COMPONENTS}
```

#### 2.2.5 Upgrade Db2 Warehouse
```
# 1.Upgrade the service
export COMPONENTS=db2wh

cpd-cli manage apply-cr --components=${COMPONENTS} --release=${VERSION} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --license_acceptance=true --upgrade=true

cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=${COMPONENTS}

# 2. Upgrading Db2 Warehouse service instances
# 2.1. Get a list of your Db2 Warehouse service instances
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME} --service-type=${COMPONENTS}

# 2.2. Upgrade Db2 Warehouse service instances
cpd-cli service-instance upgrade --profile=${CPD_PROFILE_NAME} --instance-name=${INSTANCE_NAME} --service-type=${COMPONENTS}

# 3. Verifying the service instance upgrade
# 3.1. Wait for the status to change to Ready
oc get db2ucluster <instance_id> -o jsonpath='{.status.state} {"\n"}'

#3.2. Check the service instances have updated
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME} --service-type=${COMPONENTS}
```

## Part 3: Post-upgrade

### 3.1 Configuring single sign-on
If post upgrade login using SAML doesn't work, then follow This instruction. You need to use the "/user-home/_global_/config/saml/samlConfig.json" file that you save at the beginning of upgrade.

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=environment-configuring-sso

### 3.2 Validate CPD & CPD services
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
2)Validate whether the Homepage dashboard displays the "Recent projects" and "Notifications" cards.
<br>
If there's no "Recent projects" and "Notifications" cards displayed, then do further check as follows.
<br>
a)Log into the metastore pod.
```
oc rsh zen-metastore-edb-1
```
b)Set up for running the SQL 
```
psql -U postgres -d zen
```
c)Collect the database state for the "Recent projects" and the "Notifications" cards.
```
select * from extensions_view where extension_name='homepage_card_projects';
select * from custom_extensions where extension_name='homepage_card_notifications';
```
If 3 or 4 active records returned by either of the above SQLs, then there could be a problem. The solution in the support ticket TS015636165 should be applied for fixing this problem.

3)Log into CPD web UI with admin and check out each services, including provision instance and functions of each service
<br>
Validate if there are home card issue.

### 3.3 Removing the shared operators
Log the cpd-cli in to the Red Hat OpenShift Container Platform cluster.
```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```

Delete the operators from the ibm-common-services project:
```
cpd-cli manage delete-olm-artifacts \
--cpd_operator_ns=ibm-common-services \
--delete_all_components=true \
--delete_shared_catsrc=true
```
### 3.4 CCS post-upgrade tasks
**1.Add the label back to cronjobs**
<br>
After the reconciliation is completed, add the label back to cronjobs
```
for cj in $(oc get cronjob -l runtimeAssembly --no-headers | grep "<none>" | awk '{print $1}'); do oc label cronjob $cj created-by=spawner 2>/dev/null; done
```
**2.Check if uploading JDBC drivers enabled**
```
oc get ccs ccs-cr -o yaml | grep -i wdp_connect_connection_jdbc_drivers_repository_mode
```
Make sure the `wdp_connect_connection_jdbc_drivers_repository_mode` parameter set to be enabled.

**3.Change heap size in asset-files-api deployment**
<br>
1)Put ccs-cr in maintenance mode.
```
oc patch ccs ccs-cr --type=merge --patch='{"spec":{"ignoreForMaintenance":true}}'
```

2)Edit the deployment asset-files-api

```
oc edit deployment asset-files-api
```

3)Add below section to the asset-files-api deployment and put it under the `name` of the spec.
```
     args:
       - -c
       - |
         cd /home/node/${MICROSERVICENAME}
         source /scripts/exportSecrets.sh
         export npm_config_cache=~node
         node --max-old-space-size=12288 --max-http-header-size=32768 index.js
     command:
     - /bin/bash
```
When added, it looks like this.
```
     name: asset-files-api
     args:
       - -c
       - |
         cd /home/node/${MICROSERVICENAME}
         source /scripts/exportSecrets.sh
         export npm_config_cache=~node
         node --max-old-space-size=12288 --max-http-header-size=32768 index.js
     command:
     - /bin/bash
```
4)Save and exit.
<br>
5)Double check if the heap size change is set as expected.
```
oc get deployment/catalog-api --list | grep -i "--max-old-space-size=12288" -A 5 -B 5
```
**Remove the elasticsearch_java_opts setting in CCS cr**
<br>
1)Edit the ccs-cr.
```
oc edit ccs ccs-cr
```
2)Remove the `"elasticsearch_java_opts": "-Xmx8g -Xms8g"` from the ccs-cr.
3)Save and exit.

### 3.5 WKC post-upgrade tasks
**1.Migration cleanup - legacy features**

```
oc delete scc wkc-iis-scc
oc delete sa wkc-iis-sa
```

If the cleanup is successful, "legacyCleanup" will show Completed.
```
oc get wkc wkc-cr -oyaml | grep "legacyCleanup"
```

Check whether any left overs still there.
```
oc get all -l release=0073-ug
```
Delete the leftovers if there are any.

<br>

[Reference - migration cleanup](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=tasks-migration-cleanup#migration_cleanup__services__title__1)

**2.Enable Relationship Explorer feature**
<br>
[Enable Relationship Explorer feature](https://github.com/sanjitc/Cloud-Pak-for-Data/blob/main/Upgrade/CPD%204.6%20to%204.8/Enabling_Relationship_Explorer_480%20-%20disclaimer%200208.pdf)

**3.Enable 'Allow Reporting' settings for Catalogs and Projects**
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

**4.Customized change for Finley affinity**
<br>
1)Put wkc-cr in maintenance mode.
```
oc patch wkc wkc-cr --type=merge --patch='{"spec":{"ignoreForMaintenance":true}}'
```
2)Backup the finley-public service
```
oc get svc finley-public -o yaml > svc-finley-public-bak.yaml
```
3)Edit the finley-public service
```
oc patch svc finley-public --type='json' -p='[{"op": "replace", "path": "/spec/sessionAffinity", "value": "None" },{ "op": "remove", "path": "/spec/sessionAffinityConfig"}]'
```
4)Make sure the finley-public service patched successfully
```
oc get svc finley-public -o yaml | grep -i sessionAffinity
```

**5.Apply the workaround for the problem - MDE Job failed with error "Deployment not found with given id"**
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

**6.To see your catalogs' assets in the Knowledge Graph, you need to resync your lineage metadata.** 
<br>
[For steps to run the resync, see Resync of lineage metadata](https://www.ibm.com/docs/en/SSQNUZ_4.8.x/wsj/admin/admin-lineage-resync.html)

**7.Expand the Minio PVC size for accomondendatindg the hourly diagnostics collection.** 
<br>
1)Increase the PVC size to 100Gi.
<br>
2)adding the `zenMinioReqSize: 100Gi` property to the lite-cr. This step can be done in another maintenance time window as it may trigger reconcilation.

### 3.6 Handling embedded Postgres license expiry for CPD 4.8.5
https://www.ibm.com/support/pages/node/7158524

### 3.7 Summarize and close out the upgrade

1)Schedule a wrap-up meeting and review the upgrade procedure and lessons learned from it.

2)Evaluate the outcome of upgrade with pre-defined goals.

---

End of document
