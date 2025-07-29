# watsonx Orchestrate Installation 5.2.0

---

## Setting up a client workstation

### Set the installation directory

***Note***:
<br>
You can change the directory path if needed.

```
export WXO_INSTALL_DIR=/opt/ibm/wxo
```
### Installing the IBM Cloud Pak for Data command-line interface

Download Version 14.2.0 of the cpd-cli from the IBM/cpd-cli repository on GitHub <br>

#### 1.Download with wget

```
mkdir -p $WXO_INSTALL_DIR
cd $WXO_INSTALL_DIR
wget https://github.com/IBM/cpd-cli/releases/download/v14.2.0/cpd-cli-linux-EE-14.2.0.tgz
```

#### 2.Extract the tar file

```
tar -xvf cpd-cli-linux-EE-14.2.0.tgz
```

#### 3.Make the cpd-cli executable from any directory.

```
export PATH=$WXO_INSTALL_DIR/cpd-cli-linux-EE-14.2.0-2081:$PATH
```

Validate with the following command
```
cpd-cli version
```

#### 4.Restart the olm-utils container
```
cpd-cli manage restart-container
```

### Creating an environment variables file

Create the cpd_vars.sh shell script with file content like below. <br>

***Note***:
<br>
Change the Cluster information 


```
#===============================================================================
# Cloud Pak for Data installation variables
#===============================================================================

# ------------------------------------------------------------------------------
# Cluster
# ------------------------------------------------------------------------------

export OCP_URL=<enter your Red Hat OpenShift Container Platform URL>
export OPENSHIFT_TYPE=self-managed
export IMAGE_ARCH=amd64
export OCP_USERNAME=<enter your username>
export OCP_PASSWORD=<enter your password>
# export OCP_TOKEN=sha256~JkYN46_DwXlQ4bWCOwu_rfWq_RghLsUqMCY_rkGkv4U
export SERVER_ARGUMENTS="--server=${OCP_URL}"
export LOGIN_ARGUMENTS="--username=${OCP_USERNAME} --password=${OCP_PASSWORD}"
export CPDM_OC_LOGIN="cpd-cli manage login-to-ocp ${SERVER_ARGUMENTS} ${LOGIN_ARGUMENTS}"
export OC_LOGIN="oc login ${OCP_URL} ${LOGIN_ARGUMENTS}"


# ------------------------------------------------------------------------------
# Projects
# ------------------------------------------------------------------------------

export PROJECT_CERT_MANAGER=ibm-cert-manager
export PROJECT_LICENSE_SERVICE=ibm-licensing
export PROJECT_SCHEDULING_SERVICE=ibm-cpd-scheduler
export PROJECT_IBM_EVENTS=ibm-knative-events
# export PROJECT_PRIVILEGED_MONITORING_SERVICE=<enter your privileged monitoring service project>
export PROJECT_CPD_INST_OPERATORS=cpd-operators
export PROJECT_CPD_INST_OPERANDS=cpd

# ------------------------------------------------------------------------------
# Storage
# ------------------------------------------------------------------------------

export STG_CLASS_BLOCK=ocs-storagecluster-ceph-rbd
export STG_CLASS_FILE=ocs-storagecluster-cephfs

export IBM_ENTITLEMENT_KEY=<Your IBM Entitlement Key>

# ------------------------------------------------------------------------------
# Private container registry
# ------------------------------------------------------------------------------
# Set the following variables if you mirror images to a private container registry.
#
# To export these variables, you must uncomment each command in this section.

export PRIVATE_REGISTRY_LOCATION=<enter the location of your private container registry>
export PRIVATE_REGISTRY_PUSH_USER=<enter the username of a user that can push to the registry>
export PRIVATE_REGISTRY_PUSH_PASSWORD=<enter the password of the user that can push to the registry>
export PRIVATE_REGISTRY_PULL_USER=<enter the username of a user that can pull from the registry>
export PRIVATE_REGISTRY_PULL_PASSWORD=<enter the password of the user that can pull from the registry>

# ------------------------------------------------------------------------------
# IBM Software Hub version
# ------------------------------------------------------------------------------

export VERSION=5.2.0

# ------------------------------------------------------------------------------

# Components

# ------------------------------------------------------------------------------

# Set the following variable if you want to install or upgrade multiple components at the same time.

#

# To export the variable, you must uncomment the command.

export COMPONENTS=ibm-licensing,scheduler,cpfs,cpd_platform,watsonx_orchestrate

```

