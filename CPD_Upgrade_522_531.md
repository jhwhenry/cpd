
# CPD Upgrade From 5.2.2 to 5.3.1 Patch 3

## Upgrade Context
- **OCP:** 4.16
- **CPD:** 5.2.2 → 5.3.1 Patch 3
- **Storage:** ODF
- **Components:** cpd_platform,wkc,analyticsengine,datalineage,ws,ws_runtimes,wml,openscale,db2wh,match360
- **Airgapped:** Yes

# Table of Contents
- 1. Pre-upgrade
- 2. Upgrade
- 3. Post-upgrade tasks

# Pre-requisites

### 1. Backup of the cluster is done.

Backup your Cloud Pak for Data cluster before the upgrade.

**Note:**
Make sure there are no scheduled backups conflicting with the scheduled upgrade.

### 2. The image mirroring completed successfully

If a private container registry is in-use to host the IBM Software Hub software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry. 

### 3. The CASE files and cluster resource files downloaded successfully

Before upgrading IBM Software Hub platform or any services, you must download the required cluster‑scoped resources, such as ClusterRoles and ClusterRoleBindings—for the components you plan to upgrade. Ensure that these files are available on the bastion node for use during the upgrade.

### 4. The permissions required for the upgrade is ready

- Openshift cluster permissions
<br>An Openshift cluster administrator can complete all of the installation tasks.

- IBM Software Hub permissions
<br>The Cloud Pak for Data administrator role or permissions is required for upgrading the service instances.

- Permission to access the private image registry for pushing or pull images

- Red Hat pull secret for mirroring images of Red Hat Cert Manager

- Access to the Bastion node for executing the upgrade commands

### 5. A pre-upgrade health check is made to ensure the cluster's readiness for upgrade.

- The OpenShift cluster, persistent storage, IBM Software Hub platform and services are in healthy status.

### 6. Migrating to Red Hat OpenShift certificate manager

The IBM Certificate manager is deprecated.

If the IBM Certificate manager (ibm-cert-manager) is installed on your cluster, refer to IBM Documentation to migrate your certificates from the IBM Certificate manager to the Red Hat OpenShift certificate manager (cert-manager Operator).

[Migrating from the IBM Certificate manager to the Red Hat OpenShift certificate manager](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading-migrating-red-hat-openshift-certificate-manager)

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
### 1.1.2 Tune the ccs-cams-postgres cluster
#### 1.1.2.1 Patch the ccs-cams-postgres cluster.
```
oc patch clusters.postgresql.k8s.enterprisedb.io ccs-cams-postgres \
  -n ${PROJECT_CPD_INST_OPERANDS} \
  --type merge \
  --patch '
spec:
  primaryUpdateMethod: restart
  postgresql:
    parameters:
      wal_keep_size: "4GB"
      wal_receiver_timeout: "30s"
      wal_sender_timeout: "30s"
      wal_compression: "on"
'
```
Validate whether the patch applied successfully.
```
oc get clusters.postgresql ccs-cams-postgres -o json |jq .spec.primaryUpdateMethod
```
The output should be `restart`.

#### 1.1.2.2 Make sure the ccs-cams-postgres cluster is healthy
Check the ccs-cams-postgres cluster status. 
```
oc cnp status ccs-cams-postgres
```

Make sure it is Healthy and replica is up to date (current LSN are the same), like below.
```
Cluster Summary
Name                 wkc/ccs-cams-postgres
System ID:           7632433098929098770
PostgreSQL Image:    cp.icr.io/cpopen/edb/postgresql:16.10-5.22.0-amd64@sha256:66edda691d6e760c8de46a9796664ea70cfea9052b4de65ecb3e42139d5dea3c
Primary instance:    ccs-cams-postgres-1
Primary start time:  2026-04-24 20:59:01 +0000 UTC (uptime 64h56m28s)
Status:              Cluster in healthy state            <<<<<<<<<<<<<  Cluster is in Healthy state
Instances:           2
Ready instances:     2
Size:                595M
Current Write LSN:   0/23000000 (Timeline: 1 - WAL File: 000000010000000000000022)

Streaming Replication status
Replication Slots Enabled
Name                 Sent LSN    Write LSN   Flush LSN   Replay LSN  Write Lag  Flush Lag  Replay Lag  State      Sync State  Sync Priority  Replication Slot
----                 --------    ---------   ---------   ----------  ---------  ---------  ----------  -----      ----------  -------------  ----------------
ccs-cams-postgres-2  0/23000000  0/23000000  0/23000000  0/23000000  00:00:00   00:00:00   00:00:00    streaming  async       0              active

Instances status
Name                 Current LSN  Replication role  Status  QoS        Manager Version  Node
----                 -----------  ----------------  ------  ---        ---------------  ----
ccs-cams-postgres-1  0/23000000   Primary           OK      Burstable  1.25.3           worker4.cams-vz-522.cp.fyre.ibm.com
ccs-cams-postgres-2  0/23000000   Standby (async)   OK      Burstable  1.25.3           worker3.cams-vz-522.cp.fyre.ibm.com
                   ^^^^^^^^^^^^^^. Current LSN are the same
```

