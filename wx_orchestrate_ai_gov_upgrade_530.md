# watsonx Upgrade Runbook - v.5.2.2 to 5.3.0

---

## Upgrade documentation

[Upgrading from IBM Cloud Pak for Data Version 5.2.2 to Version 5.3.0](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading-from-version-52)

## Upgrade context

From

```
OCP: OpenShift Dedicated 4.17.26 on Google Cloud
CPD: 5.2.2
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Componenets: cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway
```

To

```
OCP: OpenShift Dedicated 4.17.26 on Google Cloud 
CPD: 5.3.0
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Componenets: cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway
```

## Pre-requisites

#### 1. Backup of the cluster is done.

Backup your Cloud Pak for Data cluster before the upgrade.
For details, see [Backing up and restoring Cloud Pak for Data](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=administering-backing-up-restoring-software-hub).

**Note:**
Some services don't support the offline OADP backup. Review the backup documentation and take the dedicate approach when necessary.

#### 2. The image mirroring completed successfully

If a private container registry is in-use to host the IBM Cloud Pak for Data software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry. 
<br>
Reference: 
[Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=registry-mirroring-images-private-container)


#### 3. The permissions required for the upgrade is ready

- Openshift cluster administrator permissions
- Cloud Pak for Data administrator permissions
- Permission to access the private image registry for pushing or pull images
- Access to the Bastion node for executing the upgrade commands

#### 4. A pre-upgrade health check is made to ensure the cluster's readiness for upgrade.

- The OpenShift cluster, persistent storage and Cloud Pak for Data platform and services are in healthy status.

#### 5. Backup the Routes

```
oc get Routes -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > Routes_Bak.yaml
```

#### 6. Backup the TemporaryPatch

```
oc get TemporaryPatch -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > TemporaryPatch_Bak.yaml
```

## Table of Content

```
Part 1: Pre-upgrade
1.1 Update client workstation
1.1.1 Set up the utilities
1.1.2 Update environment variables
1.1.3 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image
1.1.4 Download CASE packages
1.1.5 Create a profile for upgrading the service instances
1.2 Health check OCP & CPD

Part 2: Upgrade
2.1 Upgrade CPD to 5.3.0
2.1.1 Upgrade shared cluster components
2.1.2 Prepare to upgrade IBM Software Hub
2.1.3 Create image pull secrets for IBM Software Hub instance
2.1.4 Upgrade IBM Software Hub
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

### 1.1 Update client workstation
#### 1.1.1 Set up the utilities

**1. Update the cpd-cli utility**

<br>

Update the cpd-cli utility following the steps in [Updating client workstations](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=52-updating-client-workstations)

<br>

After the update is done, run below commands for the confirmation:

```
cpd-cli version
```

Output like this

```
cpd-cli
	Version: 14.3.0
	Build Date: 2025-12-05T14:54:42
	Build Number: 2792
	SWH Release Version: 5.3.0
```

**2. Update the OpenShift CLI**
<br>
Check the OpenShift CLI version.

```
oc version
```

If the version doesn't match the OpenShift cluster version, update it accordingly.

**3. Install the Helm CLI**
<br>
Install Helm by following the [Helm documentation](https://www.ibm.com/links?url=https%3A%2F%2Fhelm.sh%2Fdocs%2Fintro%2Finstall%2F)

#### 1.1.2 Update environment variables
Make a copy of the environment variables script used by the existing 5.2.2 instance with the name like `cpd_vars_530.sh`. 
<br>
Update the environment variables script `cpd_vars_530.sh` as follows.

```
vi cpd_vars_530.sh
```

1.Locate the VERSION entry and update the environment variable for VERSION.

```
export VERSION=5.3.0
```

2.Locate the COMPONENTS entry and confirm the COMPONENTS entry is accurate.

```
export COMPONENTS=ibm-licensing,cpfs,cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway
```

3.Add a new section called Image pull configuration to your script and add the following environment variables

```
export IMAGE_PULL_SECRET=ibm-entitlement-key
export IMAGE_PULL_CREDENTIALS=$(echo -n "$PRIVATE_REGISTRY_PULL_USER:$PRIVATE_REGISTRY_PULL_PASSWORD" base64 -w 0)
export IMAGE_PULL_PREFIX=${PRIVATE_REGISTRY_LOCATION}
```

Save the changes. <br>

Confirm that the script does not contain any errors.

```
bash ./cpd_vars_530.sh
```

Run this command to apply cpd_vars_530.sh

```
source cpd_vars_530.sh
```

#### 1.1.3 Ensure the cpd-cli manage plug-in has the latest olm-utils image

```
cpd-cli manage restart-container
```

**Note:**
<br>Check and confirm the olm-utils-v4 container is up and running.

```
podman ps | grep olm-utils-v4
```

#### 1.1.4 Downloading CASE packages
Downloading CASE packages before running IBM Software Hub upgrade commands.
<br>

**Note:**

<br>

If the CASE packages have already been downloaded when mirroring the images, this step can be skipped.

<br>

[Downloading CASE packages](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=pruirn-downloading-case-packages-3)


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

### 2.1 Upgrade CPD to 5.3.0

#### 2.1.1 Upgrade shared cluster components

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

Confirm that the License Service pods are Running or Completed:

```
oc get pods --namespace=${PROJECT_LICENSE_SERVICE}
```

#### 2.1.2 Prepare to upgrade IBM Software Hub

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster

```
${CPDM_OC_LOGIN}
```

2.Updating the cluster-scoped resources for the platform and services

```
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cluster_resources=true
```

Change to the work directory. The default location of the work directory is `cpd-cli-workspace/olm-utils-workspace/work`.

<br>

Log in to Red Hat® OpenShift® Container Platform as a cluster administrator.

```
${OC_LOGIN}
```

Apply the cluster-scoped resources for the from the `cluster_scoped_resources.yaml` file.

```
oc apply --server-side --force-conflicts -f cluster_scoped_resources.yaml
```

Have a record of the resources that you generated.
```
mv cluster_scoped_resources.yaml ${VERSION}-${PROJECT_CPD_INST_OPERATORS}-cluster_scoped_resources.yaml
```

3.Applying your entitlements to monitor and report use against license terms
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

[Applying your entitlements](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=aye-applying-your-entitlements-without-node-pinning-3)

#### 2.1.3 Create image pull secrets for IBM Software Hub instance
Log in to OpenShift cluster.
```
${OC_LOGIN}
```

Generate the image pull credentials:

```
export IMAGE_PULL_CREDENTIALS=$(echo -n "$PRIVATE_REGISTRY_PULL_USER:$PRIVATE_REGISTRY_PULL_PASSWORD" | base64 -w 0)
```

Create a file named dockerconfig.json based on where your cluster pulls images from:

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

Create the image pull secret in the `operators` project for the instance.
```
oc create secret docker-registry ${IMAGE_PULL_SECRET} --from-file ".dockerconfigjson=dockerconfig.json" --namespace=${PROJECT_CPD_INST_OPERATORS}
```

Create the image pull secret in the `operands` project for the instance.
```
oc create secret docker-registry ${IMAGE_PULL_SECRET} --from-file ".dockerconfigjson=dockerconfig.json" --namespace=${PROJECT_CPD_INST_OPERANDS}
```

#### 2.1.4 Upgrade IBM Software Hub

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```
${CPDM_OC_LOGIN}
```

2.Upgrade the required operators and custom resources for the instance.

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

Once the above command `cpd-cli manage install-components` complete, make sure the status of the IBM Software Hub is in 'Completed' status.

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \ 
--components=cpd_platform
```

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
The parameter `watson_orchestrate_install_mode: agentic_skills_assistant` needs to be confirmed. Where should it be placed in the `install-options.yml`?

