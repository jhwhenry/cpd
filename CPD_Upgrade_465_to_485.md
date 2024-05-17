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
Componenets: cpfs,cpd_platform,ws,ws_runtimes,wml,datastage_ent,wkc,analyticsengine,openscale,db2wh,dp
```

To

```
OCP: 4.12
CPD: 4.8.5
Storage: Storage Fusion 2.7.2
Componenets: cpfs,cpd_platform,ws,ws_runtimes,wml,datastage_ent,wkc,analyticsengine,openscale,db2wh,dp
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

2) Mirror image for RSI adm controller
```
cpd-cli manage copy-image \
--from=icr.io/cpopen/cpd/zen-rsi-adm-controller:4.8.5-x86_64 \
--to=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/zen-rsi-adm-controller:4.8.5-x86_64
```
3) Mirror image for Manta
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
1.1.3 If you installed hotfixes, uninstall all hotfixes
1.1.4 1.1.4 If you installed the resource specification injection (RSI) feature, uninstall the cluster-scoped webhook
1.1.5 If use SAML SSO, export SSO configuration
1.1.6 Enable default admin user if it's disabled. Once upgrade complet you can disable it again.
1.2 Set up client workstation 
1.2.1 Prepare a client workstation
1.2.2 Update cpd_vars.sh for the upgrade to Version 4.8.5
1.2.3 Make olm-utils available
1.2.4 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image
1.2.5 Ensure the images were mirrored to the private container registry
1.2.6 Creating a profile for upgrading the service instances
1.2.7 Download CASE files
1.3 Health check OCP & CPD

Part 2: Upgrade
2.1 Upgrade CPD to 4.8.5
2.1.1 Migrate to private topology
2.1.2 Preparing to upgrade an CPD instance
2.1.3 Upgrade foundation service and CPD platform to 4.8.5
2.1.3 Upgrade foundation service
2.1.4 Upgrade CPD platform
2.2 Upgrade CPD services
2.2.1 Upgrade IBM Knowledge Catalog service
2.2.2 Upgrade MANTA service
2.2.3 Upgrade Analytics Engine service
2.2.4 Upgrade DataStage Enterprise
2.2.5 Upgrade Data Privacy,Watson Studio, Watson Studio Runtimes and Watson Machine Learning
2.2.6 Upgrade Db2 Warehouse

Part 3: Post-upgrade
3.1 Configuring single sign-on
3.2 Validate CPD & CPD services
3.3 Enabling users to upload JDBC drivers
3.4 Removing the shared operators
3.5 WKC post-upgrade tasks
3.6 Summarize and close out the upgrade
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

#### 1.1.3 If you installed hotfixes, uninstall all hotfixes
Edit custom resources (e.g. Zensevice, CCS, WKC, AE) and remove all hotfix references if any.

#### 1.1.4 If you installed the resource specification injection (RSI) feature, uninstall the cluster-scoped webhook
1.Run the cpd-cli manage login-to-ocp command to log in to the cluster as a user with sufficient permissions.
```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```
2.Get a list of all patches in the project:
```
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --all

cat cpd-cli-workspace/olm-utils-workspace/work/get_rsi_patch_info.log
```
3.Set active patches to inactive.
- Inactivate rsi-asset-files-api-annotation-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-asset-files-api-annotation-selinux \
--state=inactive
```
- Inactivate rsi-asset-files-api-pod-spec-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-asset-files-api-pod-spec-selinux \
--state=inactive
```
- Inactivate rsi-create-dap-directories-annotation-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-create-dap-directories-annotation-selinux \
--state=inactive
```
- Inactivate rsi-create-dap-directories-pod-spec-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-create-dap-directories-pod-spec-selinux \
--state=inactive
```
- Inactivate rsi-dp-api-annotation-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-dp-api-annotation-selinux \
--state=inactive
```

- Inactivate rsi-dp-api-pod-spec-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-dp-api-pod-spec-selinux \
--state=inactive
```