### Source the environment variable

```
source cpd_vars.sh
```

## Change cluster settings
Log the cpd-cli into the OpenShift cluster: 

```
${CPDM_OC_LOGIN}
```

### Configuring image content source policy

Run the following command to get the list of image content source policy resources:

```
oc get imageContentSourcePolicy -o yaml > icsp.yaml
oc get ImageDigestMirrorSet -o yaml > idms.yaml
```

If the command returns `No resources found`, then run below commands:
```
cpd-cli manage apply-icsp \
--registry=${PRIVATE_REGISTRY_LOCATION}
```

Get the status of the nodes:
```
oc get nodes
```

Wait until all the nodes are Ready before you proceed to the next step. 

### Updating the global image pull secret
Log the cpd-cli into the OpenShift cluster: 

```
${CPDM_OC_LOGIN}
```

This step may take minutes to complete.

```
cpd-cli manage add-cred-to-global-pull-secret \
--registry=${PRIVATE_REGISTRY_LOCATION} \
--registry_pull_user=${PRIVATE_REGISTRY_PULL_USER} \
--registry_pull_password=${PRIVATE_REGISTRY_PULL_PASSWORD}
```

Get the status of the nodes:

```
oc get nodes
```

Wait until all the nodes are Ready before you proceed to the next step. 

## Manually creating projects

```
#Shared cluster components
oc new-project ${PROJECT_CERT_MANAGER}
oc new-project ${PROJECT_LICENSE_SERVICE}

#Instance operators and operands
oc new-project ${PROJECT_CPD_INST_OPERATORS}
oc new-project ${PROJECT_CPD_INST_OPERANDS}
```

## Installing shared cluster components
Log the cpd-cli into the OpenShift cluster: 

```
${CPDM_OC_LOGIN}
```

Install the Certificate manager and the License Service:
```
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```

Wait for the cpd-cli to return the following message before proceeding to the next step:

```
[SUCCESS]... The apply-cluster-components command ran successfully.
```

## Applying the required permissions to the projects (namespaces) for CPD instance

Applying the required permissions by running the authorize-instance-topology command