<br>
Make sure you edit or create the `install-options.yml` file in the right `work` folder. You can identify the location of the `work` folder using below command.

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
cpd-cli manage install-components \
--license_acceptance=true \
--components=watsonx_orchestrate \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--param-file=/tmp/work/install-options.yml \
--upgrade=true
```

**Note:**
<br>
During the watsonx Orchestrate reconcilation, check if the UAB enabled in the custom resource.
```
oc get wo wo --namespace="${PROJECT_CPD_INST_OPERANDS}" -o yaml | grep uab -A5 -B5
```

If UAB enabled, then disable it using below command.

```
oc patch wo wo \
  --namespace="${PROJECT_CPD_INST_OPERANDS}" \
  --type=merge \
  --patch='{"spec": {"wxolite": {"enabled": false}, "uab": {"enabled": false}}}'
```

Double confirmation to ensure the UAB disabled.

<br>

- Validate the upgrade
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=watsonx_orchestrate
```

#### 2.2.3 Apply the hot fix if any
This is a place holder.

### 2.3 Upgrade the watsonx.ai services
- Log in to the cluster

```
${CPDM_OC_LOGIN}
```

- Run the upgrade command

```
cpd-cli manage install-components \
--license_acceptance=true \
--components=watsonx_ai \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

- Validate the upgrade
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=watsonx_ai,ws,wml,ws_runtimes,ccs
```

Check the version of each custom resource by following the [Operator and operand versions](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=versions-522-october-2025)

#### 2.4 Upgrade the watsonx.governance services
- Log in to the cluster

```
${CPDM_OC_LOGIN}
```

- Run the upgrade command

```
cpd-cli manage install-components \
--license_acceptance=true \
--components=watsonx_governance \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
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
cpd-cli service-instance upgrade \
--service-type=openpages \
--profile=${CPD_PROFILE_NAME} \
--all
```

#### 2.5 Upgrade the watson Speech services
- Log in to the cluster

```
${CPDM_OC_LOGIN}
```

- Run the upgrade command

```
cpd-cli manage install-components \
--license_acceptance=true \
--components=watson_speech \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

- Validate the upgrade
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=watson_speech
```

[Cleaning up resources](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=u-upgrading-from-version-52-8#cli-upgrade__clean__title__1)

#### 2.6 Upgrade the Voice Gateway

- Log in to the cluster

```
${CPDM_OC_LOGIN}
```

- Run the upgrade command

```
cpd-cli manage install-components \
--license_acceptance=true \
--components=voice_gateway \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

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
--cpd-scheduler-namespace=${PROJECT_SCHEDULING_SERVICE} \
--skip-recipes=true \
--upgrade=true \
--log-level=debug \
--verbose
```

## Summarize and close out the upgrade
1)Prepare for applying the TemporaryPatch if needed as a post-upgrade task.

2)Schedule a wrap-up meeting and review the upgrade procedure and lessons learned from it.

3)Evaluate the outcome of upgrade with pre-defined goals.

---

End of document
