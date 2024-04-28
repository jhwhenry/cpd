# CPD Upgrade Runbook - v.4.8.2 to 4.8.5

---
## Upgrade documentation
[Upgrading from IBM Cloud Pak for Data Version 4.8.x to a later 4.8 refresh](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=upgrading-from-cloud-pak-data-version-48)

## Upgrade context

From

```
OCP: 4.12
CPD: 4.8.2
Storage: Storage Fusion 2.7.2
Componenets: cpfs,cpd_platform,ws,ws_runtimes,wml,datastage_ent,datastage_ent_plus,dmc,wkc,analyticsengine,openscale,db2wh,match360,mantaflow
```

To

```
OCP: 4.12
CPD: 4.8.5
Componenets: cpfs,cpd_platform,ws,ws_runtimes,wml,datastage_ent,datastage_ent_plus,dmc,wkc,analyticsengine,openscale,db2wh,match360,mantaflow
```

## Table of Content

```
Part 1: Pre-upgrade
1.1 Collect information and review upgrade runbook
1.1.1 Prepare cpd_vars.sh
1.1.2 Review the upgrade runbook
1.1.3 Backup before upgrade
1.1.4 If you installed hotfixes, uninstall all hotfixes
1.1.5 If use SAML SSO, export SSO configuration
1.2 Set up client workstation 
1.2.1 Prepare a client workstation
1.2.2 Update cpd_vars.sh for the upgrade to Version 4.8.5
1.2.3 Make olm-utils available in bastion
1.2.4 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image
1.2.5 Creating a profile for upgrading the service instances
1.2.6 Download CASE files
1.3 Health check OCP & CPD

Part 2: Upgrade
2.1 Upgrade CPD to 4.8.5
2.1.1 Upgrading shared cluster components
2.1.2 Preparing to upgrade an CPD instance
2.1.3 Upgrade foundation service
2.1.4 Upgrade CPD platform
2.2 Upgrade CPD services

Part 3: Post-upgrade
3.1 Validate CPD & CPD services
3.2 Enabling users to upload JDBC drivers
3.3 Enable Relationship Explorer feature
3.4 Configuring single sign-on
3.5 Summarize and close out the upgrade
```

## Part 1: Pre-upgrade
### 1.1 Collect information and review upgrade runbook

#### 1.1.1 Review the upgrade runbook

Review upgrade runbook

#### 1.1.2 Backup before upgrade
Note: Create a folder for 4.8.2 and maintain below created copies in that folder. <br>

Capture data for the CPD 4.8.2 instance. No sensitive information is collected. Only the operational state of the Kubernetes artifacts is collected.The output of the command is stored in a file named collect-state.tar.gz in the cpd-cli-workspace/olm-utils-workspace/work directory.

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

oc get MantaFlow mantaflow-wkc -o yaml > mantaflow-cr.yaml

oc get DataStage datastage -o yaml > datastage-cr.yaml

oc get MasterDataManagement mdm-cr -o yaml > mdm-cr.yaml 

```

#### 1.1.3 if you installed hotfixes, uninstall all hotfixes
Edit Zensevice, CCS, WKC, AE custom resources and remove all hotfix references.

#### 1.1.4 If use SAML SSO, export SSO configuration

If you use SAML SSO, export your SSO configuration. You will need to reapply your SAML SSO configuration after you upgrade to Version 4.8. Skip this step if you use the IBM Cloud Pak foundational services Identity Management Service

```
oc cp -n=${PROJECT_CPD_INST_OPERANDS} $(oc get pods -l component=usermgmt -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath='{.items[0].metadata.name}'):/user-home/_global_/config/saml ./samlConfig.json
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

To upgrade from Cloud Pak for Data Version 4.8.2 to Version 4.8.5, you must update the environment variable for VERSION. 

```
export VERSION=4.8.5
#export OLM_UTILS_IMAGE=${PRIVATE_REGISTRY_LOCATION}/cpd/olm-utils-v2:latest
```
Save the changes. <br>

