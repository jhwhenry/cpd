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
1.1 Set up client workstation 
1.1.1 Prepare a client workstation
1.1.2 Update environment variables
1.1.3 Obtain the olm-utils-v3 available
1.1.4 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image
1.1.5 Creating a profile for upgrading the service instances
1.2 Health check OCP & CPD

Part 2: Upgrade
2.1 Upgrade CPD to 5.2.2
2.1.1 Upgrading shared cluster components
2.1.2 Preparing to upgrade the CPD instance to IBM Software Hub
2.1.3 Upgrading to IBM Software Hub
2.1.4 Upgrading the operators for the services
2.2 Upgrade watsonx Orchestrate
2.2.1 Specify the parameters in the `install-options.yml` file
2.2.2 Upgrade the watsonx Orchestrate service
2.2.3 Apply the hot fix
2.2 Upgrade watsonx.ai services
2.3 Upgrade watsonx.governance services
2.4 Upgrade watson Speech
2.5 Upgrade Voice Gateway
2.6 Upgade CPD BR service

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

#### 1.1.2 Update environment variables

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

#### 2.1.5 Applying the patches

[Patches](https://ibm-analytics.slack.com/archives/C09NV8BQJCV/p1762527063209059?thread_ts=1762434453.089799&cid=C09NV8BQJCV)

### 2.2 Upgrade watsonx Orchestrate
#### 2.2.1 Specify the parameters in the `install-options.yml` file
<br>
Specify the following options in the `install-options.yml` file in the `work` directory. Create the `install-options.yml` file if it doesn't exist in the `work` directory.

```
################################################################################
# watsonx Orchestrate parameters
################################################################################ 
watson_orchestrate_install_mode: agentic_skills_assistant
watson_orchestrate_watsonx_ai_type: true
watson_orchestrate_ootb_models:
  - llama-3-2-90b-vision-instruct
  - ibm-slate-30m-english-rtrvr
```

**Note:**

<br>
Make sure you edit or create the `install-options.yml` file in the right `work` folder.

<br>

Identify the location of the `work` folder using below command.

```
podman inspect olm-utils-play-v3 | jq -r '.[0].Mounts' |jq -r '.[] | select(.Destination == "/tmp/work") | .Source'
```

#### 2.2.2 Upgrade the watsonx Orchestrate service.
- Log in to the cluster

```
${CPDM_OC_LOGIN}
```

- Run the upgrade command

```
cpd-cli manage apply-cr \
--components=watsonx_orchestrate \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--param-file=/tmp/work/install-options.yml \
--license_acceptance=true \
--upgrade=true
```

- Validate the upgrade
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=watsonx_orchestrate
```

#### 2.2.3 Apply the hot fix
[Hot fix documentation](https://www.ibm.com/support/pages/node/7249508)


### 2.3 Upgrade the watsonx.ai services.
- Log in to the cluster

```
${CPDM_OC_LOGIN}
```

- Run the upgrade command

```
cpd-cli manage apply-cr \
--components=watsonx_ai \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--upgrade=true
```

- Validate the upgrade
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=watsonx_ai,ws,wml,ws_runtimes,ccs
```

Check the version of each custom resource by following the [Operator and operand versions](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=versions-522-october-2025)

#### 2.4 Upgrade the watsonx.governance services
- Log in to the cluster

```
${CPDM_OC_LOGIN}
```

- Run the upgrade command

```
cpd-cli manage apply-cr \
--components=watsonx_governance \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--upgrade=true
```

- Validate the upgrade
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=watsonx_governance
```

- Upgrade the service instances
```
cpd-cli service-instance upgrade \
--service-type=openpages \
--profile=${CPD_PROFILE_NAME} \
--all
```

Validate the service instance upgrade.
```
cpd-cli service-instance list \
--service-type=openpages \
--profile=${CPD_PROFILE_NAME} \
```

#### 2.5 Upgrade the watson Speech services
- Log in to the cluster

```
${CPDM_OC_LOGIN}
```

- Run the upgrade command

```
cpd-cli manage apply-cr \
--components=watson_speech \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--upgrade=true
```

- Validate the upgrade
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=watson_speech
```

- Upgrade the service instances
```
cpd-cli service-instance upgrade \
--service-type=openpages \
--profile=${CPD_PROFILE_NAME} \
--all
```

Validate the service instance upgrade.
```
cpd-cli service-instance list \
--service-type=openpages \
--profile=${CPD_PROFILE_NAME} \
```

[Cleaning up resources](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=u-upgrading-from-version-52-8#cli-upgrade__clean__title__1)

#### 2.6 Upgrade the Voice Gateway

The Voice Gateway custom resource does not include a version entry, so the Voice Gateway service is automatically upgraded when you install a newer version of the Voice Gateway operator on the cluster.

- Validate the upgrade

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=voice_gateway
```

#### 2.7 Upgrade the cpdbr service
1.Set the `OADP_OPERATOR_NS` environment variable to the project where the OADP operator is installed:
```
export OADP_OPERATOR_NS=<oadp-operator-project>
```
2.Upgrade the cpdbr-tenant component for the instance.
```
cpd-cli oadp install \
--component=cpdbr-tenant \
--cpdbr-hooks-image-prefix=${PRIVATE_REGISTRY_LOCATION} \
--cpfs-image-prefix=${PRIVATE_REGISTRY_LOCATION} \
--namespace=${OADP_OPERATOR_NS} \
--tenant-operator-namespace=${PROJECT_CPD_INST_OPERATORS} \
--skip-recipes=true \
--upgrade=true \
--log-level=debug \
--verbose
```

## Summarize and close out the upgrade

1)Schedule a wrap-up meeting and review the upgrade procedure and lessons learned from it.

2)Evaluate the outcome of upgrade with pre-defined goals.

---

End of document
