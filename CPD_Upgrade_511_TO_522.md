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

#### 3. The permissions required for the upgrade is ready

- Openshift cluster permissions
  `<br>`
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

#### 4. Migrate environments based on Watson Studio Runtime 22.2 and Runtime 23.1 from IBM Cloud Pak® for Data 5.1 (optional)

Runtime 22.2 and Runtime 23.1 are not included in IBM® Software Hub. To continue using notebooks, scripts, and jobs that use environments based on Runtime 22.2 or Runtime 23.1, you must review them and manually migrate them, if necessary.

<br>

[Migrating notebooks, scripts, and jobs that use outdated environments](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=upgrading-migrating-notebooks-scripts-jobs-that-use-outdated-environments)

#### 5. A pre-upgrade health check is made to ensure the cluster's readiness for upgrade.

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
1.2.2 Update cpd_vars.sh for the upgrade to Version 5.2.2
1.2.3 Obtain the olm-utils-v3 available
1.2.4 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image
1.2.5 Ensure the images were mirrored to the private container registry
1.2.6 Creating a profile for upgrading the service instances
1.3 Health check OCP & CPD

Part 2: Upgrade
2.1 Upgrade CPD to 5.2.2
2.1.1 Upgrading shared cluster components
2.1.2 Preparing to upgrade the CPD instance to IBM Software Hub
2.1.3 Upgrading to IBM Software Hub
2.1.4 Upgrading the operators for the services
2.1.5 Applying the RSI patches
2.2 Upgrade CPD services
2.2.1 Upgrading IBM Knowledge Catalog service and apply hot fixes
2.2.2 Upgrading Datalineage service
2.2.3 Upgrading Analytics Engine service
2.2.4 Upgrading Watson Studio, Watson Studio Runtimes, Watson Machine Learning and OpenScale
2.2.5 Upgrading Db2 Warehouse
2.2.6 Upgrading Match360

Part 3: Post-upgrade
3.1 Validate the external vault connection setting 
3.2 CCS post-upgrade tasks
3.3 WKC post-upgrade tasks

Part 4: Maintenance

Summarize and close out the upgrade

```

## Part 1: Pre-upgrade

### 1.1 Collect information and review upgrade runbook

#### 1.1.1 Review the upgrade runbook

Review upgrade runbook

#### 1.1.2 Pre-migration check

Note: Create a folder for 5.2.2 and maintain below created copies in that folder. 
<br>
Login to the OCP cluster for cpd-cli utility.

```
${CPDM_OC_LOGIN}
```

Collect the catalog-api migration relevant information.

- Set  INSTANCE_URL to CPD route

```
export INSTANCE_URL=<CPD route>
```

- Retrieve the token

```
TOKEN=$(oc get -n ${PROJECT_CPD_INST_OPERANDS} secrets wdp-service-id -o yaml | grep service-id-credentials | cut -d':' -f2- | sed -e 's/ //g' | base64 -d)
```

- Get the number of catalogs, projects and spaces:

```
curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=catalogs&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length' > catalogcount.txt

curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=projects&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length' > projects.txt

curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=spaces&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length' > spaces.txt
```

 Check if Postgres migration can be semi-auto or auto.

- Create and run following script ./precheck_migration.sh

```bash
#!/bin/bash

# Default ranges for couchdb size
SMALL=30
LARGE=200

echo "Performing pre-migration checks"

check_resources(){
        scale_config=$1
        pvc_size=$(oc get pvc -n ${PROJECT_CPD_INST_OPERANDS} database-storage-wdp-couchdb-0 --no-headers | awk '{print $4}')
        size=$(awk '{print substr($0, 1, length($0)-2)}' <<< "$pvc_size")

        if [[ "$size" -le "$SMALL" ]];then
          echo "The system is ready for migration. Upgrade your cluster as usual."
        elif [ "$size" -ge "$SMALL" ] && [ "$size" -le "$LARGE" ] && [[ $scale_config == "small" ]];then
          echo -e "Run the following command to increase the CPU and memory:\n"
          cat << EOF
oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge --patch '{"spec": {
  "catalog_api_postgres_migration_threads": 6,
  "catalog_api_migration_job_resources": { 
    "requests": {"cpu": "3", "ephemeral-storage": "10Mi", "memory": "4Gi"},
    "limits": {"cpu": "8", "ephemeral-storage": "4Gi", "memory": "8Gi"}}
}}'
EOF
          echo
          echo "The system is ready for migration. Upgrade your cluster as usual."
        elif [[ $scale_config == "medium" ]];then
          echo -e "Run the following command to increase the CPU and memory:\n"
          cat << EOF
oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge --patch '{"spec": {
  "catalog_api_postgres_migration_threads": 8,
  "catalog_api_migration_job_resources": { 
    "requests": {"cpu": "6", "ephemeral-storage": "10Mi", "memory": "6Gi"},
    "limits": {"cpu": "10", "ephemeral-storage": "6Gi", "memory": "10Gi"}}
}}'
EOF
          echo
          echo "Before you can start the upgrade, you must prepare the system for migration."
        fi
}

check_upgrade_case(){   
        scale_config=$(oc get ccs -n ${PROJECT_CPD_INST_OPERANDS} ccs-cr -o json | jq -r '.spec.scaleConfig')

        # Default case, scale config is set to small
        if [[ -z "${scale_config}" ]];then
          scale_config=small
        fi

        if [[ $scale_config == "large" ]];then
          echo "Before you can start the upgrade, you must prepare the system for migration."
        elif [[ $scale_config == "small" ]] || [[ $scale_config == "medium" ]];then
          check_resources $scale_config
        fi
}

check_upgrade_case
```

Backup the RSI patches.

```
cpd-cli manage get-rsi-patch-info \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--all
```

#### 1.1.3 Uninstall all hotfixes and apply preventative measures

Remove the hotfixes by removing the images or configurations from the CRs. (Note: Hot fixes should already be removed)
<br>

1. Double check WKC maintenance mode and usefdb configuration. (WKC handles Hotfix removal on its own)

```
useFDB: false
ignoreForMaintenance: true
```

4)Save and Exit. Wait until the WKC Operator reconcilation completed and also the wkc-cr in 'Completed' status.

```
oc get WKC wkc-cr -o yaml
```

2. Double check AnalyticsEngine SELinux config. Hotfix auto removal.

Make change for skipping SELinux Relabeling

```
  serviceConfig:
    skipSELinuxRelabeling: true