- Inactivate rsi-event-logger-api-annotation-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-event-logger-api-annotation-selinux \
--state=inactive
```

- Inactivate rsi-event-logger-api-pod-spec-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-event-logger-api-pod-spec-selinux \
--state=inactive
```

- Inactivate rsi-finley-public-service-patch.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-finley-public-service-patch \
--state=inactive
```

- Inactivate rsi-iae-nginx-ephemeral-patch.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-iae-nginx-ephemeral-patch \
--state=inactive
```

- Inactivate rsi-mde-service-manager-patch.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-mde-service-manager-patch \
--state=inactive
```

- Inactivate rsi-mde-service-manager-patch-2.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-mde-service-manager-patch-2 \
--state=inactive
```

- Inactivate rsi-spark-runtimes-annotation-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-spark-runtimes-annotation-selinux \
--state=inactive
```

- Inactivate rsi-spark-runtimes-pod-spec-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-spark-runtimes-pod-spec-selinux \
--state=inactive
```
4.Check the RSI patches status again:
```
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --all

cat cpd-cli-workspace/olm-utils-workspace/work/get_rsi_patch_info.log
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
--rsi_image=${PRIVATE_REGISTRY_LOCATION}/cpd/zen-rsi-adm-controller:4.6.5-x86_64
```

**Important:** 
<br>We need to disable the RSI patches. Once the upgrade is complete we can enable them depending on if not fixed on 4.8.5.
<br>If you plan to use the RSI feature in IBM Cloud Pak for Data Version 4.8, skip the step to delete the patches. You can re-activate the patches after you install the RSI-webhook at the instance-level.

#### 1.1.5 If use SAML SSO, export SSO configuration

If you use SAML SSO, export your SSO configuration. You will need to reapply your SAML SSO configuration after you upgrade to Version 4.8. Skip this step if you use the IBM Cloud Pak foundational services Identity Management Service

```
oc cp -n=${PROJECT_CPD_INSTANCE} \
$(oc get pods -l component=usermgmt -n ${PROJECT_CPD_INSTANCE} \
-o jsonpath='{.items[0].metadata.name}'):/user-home/_global_/config/saml ./samlConfig.json
```
#### 1.1.6 Enable default admin user if it's disabled. Once upgrade complet you can disable it again.

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.6.x?topic=users-disabling-default-admin-user#taskremove-admin__postreq__1

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
**Note:** 
<br>When adding the following environment variables, The value of PROJECT_CPD_INST_OPERANDS is the same as that of PROJECT_CPD_INSTANCE.
```
export PROJECT_CERT_MANAGER=ibm-cert-manager
export PROJECT_LICENSE_SERVICE=ibm-licensing
export PROJECT_CS_CONTROL=ibm-licensing
export PROJECT_CPD_INST_OPERATORS=cpd-operators
export PROJECT_CPD_INST_OPERANDS=cpd-instance
```
3.Remove the PROJECT_CATSRC entry from the Projects section of the script.

4.Locate the COMPONENTS entry and upate the COMPONENTS entry.
If the advanced metadata import feature in IBM® Knowledge Catalog is used, add the mantaflow component to the COMPONENTS variable.
```
export COMPONENTS=ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,ws,ws_runtimes,wml,datastage_ent,wkc,analyticsengine,mantaflow,openscale,db2wh,dp
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
If the bastion node is internet connected, then you can ignore below steps in this section.
<br>
Go to the client workstation with internet

```
cd ${CPD485_WORKSPACE}
source cpd_vars_485.sh

cpd-cli manage save-image \
--from=icr.io/cpopen/cpd/olm-utils-v2:latest
```

This command saves the image as a compressed TAR file named icr.io_cpopen_cpd_olm-utils-v2_latest.tar.gz in the cpd-cli-workspace/olm-utils-workspace/work/offline directory

Ship the tarbll into bastion node

Go to bastion node

```
cpd-cli manage load-image \
--source-image=icr.io/cpopen/cpd/olm-utils-v2:latest
```

The command returns the following message when the image is loaded:

```
Loaded image: icr.io/cpopen/cpd/olm-utils-v2:latest
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