#### 1.1.2.3 Make a backup for CAMS postgres
Make a backup for CAMS postgres by referring to the step 5 in this documentation [Backing up the PostgreSQL database](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=services-completing-catalog-api-migration#catalog-api-migration__backup__title__1)

### 1.1.3 Global Search legacy index compatibility check before upgrade
**Note:** the step 1.1.3 and 1.1.4 can be done as one-go.
<br>
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

### 1.1.4 Uninstall all hot fixes if any

Uninstal the CCS (portal-project) hotfix.
<br>
Edit the CCS custom resource.
```
oc edit ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

Remove the hot fix.

```
image_digests:
  portal_projects_image: sha256:e9f85d5e4f0c021d8d16443012cf4de3ec823e16731ba17a3be99a6c233d4849
```

Save and Exit. Wait until the CCS Operator reconcilation completed and also the ccs-cr in 'Completed' status.

```
oc get CCS ccs-cr -o yaml -n ${PROJECT_CPD_INST_OPERANDS}
```

### 1.1.5 Take the DataLineage service out of maintenance mode

```
oc patch DataLineage datalineage-cr -p "{\"spec\":{\"ignoreForMaintenance\": false}}" --type=merge -n ${PROJECT_CPD_INST_OPERANDS}
```

Wait until the DataLineage Operator reconcilation completed and also the datalineage-cr in 'Completed' status.

```
oc get DataLineage datalineage-cr -n ${PROJECT_CPD_INST_OPERANDS}
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

## 1.3 Installing Helm CLI

[Installing Helm](https://www.ibm.com/links?url=https%3A%2F%2Fhelm.sh%2Fdocs%2Fintro%2Finstall%2F)

## 1.4 Updating your environment variables script
Make a copy of the environment variables script used by the existing 5.2.2 variables with the name like `cpd_vars_531.sh`.

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
export COMPONENTS=cpd_platform,wkc,analyticsengine,datalineage,ws,ws_runtimes,wml,openscale,db2wh,match360
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

### 1.5.2 Downloading the cluster-scoped resources for the platform and services

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
## 2.1 Migrating to Red Hat OpenShift certificate manager

The IBM Certificate manager is deprecated.

If the IBM Certificate manager (ibm-cert-manager) is installed on your cluster, use the following steps to migrate your certificates from the IBM Certificate manager to the Red Hat OpenShift certificate manager (cert-manager Operator).

[Migrating from the IBM Certificate manager to the Red Hat OpenShift certificate manager](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading-migrating-red-hat-openshift-certificate-manager)

## 2.2 Upgrading the License Service

### 2.2.1 Get the project of the License service

If you're not sure which project the License Service is in, run the following command:
```
oc get deployment -A | grep ibm-licensing-operator
```

### 2.2.2  Log in to the Red Hat OpenShift Container Platform cluster
```
${CPDM_OC_LOGIN}
```

### 2.2.3 Upgrading the License Service

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

## 2.3 Preparing to upgrade IBM Software Hub

### 2.3.1 Updating the cluster-scoped resources for the platform and services

1.Generate cluster-scoped resources for platform and services
<br>

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

## 2.4 Upgrading IBM Software Hub
## 2.4.1 Creating image pull secrets for an instance of IBM Software Hub
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

### 2.4.2 Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

### 2.4.3 Upgrading the required operators and custom resources for the instance
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

Once the above command `cpd-cli manage install-components` is completed, make sure the status of the IBM Software Hub is in 'Completed' status.
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \ 
--components=cpd_platform
```

### 2.4.4 Applying the RSI patches
Run the following command to re-apply your existing custom patches.
```
cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check the RSI patches status again: 
```
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --all
```

## 2.5 Upgrading WKC

### 2.5.1 Preparing to upgrade IBM Knowledge Catalog

Before you can upgrade IBM Knowledge Catalog to Version 5.3, revert the temporary patches if any.
<br>
[Reverting temporary patches](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=u-upgrading-from-version-52-33#cli-upgrade__patch-revert__title__1)

### 2.5.2 Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

### 2.5.3 Upgrading the operator and custom resource for IBM Knowledge Catalog

```
cpd-cli manage install-components \
--license_acceptance=true \
--components=wkc \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

Once the above command `cpd-cli manage install-components` completed successfully, you can run the `cpd-cli manage get-cr-status` command for the validation.

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=wkc
```

Make sure the ccs-cams-postgres cluster is healthy.

```
oc cnp status ccs-cams-postgres
```
Make sure it is Healthy and replica is up to date (current LSN are the same).


## 2.6 Upgrading DataStage, MANTA Automated Data Lineage and Data Management Console

### 2.6.1 Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

### 2.6.2 Upgrading the operator and custom resource

```
cpd-cli manage install-components \
--license_acceptance=true \
--components=datastage_ent,datalineage \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

Once the above command `cpd-cli manage install-components` completed successfully, run the `cpd-cli manage get-cr-status` command for the validation.
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=datastage_ent,datalineage
```

## 2.8 Upgrading Watson Studio, Watson Machine Learning, OpenScale

### 2.8.1 Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

### 2.8.2 Upgrading the operator and custom resource
```
cpd-cli manage install-components \
--license_acceptance=true \
--components=ws,ws_runtimes,wml,openscale \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

Once the above command `cpd-cli manage install-components` completed successfully, run the `cpd-cli manage get-cr-status` command for the validation.

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=ws,ws_runtimes,wml,openscale
```

## 2.9 Upgrading Db2 Warehouse and Match 360

### 2.9.1 Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

### 2.9.2 Upgrading the operator and custom resource
```
cpd-cli manage install-components \
--license_acceptance=true \
--components=db2wh,match360 \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

Once the above command `cpd-cli manage install-components` completed successfully, run the `cpd-cli manage get-cr-status` command for the validation.

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=db2wh,match360
```

# 3. Post-upgrade tasks

#### 3.1 Creating a profile to use the cpd-cli management commands
Create a profile on the workstation from which you will upgrade the service instances.
<br>
[Creating a profile to use the cpd-cli management commands](https://www.ibm.com/docs/en/SSNFH6_5.3.x/cpd-cli/cpd-profile-mgmt.html)
<br>
**Note:**
The profile must be associated with a IBM Software Hub user who has either the following permissions:
<br>
- Create service instances (can_provision)
- Manage service instances (manage_service_instances)


## 3.2 Post-upgrade of WKC

### 3.2.1 Upgrading Analytics Engine service instance

- Set the environment variable `CPD_PROFILE_NAME`.
<br>

```
export CPD_PROFILE_NAME=<the profile name created in the Step 3.1>
```

- Upgrading the Spark service instance

```
cpd-cli service-instance upgrade \
--service-type=spark \
--profile=${CPD_PROFILE_NAME} \
--all
```

- Validating the service instance upgrade status.

```
cpd-cli service-instance list \
--service-type=spark \
--profile=${CPD_PROFILE_NAME}
```

### 3.2.2 Recreate missing CAMS Postgres indexes
Connect to the ccs-cams-postgres database.

```
oc -n ${PROJECT_CPD_INST_OPERANDS} exec -it ccs-cams-postgres-1 sh
sh-5.1$psql -h ccs-cams-postgres-rw -p 5432 -U cams_user -d camsdb
```

Create the missing indexes.

```
-- Catalog indexes
CREATE INDEX CONCURRENTLY IF NOT EXISTS catalog_bss_subtype_state_idx ON cams.catalog(bss_account, subtype, state, is_public, is_consolidated_db) WHERE bss_account IS NOT NULL;  
   
CREATE INDEX CONCURRENTLY IF NOT EXISTS catalog_active_archive_order_idx ON cams.catalog(state, version, create_time_ticks) WHERE archive_info_state IS NULL; 
     
CREATE INDEX CONCURRENTLY IF NOT EXISTS catalog_bss_uid_state_idx ON cams.catalog(bss_account, uid, state);   
      
CREATE INDEX CONCURRENTLY IF NOT EXISTS catalog_bucket_lookup_idx ON cams.catalog(version, state) INCLUDE (id, bucket_container_ids, bucket_states) WHERE bucket_container_ids IS NOT NULL
 AND bucket_states IS NOT NULL;
      
-- Asset type indexes 
CREATE INDEX CONCURRENTLY IF NOT EXISTS asset_type_active_id_order_idx ON cams.asset_type(id) WHERE asset_type_state_state IN ('CREATED', 'PROCESSING');  
     
CREATE INDEX CONCURRENTLY IF NOT EXISTS asset_type_state_id_idx ON cams.asset_type(asset_type_state_state, id);   
     
CREATE INDEX CONCURRENTLY IF NOT EXISTS asset_type_tenancy_account_idx ON cams.asset_type(asset_type_tenancy_level, bss_account_id, asset_type_state_state) INCLUDE (id); 
```

## 3.3 Post-upgrade of Db2Wh
### 3.3.1 Upgrading existing service instances
[Upgrading existing service instances](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=u-upgrading-from-version-52-41#cli-upgrade__svc-inst__title__1)

## 3.4 Upgrade the cpdbr service
[Updating the cpdbr service](https://www.ibm.com/docs/en/SSNFH6_5.3.x/hub/upgrade/v52/upgrade-platform-bar-recipe.html)