```

4)Save and Exit. Wait until the AnalyticsEngine Operator reconcilation completed and also the analyticsengine-sample in 'Completed' status.

```
oc get AnalyticsEngine analyticsengine-sample -o yaml
```

- 3.Patch the CCS and uninstall the CCS hot fixes.
  `<br>`

1)Edit the CCS cr with below command. Hotfix auto removal is enabled

2)Apply preventative measures for OpenSearch pvc customization problem

`<br>`
This step is for applying the preventative measures for OpenSearch problem. Applying the preventative measures in this timing can also help to minimize the number of CCS operator reconcilations.
`<br>`
List OpenSearch PVC sizes, and make sure to preserve the type, and the size of the largest one (PVC names may be different depending on client environment):
`<br>`

```
oc get pvc | grep elasticsea
dev                        data-elasticsea-0ac3-ib-6fb9-es-server-esnodes-0           Bound    pvc-ac773a46-0110-48f6-be27-51b6db332945   150Gi      RWO  
dev                        data-elasticsea-0ac3-ib-6fb9-es-server-esnodes-1           Bound    pvc-1e73b25c-c0ca-4bdd-b5ba-b4484e18ade9   150Gi      RWO
dev                        data-elasticsea-0ac3-ib-6fb9-es-server-esnodes-2           Bound    pvc-1a5a6c47-4221-4ecf-96e4-ddef689fd527   150Gi      RWO
dev                        elasticsea-0ac3-ib-6fb9-es-server-snap                     Bound    pvc-eba99ac7-0f9f-481e-883c-56dc8e9ca65c   608Gi      RWX
dev                        elasticsearch-master-backups                               Bound    pvc-8ab9d47d-de90-449c-99b8-89fed818c727   608Gi      RWX
```

In the above example, `150Gi` is the OpenSearch pvc size, `608Gi` is backup/snapshot storage size.
`<br>`
**Note** if PVCs are of different sizes, we want to make sure to take the biggest one.
`<br>`

In CCS CR make sure to set the following properties, with above values used as example:

```
elasticsearch_persistence_size: "608Gi"
elasticsearch_backups_persistence_size: "608Gi"
```

This will make sure that the Opensearch operator will properly reconcile, - as provided values will match the state of the cluster.

4) (Search has already been Migrated)

6)Remove the `ignoreForMaintenance: true` from the CCS custom resource

7)Save and Exit. Wait until the CCS Operator reconcilation completed and also the ccs-cr in 'Completed' status.

```
oc get CCS ccs-cr -o yaml
```

8)Wait until the WKC Operator reconcilation completed and also the wkc-cr in 'Completed' status.

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

<br>
Save and Exit. Wait until the ZenService Operator reconcilation completed and also the lite-cr in 'Completed' status. 
<br>

- 5.Remove stale secret of global search
  Check if the elasticsearch-master-ibm-elasticsearch-cred-secret exists.

```
oc get secret -n ${PROJECT_CPD_INST_OPERANDS} | grep elasticsearch-master-ibm-elasticsearch-cred-secret
```

If yes, then delete this stale secret.

```
oc delete elasticsearch-master-ibm-elasticsearch-cred-secret -n ${PROJECT_CPD_INST_OPERANDS}
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
wget https://github.com/IBM/cpd-cli/releases/download/v14.1.1/cpd-cli-linux-EE-14.2.2.tgz
```

2. Install tools.

```
yum install openssl httpd-tools podman skopeo wget -y
```

The version in below commands may need to be updated accordingly.

```
tar xvf cpd-cli-linux-EE-14.2.2.tgz
mv cpd-cli-linux-EE-14.1.1-1542/* .
rm -rf cpd-cli-linux-EE-14.2.2-1542
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
`<br>`
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
export COMPONENTS=ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,ws,ws_runtimes,wml,wkc,datastage_ent,datastage_ent_plus,analyticsengine,mantaflow,datalineage,openscale,db2wh,match360
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

### 2.1 Upgrade CPD to 5.2.2

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

Make sure the project returned by the command matches the environment variable PROJECT_LICENSE_SERVICE in your environment variables script `cpd_vars_522.sh`.
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
--entitlement=cpd-enterprise \
--production=false
```

Reference: `<br>`

[Applying your entitlements](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=puish-applying-your-entitlements)

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

3.Check to see if Elastic Search and catalog-api is running before upgrading.

```
oc get pods --namespace=${PROJECT_CPD_INST_OPERANDS} | grep elasticsea-0ac3

oc get pods --namespace=${PROJECT_CPD_INST_OPERANDS} | grep catalog-api
```

4.Upgrade the required operators and custom resources for the instance. (Storage test should be performed already)

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
oc get csv ibm-cpd-ccs.v10.1.1 -n ${PROJECT_CPD_INST_OPERATORS} -o yaml > ibm-cpd-ccs-csv-522.yaml
```

Edit the CCS CSV:

```
oc edit csv ibm-cpd-ccs.v10.1.1 -n ${PROJECT_CPD_INST_OPERATORS} 
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

3).Check the RSI patches status again: (Note: Profiling RSI path from 4.8.5 to 5.1.1 should still be configured)

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
    enableKnowledgeGraph: True
    enableDataQuality: True
    useFDB: False
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

2)Make sure the `useFDB` is set to be `False` in the install-options.yml file. And remove FDBcluster for WKC if present.

```bash
oc get fdbcluster -n ${PROJECT_CPD_INST_OPERANDS}

(If present)
oc delete fdbcluster wkc-foundationdb-cluster -n ${PROJECT_CPD_INST_OPERANDS}
```

`<br>`

##### 2.Upgrade WKC with custom installation

Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
${CPDM_OC_LOGIN}
```

Delete the FDB cluster for WKC if present.

```
oc get fdbcluster -n ${PROJECT_CPD_INST_OPERANDS} |grep wkc

oc delete fdbcluster wkc-foundationdb-cluster -n  ${PROJECT_CPD_INST_OPERANDS}
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

##### 4.Apply the customizations (Should not need any hotfixes)

#### 2.2.2 Upgrading IBM MANTA Lineage service

```
export COMPONENTS=datalineage

```

- Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
${CPDM_OC_LOGIN}
```

- Run the command for upgrading MANTA service.

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

**Apply Profiling Lineage Hotfix (Should no longer need any hotfixes)**

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=datalineage
```

#### 2.2.3 Upgrading Analytics Engine service

##### 2.2.3.1 Upgrading the service

Check the Analytics Engine service version and status.

```
export COMPONENTS=analyticsengine

cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=${COMPONENTS}
```

The Analytics Engine service should have been upgraded as part of the WKC service upgrade. If the Analytics Engine service version is **not 5.2.2**, then run below commands for the upgrade. `<br>`

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

1)Add Verizon logo on CPD homepage

<br>

[Customizing the branding of the web client](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=users-customizing-branding-web-client)

<br>

2)Create a custom route

<br>

[Create a custom route using cpd-cli](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=platform-modifying-route)

**Note**
`<br>`

Refer to the backup file `routes.yaml` created in the Step **1.1.2**.

### 3.2 CCS post-upgrade tasks (Need to check with Steven)

https://www.ibm.com/docs/en/software-hub/5.2.x?topic=services-completing-catalog-api-migration

https://www.ibm.com/docs/en/software-hub/5.2.x?topic=upgrading-post-upgrade-setup-knowledge-catalog

### 3.3 WKC post-upgrade tasks (Need to check with Steven)

## Part 4: Maintenance (Migrated to leverage Sanjit's Runbook)

This part is beyond the upgrade scope. And we are not commited to complete them in the two days time window.

## Summarize and close out the upgrade

1)Schedule a wrap-up meeting and review the upgrade procedure and lessons learned from it.

2)Evaluate the outcome of upgrade with pre-defined goals.

---

End of document