```
cpd-cli manage authorize-instance-topology \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

## Installing Software Hub

Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster: 

```
${CPDM_OC_LOGIN}
```

Install the required components for an instance of IBM Software Hub

```
cpd-cli manage setup-instance \
--release=${VERSION} \
--license_acceptance=true \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--run_storage_tests=false
```
Confirm that the status of the operands is Completed

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Get the URL and default credentials of the web client: 
```
cpd-cli manage get-cpd-instance-details \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--get_admin_initial_credentials=true
```

## Applying your entitlements without node pinning
Log the cpd-cli into the OpenShift cluster: 

```
${CPDM_OC_LOGIN}
```
Apply the entitlements.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cpd-enterprise \
--production=false
```

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-orchestrate \
--production=false
```

## Install the pre-requisites
### Install the GPU operators
If the GPU operators has been installed and configured, you can skip this step.
If this task is not complete, see [Installing operators for services that require GPUs](https://www.ibm.com/docs/en/SSNFH6_5.2.x/hub/install/prep-cluster-node-gpu.html).

### Install the OpenShift AI
If OpenShift AI has been installed and configured, you can skip this step.
If this task is not complete, see [Installing Red Hat OpenShift AI](https://www.ibm.com/docs/en/SSNFH6_5.2.x/hub/install/prep-cluster-openshift-ai.html).

### Install and configure Multicloud Object Gateway

#### Install Multicloud Object Gateway
If Multicloud Object Gateway has been installed and configured, you can skip this step.
If this task is not complete, see [Installing Multicloud Object Gateway](https://www.ibm.com/docs/en/SSNFH6_5.2.x/hub/install/setup-cluster-storage-mcg.html).

#### Create the secrets that watsonx Orchestrate uses to connect to Multicloud Object Gateway

- 1.Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster
```
${CPDM_OC_LOGIN}
```
- 2.Get the names of the secrets that contain the NooBaa account credentials and certificate
```
oc get secrets --namespace=openshift-storage
```
- 3.Set the following environment variables based on the names of the secrets on your cluster.
<br>

Set `NOOBAA_ACCOUNT_CREDENTIALS_SECRET` to the name of the secret that contains the NooBaa account credentials. The default name is `noobaa-admin`.
If you created multiple backing stores, ensure that you specify the credentials for the appropriate backing store.

```
export NOOBAA_ACCOUNT_CREDENTIALS_SECRET=<secret-name>
```

Set `NOOBAA_ACCOUNT_CERTIFICATE_SECRET` to the name of the secret that contains the NooBaa account certificate. The default name is `noobaa-s3-serving-cert`.

```
export NOOBAA_ACCOUNT_CERTIFICATE_SECRET=<secret-name>
```

- 4.Run the setup-mcg command to create the secrets for watsonx Assistant, which is automatically installed with watsonx Orchestrate

```
cpd-cli manage setup-mcg \
--components=watson_assistant \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--noobaa_account_secret=${NOOBAA_ACCOUNT_CREDENTIALS_SECRET} \
--noobaa_cert_secret=${NOOBAA_ACCOUNT_CERTIFICATE_SECRET}
```
Wait for the cpd-cli to return the following message before proceeding to the next step:

```
[SUCCESS] ... setup-mcg completed successfully.
```

Confirm that the watson-assistant secrets were created in the operands project for the instance:

```
oc get secrets --namespace=${PROJECT_CPD_INST_OPERANDS} \
noobaa-account-watson-assistant \
noobaa-cert-watson-assistant \
noobaa-uri-watson-assistant
```

If the command returns `Error from server (NotFound)`, re-run the `setup-mcg` command in the preceding step.

- 5.Run the `setup-mcg` command to create the secrets for watsonx Orchestrate
```
cpd-cli manage setup-mcg \
--components=watsonx_orchestrate \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--noobaa_account_secret=${NOOBAA_ACCOUNT_CREDENTIALS_SECRET} \
--noobaa_cert_secret=${NOOBAA_ACCOUNT_CERTIFICATE_SECRET}
```

Wait for the cpd-cli to return the following message before proceeding to the next step:
```
[SUCCESS] ... setup-mcg completed successfully.
```

Confirm that the secrets were created in the operands project for the instance:
```
oc get secrets --namespace=${PROJECT_CPD_INST_OPERANDS} \
noobaa-account-watsonx-orchestrate \
noobaa-cert-watsonx-orchestrate \
noobaa-uri-watsonx-orchestrate
```
If the command returns Error from server (NotFound), re-run the `setup-mcg` command in the preceding step.

### Install Red Hat OpenShift Serverless Knative Eventing
If OpenShift Serverless Knative Eventing has been installed and configured, you can skip this step.
If this task is not complete, see [Installing Red Hat OpenShift Serverless Knative Eventing]([https://www.ibm.com/docs/en/SSNFH6_5.2.x/hub/install/prep-cluster-openshift-ai.html](https://www.ibm.com/docs/en/SSNFH6_5.2.x/hub/install/prep-cluster-eventing.html)).


## Installing the operators

Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster: 

```
${CPDM_OC_LOGIN}
```

Install the operators in the operators project for the instance.

```
cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--components=${COMPONENTS}
```
## Install watonsx Orchestrate

Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster: 

```
${CPDM_OC_LOGIN}
```

Create a file named `install-options.yml` in the work directory and specify installation options in it as follows

```
################################################################################
# IBM Knowledge Catalog parameters
################################################################################
custom_spec:
  wkc:
    enableDataQuality: True
    enableSemanticAutomation: True
    enableSemanticEnrichment: True
    enableModelsOn: 'cpu'
```

Apply the custom resource

```
cpd-cli manage apply-cr \
--components=${COMPONENTS} \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true
```

Validate the upgrade

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

## Apply the Day 0 patch

https://www.ibm.com/support/pages/node/7236425

---

End of document