#### 1.2.7 Download CASE files

1. Go to the client workstation with internet
Log into IBM registry and list images

```
cpd-cli manage login-entitled-registry \
${IBM_ENTITLEMENT_KEY}
```

Download case package. <br>
```
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION}
```

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

### 2.1 Upgrade CPD to 4.8.5

#### 2.1.1 Migrate to private topology
1.Create new projects
```
${OC_LOGIN}
oc new-project ${PROJECT_CS_CONTROL}             # This is for ibm-licensing operator and instance
oc new-project ${PROJECT_CERT_MANAGER}           # This is for ibm-cert-manager operator and instance
oc new-project ${PROJECT_LICENSE_SERVICE}        # This is for the License Service
oc new-project ${PROJECT_CPD_INST_OPERATORS}     # This is for migrated cpd operator
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
--license_acceptance=true
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

6. **Re-activate the RSI patches.**
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
4).Set inactive patches to active.

- Activate rsi-asset-files-api-annotation-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-asset-files-api-annotation-selinux \
--state=active
```
- Activate rsi-asset-files-api-pod-spec-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-asset-files-api-pod-spec-selinux \
--state=active
```
- Activate rsi-create-dap-directories-annotation-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-create-dap-directories-annotation-selinux \
--state=active
```
- Activate rsi-create-dap-directories-pod-spec-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-create-dap-directories-pod-spec-selinux \
--state=active
```
- Activate rsi-dp-api-annotation-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-dp-api-annotation-selinux \
--state=active
```

- Activate rsi-dp-api-pod-spec-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-dp-api-pod-spec-selinux \
--state=active
```

- Activate rsi-event-logger-api-annotation-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-event-logger-api-annotation-selinux \
--state=active
```

- Activate rsi-event-logger-api-pod-spec-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-event-logger-api-pod-spec-selinux \
--state=active
```

- Activate rsi-finley-public-service-patch.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-finley-public-service-patch \
--state=active
```

- Activate rsi-iae-nginx-ephemeral-patch.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-iae-nginx-ephemeral-patch \
--state=active
```

- Activate rsi-mde-service-manager-patch.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-mde-service-manager-patch \
--state=active
```

- Activate rsi-mde-service-manager-patch-2.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-mde-service-manager-patch-2 \
--state=active
```

- Activate rsi-spark-runtimes-annotation-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-spark-runtimes-annotation-selinux \
--state=active
```

- Activate rsi-spark-runtimes-pod-spec-selinux.
```
cpd-cli manage create-rsi-patch \
--cpd_instance_ns=${PROJECT_CPD_INSTANCE} \
--patch_name=rsi-spark-runtimes-pod-spec-selinux \
--state=active
```
5).Check the RSI patches status again:
```
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --all

cat cpd-cli-workspace/olm-utils-workspace/work/get_rsi_patch_info.log
```

### 2.2 Upgrade CPD services to 4.8.5
#### 2.2.1 Upgrade IBM Knowledge Catalog service
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
    enableDataQuality: True
```
##### 2.Upgrade WKC with custom installation

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

##### 3.Validate the upgrade
```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

##### 4.Apply the hotfixes if available.


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

If the Analytics Engine service version is **not 4.8.5**, then run below commands for the upgrade. <br>

Check if the Analytics Engine service was installed with the custom install options. <br>

- If it's custom installation, check the previous install-options.yaml or analyticsengine-sample yaml, make sure to keep original custom settings.
```
vim cpd-cli-workspace/olm-utils-workspace/work/install-options.yml
```

Then run this upgrade command.
```
cpd-cli manage apply-cr \
--components=analyticsengine \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--param-file=/tmp/work/install-options.yml \
--license_acceptance=true \
--upgrade=true
```

- If it's **NOT** custom installation, then run this upgrade command.

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

