# watsonx Orchestrate Installation 5.2.0.1

---

## 1 Setting up a client workstation

### 1.1 Set the installation directory

**Note**:
<br>
You can change the directory path if necessary.

```
export WXO_INSTALL_DIR=/opt/ibm/wxo
```
### 1.2 Installing the IBM Cloud Pak for Data command-line interface

Download Version `14.2.0 Refresh 1` of the cpd-cli from the IBM/cpd-cli repository on GitHub.

#### 1.2.1 Download with wget

```
mkdir -p $WXO_INSTALL_DIR
cd $WXO_INSTALL_DIR
wget https://github.com/IBM/cpd-cli/releases/download/v14.2.0_refresh_1/cpd-cli-linux-EE-14.2.0.tgz
```
#### 1.2.2 Extract the tar file

```
tar -xvf cpd-cli-linux-EE-14.2.0.tgz
```

#### 1.2.3 Make the cpd-cli executable from any directory.

```
export PATH=$WXO_INSTALL_DIR/cpd-cli-linux-EE-14.2.0-2124:$PATH
```

Validate with the following command
```
cpd-cli version
```
The output looks like below. <br>

**Note**: The build number is `2124`

```
cpd-cli
        Version: 14.2.0
        Build Date: 2025-06-23T13:49:48
        Build Number: 2124
        CPD Release Version: 5.2.0
```

#### 1.2.4 Restart the olm-utils container
```
cpd-cli manage restart-container
```

### 1.3 Update the existing environment variables file

Update the cpd_vars.sh shell script with file content like below. <br>

***Note***:
<br>

Make sure the `VERSION`, `COMPONENTS` and `IMAGE_GROUPS` are specified properly.

```
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

#Choose the Foundation models to be usedfor watsonx Orchestrate
export IMAGE_GROUPS=ibmwxSlate30mEnglishRtrvr
```

### 1.4 Source the environment variable

```
source cpd_vars.sh
```

## 2 Applying the entitlement
Log the cpd-cli into the OpenShift cluster: 

```
${CPDM_OC_LOGIN}
```
Apply the entitlements.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-orchestrate \
--production=false
```

## 3 Create the secrets that watsonx Orchestrate uses to connect to Multicloud Object Gateway

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
If the command returns `Error from server (NotFound)`, re-run the `setup-mcg` command in the preceding step.

### 4 Install Red Hat OpenShift Serverless Knative Eventing
If OpenShift Serverless Knative Eventing has been installed and configured, you can skip this step.
If this task is not complete, see [Installing Red Hat OpenShift Serverless Knative Eventing]([https://www.ibm.com/docs/en/SSNFH6_5.2.x/hub/install/prep-cluster-openshift-ai.html](https://www.ibm.com/docs/en/SSNFH6_5.2.x/hub/install/prep-cluster-eventing.html)).


## 5 Installing the operators

Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster: 

```
${CPDM_OC_LOGIN}
```

Install the operators in the operators project for the instance.

```
cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--components=watsonx_orchestrate
```
## 6 Install watonsx Orchestrate

Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster: 

```
${CPDM_OC_LOGIN}
```

Create a file named `install-options.yml` in the work directory and specify installation options in it as follows

```
################################################################################
# watsonx Orchestrate parameters
################################################################################
#watson_orchestrate_syom_models: []
watson_orchestrate_ootb_models:
  - ibm-slate-30m-english-rtrvr
```

Apply the custom resource

```
cpd-cli manage apply-cr \
--components=watsonx_orchestrate \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--param-file=/tmp/work/install-options.yml \
--license_acceptance=true
```

Validate the upgrade

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

---

End of document