Run this command to apply cpd_vars_485.sh
```
source cpd_vars_485.sh
```

#### 1.2.3 Make olm-utils available

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

For details please refer to 4.8 doc (https://www.ibm.com/docs/en/SSQNUZ_4.8.x/cpd/upgrade/v48-setup-client.html)

#### 1.2.4 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image
```
podman stop olm-utils-play-v2
cpd-cli manage restart-container
```
#### 1.2.5 Creating a profile for upgrading the service instances
Create a profile on the workstation from which you will upgrade the service instances. <br>

The profile must be associated with a Cloud Pak for Data user who has either the following permissions:

- Create service instances (can_provision)
- Manage service instances (manage_service_instances)

Click this link and follow these steps for getting it done. https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=cli-creating-cpd-profile#taskcpd-profile-mgmt__steps__1

#### 1.2.6 Download CASE files

1. Go to the client workstation with internet

Log into IBM registry and list images

```
cpd-cli manage login-entitled-registry \
${IBM_ENTITLEMENT_KEY}
```

Download case package from either of the following locations:<br>
Option 1 - GitHub (github.com)
```
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION}
```

Option 2 - IBM Cloud Pak Open Container Initiative (icr.io)
```
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--from_oci=true
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
cpd-cli manage get-cr-status -n ${PROJECT_CPD_INST_OPERANDS}
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

#### 2.1.1 Upgrading shared cluster components
1. Find out which project the License Service installed. Assuming it installed in ${PROJECT_CS_CONTROL}. If not, upgrade command needs to change.
```
oc get deployment -A |  grep ibm-licensing-operator
```   
2.	Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```
3. Upgrade the Certificate manager and License Service

The License Service will remain in the cs-control ${PROJECT_CS_CONTROL} project.
```
cpd-cli manage apply-cluster-components --release=${VERSION} --license_acceptance=true --cert_manager_ns=${PROJECT_CERT_MANAGER} --licensing_ns=${PROJECT_CS_CONTROL}

```
- Confirm that the Certificate manager pods in the ${PROJECT_CERT_MANAGER} project are Running:
```
oc get pod -n ${PROJECT_CERT_MANAGER}
```

- Confirm that the License Service pods in the ${PROJECT_CS_CONTROL} project are Running:
```
oc get pods --namespace=${PROJECT_CS_CONTROL}
``` 

#### 2.1.2 Preparing to upgrade Cloud Pak for Data instance
1.	Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster:
```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```
2.	Apply the required permissions to the projects.
<br>
Preview

```
cpd-cli manage authorize-instance-topology \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}  --preview=true
```
<br>Apply
```
cpd-cli manage authorize-instance-topology \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

#### 2.1.3 Upgrade foundation service to 4.8.5
1.	Run the cpd-cli manage login-to-ocp command to log in to the cluster.
```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```
2.	Upgrade IBM Cloud Pak foundational services and create the required ConfigMap.
<br>Preview
```
cpd-cli manage setup-instance-topology --release=${VERSION} --cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --license_acceptance=true --block_storage_class=${STG_CLASS_BLOCK} --preview=true
```
<br>Apply
```
cpd-cli manage setup-instance-topology --release=${VERSION} --cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --license_acceptance=true --block_storage_class=${STG_CLASS_BLOCK}
```
- Confirm common-service, namespace-scope, opencloud and odlm operator running in the ${PROJECT_CPD_INST_OPERATORS} namespace
```
oc get pod -n ${PROJECT_CPD_INST_OPERATORS}
```

#### 2.1.4 Upgrade CPD platform (control plane) to 4.8.5
1.	Run the cpd-cli manage login-to-ocp command to log in to the cluster.
```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```

2.	Upgrade the operators in the operators project for CPD instance. First run the oc command with the --preview=true option.
<br>Preview
```
cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--upgrade=true \
--preview=true
```
<br>Apply
```
cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--upgrade=true
```

In another terminal, keep running below command and monitoring "InstallPlan" to find which one need manual approval.
```
oc get ip -n ibm-cpd-operators -o=jsonpath='{.items[?(@.spec.approved==false)].metadata.name}'
```
Approve the upgrade request and run below command as soon as we find it.
```
oc patch installplan $(oc get ip -n ibm-cpd-operators -o=jsonpath='{.items[?(@.spec.approved==false)].metadata.name}') -n ibm-cpd-operators --type merge --patch '{"spec":{"approved":true}}'
```

3.	Confirm that the operator pods are Running or Copmpleted:
NOTE: You will find CPD operators and catalog sources running in the ${PROJECT_CPD_INST_OPERATORS} namespace
```
oc get pods --namespace=${PROJECT_CPD_INST_OPERATORS}
```

4.	Upgrade the operands in the operands project for CPD instance.
<br>Run the cpd-cli manage login-to-ocp command to log in to the cluster.
```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```
<br>First run the oc command with the --preview=true option.
```
cpd-cli manage apply-cr --release=${VERSION} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=cpd_platform --block_storage_class=${STG_CLASS_BLOCK} --file_storage_class=${STG_CLASS_FILE} --license_acceptance=true --upgrade=true --preview=true

cpd-cli manage apply-cr --release=${VERSION} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=cpd_platform --block_storage_class=${STG_CLASS_BLOCK} --file_storage_class=${STG_CLASS_FILE} --license_acceptance=true --upgrade=true

oc logs -f cpd-platform-operator-manager-XXXX-XXXX -n ${PROJECT_CPD_INST_OPERATORS}
```

5.	Confirm that the status of the operands is Completed:
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```
NOTE: cpd_platform has been upgraded to 4.8.5

### 2.2 Upgrade CPD services to 4.8.5
#### 2.2.1 Upgrade IBM Knowledge Catalog service

##### 1. For custom installation, check the previous install-options.yaml or wkc-cr yaml, make sure to keep original custom settings
```
vim cpd-cli-workspace/olm-utils-workspace/work/install-options.yml

################################################################################
# IBM Knowledge Catalog parameters
################################################################################
custom_spec:
  wkc:
#    enableKnowledgeGraph: False
#    enableDataQuality: False
```
##### 2.Upgrade WKC instance with default or custom installation

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

##### Custom upgrade with installation options
```
cpd-cli manage apply-cr --components=${COMPONENTS} --release=${VERSION} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --block_storage_class=${STG_CLASS_BLOCK} --file_storage_class=${STG_CLASS_FILE} --param-file=/tmp/work/install-options.yml --license_acceptance=true --upgrade=true
```

##### Default upgrade without installation options
```
cpd-cli manage apply-cr --components=${COMPONENTS} --release=${VERSION} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --block_storage_class=${STG_CLASS_BLOCK} --file_storage_class=${STG_CLASS_FILE} --license_acceptance=true --upgrade=true
```
##### Validate the upgrade
```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

##### Run the bulk sync utility before start using Global Search indexed data for relationships
Follow the step in [Bulk sync relationships for global search (IBM Knowledge Catalog)](https://www.ibm.com/docs/en/SSQNUZ_4.8.x/wsj/admin/admin-bulk-sync.html)

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
Run the command for upgrade MANTA service.

```
cpd-cli manage apply-cr --components=${COMPONENTS} --release=${VERSION} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --block_storage_class=${STG_CLASS_BLOCK} --file_storage_class=${STG_CLASS_FILE} --license_acceptance=true --upgrade=true
```

Validating the upgrade.
```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --components=${COMPONENTS}
```

#### 2.2.3 Upgrade Analytics Engine service
##### 2.2.3.1 Upgrade the service

Check the Analytics Engine service version and status. 
```
export COMPONENTS=analyticsengine

cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --components=${COMPONENTS}
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
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --components=${COMPONENTS}
```

##### 2.2.3.2 Upgrade the service instances
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

cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --components=${COMPONENTS}
```

If the DataStage Enterprise service version is **not 4.8.5**, then run below commands for the upgrade. <br>

```
cpd-cli manage apply-cr --components=${COMPONENTS} --release=${VERSION} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --block_storage_class=${STG_CLASS_BLOCK} --file_storage_class=${STG_CLASS_FILE} --license_acceptance=true --upgrade=true
```

Validate the upgrade status.
```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --components=${COMPONENTS}
```

#### 2.2.4 Upgrade DataStage Enterprise plus
```
export COMPONENTS=datastage_ent_plus

cpd-cli manage apply-cr --components=${COMPONENTS} --release=${VERSION} --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --block_storage_class=${STG_CLASS_BLOCK} --file_storage_class=${STG_CLASS_FILE} --license_acceptance=true --upgrade=true
```

Validate the upgrade status.
```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --components=${COMPONENTS}
```
#### 2.2.5 Upgrade Match 360
```
export COMPONENTS=match360
```

Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```

Check if the Match 360 service was installed with the custom install options. <br>

- If it's custom installation, check the previous install-options.yaml or mdm-cr yaml, make sure to keep original custom settings.
```
vim cpd-cli-workspace/olm-utils-workspace/work/install-options.yml
```

Then run this upgrade command.
```
cpd-cli manage apply-cr \
--components=${COMPONENTS} \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--param-file=/tmp/work/install-options.yml \
--license_acceptance=true \
--upgrade=true
```

- If it's **NOT** custom installation, then run this upgrade command.

```
cpd-cli manage apply-cr \
--components=${COMPONENTS} \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true \
--upgrade=true
```

Validate the service upgrade status.
```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --components=${COMPONENTS}
```

#### 2.2.5 Upgrade Watson Studio, Watson Studio Runtimes and Watson Machine Learning 
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
cpd-cli manage apply-cr --components=${COMPONENTS} --release=${VERSION} --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --block_storage_class=${STG_CLASS_BLOCK} --file_storage_class=${STG_CLASS_FILE} --license_acceptance=true --upgrade=true
```
Validate the service upgrade status.
```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INSTANCE} --components=${COMPONENTS}
```
#### 2.2.6 Upgrade Db2 Warehouse and Data Management Console 
```
# 1.Upgrade the service
export COMPONENTS=db2wh,dmc

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

### 3.1 Validate CPD & CPD services

Log into CPD web UI with admin and check out each services, including provision instance and functions of each service

### 3.2 Enabling users to upload JDBC drivers
#### 3.2.1 Set the wdp_connect_connection_disable_jar_tab parameter to false
```
oc patch ccs ccs-cr \
--namespace=${PROJECT_CPD_INST_OPERANDS} \
--type=merge \
--patch '{"spec": {"wdp_connect_connection_disable_jar_tab": "false"}}'
```

#### 3.2.2 Wait for the common core services status to be Completed
```
oc get ccs ccs-cr --namespace=${PROJECT_CPD_INST_OPERANDS}
```

### 3.3 Enable Relationship Explorer feature
[Enable Relationship Explorer feature](https://github.com/sanjitc/Cloud-Pak-for-Data/blob/main/Upgrade/CPD%204.6%20to%204.8/Enabling_Relationship_Explorer_480%20-%20disclaimer%200208.pdf)

### 3.4 Configuring single sign-on
If post upgrade login using SAML doesn't work, then follow This instruction. You need to use the "/user-home/_global_/config/saml/samlConfig.json" file that you save at the beginning of upgrade.

https://www.ibm.com/docs/en/cloud-paks/cp-data/4.8.x?topic=environment-configuring-sso

### 3.5 Summarize and close out the upgrade

Schedule a wrap-up meeting and review the upgrade procedure and lessons learned from it.

Evaluate the outcome of upgrade with pre-defined goals.

Close out the upgrade git issue.

---

End of document