#### 2.2.4 Upgrade DataStage Enterprise 
Check the DataStage Enterprise service version and status.
```
export COMPONENTS=datastage_ent

cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=${COMPONENTS}
```

If the DataStage Enterprise service version is **not 4.8.5**, then run below commands for the upgrade. <br>

```
cpd-cli manage apply-cr --components=${COMPONENTS} --release=${VERSION} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --block_storage_class=${STG_CLASS_BLOCK} --file_storage_class=${STG_CLASS_FILE} --license_acceptance=true --upgrade=true
```

Validate the upgrade status.
```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=${COMPONENTS}
```

#### 2.2.5 Upgrade Data Privacy,Watson Studio, Watson Studio Runtimes and Watson Machine Learning 
```
export COMPONENTS=dp,ws,ws_runtimes,wml,openscale
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

#### 2.2.6 Upgrade Db2 Warehouse
```
# 1.Upgrade the service
export COMPONENTS=db2wh

cpd-cli manage apply-cr --components=${COMPONENTS} --release=${VERSION} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --license_acceptance=true --upgrade=true

cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=${COMPONENTS}

# 2. Upgrading Db2 Warehouse service instances
# 2.1. Get a list of your Db2 Warehouse service instances
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME} --service-type=${COMPONENTS}

# 2.2. If you have applied any custom patches to override scripts, remove them. This will restart Db2 Warehouse pods. 
oc set volume statefulset/c-${DB2U_ID}-db2u -n ${PROJECT_CPD_INST_OPERANDS} --remove --name=<volume_name>

# 2.3. Upgrade Db2 Warehouse service instances
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

Log into CPD web UI with admin and check out each services, including provision instance and functions of each service

### 3.3 Enabling users to upload JDBC drivers
Reference: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=environment-enabling-users-upload-jdbc-drivers

#### 3.3.1 Set the wdp_connect_connection_disable_jar_tab parameter to false
```
oc patch ccs ccs-cr \
--namespace=${PROJECT_CPD_INST_OPERANDS} \
--type=merge \
--patch '{"spec": {"wdp_connect_connection_jdbc_drivers_repository_mode": "enabled"}}'
```

#### 3.3.2 Wait for the common core services status to be Completed
```
oc get ccs ccs-cr --namespace=${PROJECT_CPD_INST_OPERANDS}
```

### 3.4 Removing the shared operators
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
--delete_all_components=true
```

Delete the operators from the ${PROJECT_CPD_OPS} project:
```
cpd-cli manage delete-olm-artifacts \
--cpd_operator_ns=${PROJECT_CPD_OPS} \
--delete_all_components=true \
--delete_shared_catsrc=true
```
### 3.5 WKC post-upgrade tasks
1.Migration cleanup - legacy features

```
oc -n ${NAMESPACE} patch wkc/wkc-cr --patch '{"spec":{"legacyCleanup":true}}' --type=merge
oc delete scc wkc-iis-scc
oc delete sa wkc-iis-sa
```
If the cleanup is successful, "legacyCleanup" will show Completed.
```
oc get wkc wkc-cr -oyaml | grep "legacyCleanup"
```
Reference: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=tasks-migration-cleanup#migration_cleanup__services__title__1
2.Run the bulk sync utility before start using Global Search indexed data for relationships
Follow the step in [Bulk sync relationships for global search (IBM Knowledge Catalog)](https://www.ibm.com/docs/en/SSQNUZ_4.8.x/wsj/admin/admin-bulk-sync.html)

3.To see your catalogs' assets in the Knowledge Graph, you need to resync your lineage metadata. 
[For steps to run the resync, see Resync of lineage metadata](https://www.ibm.com/docs/en/SSQNUZ_4.8.x/wsj/admin/admin-lineage-resync.html)

### 3.6 Summarize and close out the upgrade

Schedule a wrap-up meeting and review the upgrade procedure and lessons learned from it.

Evaluate the outcome of upgrade with pre-defined goals.

---

End of document
