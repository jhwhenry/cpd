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
Componenets: cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway,db2oltp,cognos_analytics
```

To

```
OCP: OpenShift Dedicated 4.17.26 on Google Cloud 
CPD: 5.3.0
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Componenets: cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway,db2oltp,cognos_analytics
```

## Pre-requisites

#### 1. Backup of the cluster is done.

Backup your Cloud Pak for Data cluster before the upgrade.
For details, see [Backing up and restoring Cloud Pak for Data](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=administering-backing-up-restoring-software-hub).

**Note:**
Some services don't support the offline OADP backup. Review the backup documentation and take the dedicate approach when necessary.

#### 2. The image mirroring completed successfully

If a private container registry is in-use to host the IBM Cloud Pak for Data software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry.
`<br>`
Reference:
[Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-1)

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
`<br>`
Check the OpenShift CLI version.

```
oc version
```

If the version doesn't match the OpenShift cluster version, update it accordingly.

**3. Install the Helm CLI**
`<br>`
Install Helm by following the [Helm documentation](https://www.ibm.com/links?url=https%3A%2F%2Fhelm.sh%2Fdocs%2Fintro%2Finstall%2F)

#### 1.1.2 Update environment variables

Make a copy of the environment variables script used by the existing 5.2.2 instance with the name like `cpd_vars_530.sh`.
`<br>`
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
export IMAGE_PULL_CREDENTIALS=$(echo -n "$PRIVATE_REGISTRY_PULL_USER:$PRIVATE_REGISTRY_PULL_PASSWORD" | base64 -w 0)
export IMAGE_PULL_PREFIX=${PRIVATE_REGISTRY_LOCATION}
```

Save the changes. `<br>`

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
`<br>`Check and confirm the olm-utils-v4 container is up and running.

```
podman ps | grep olm-utils-v4
```

#### 1.1.4 Downloading CASE packages

Downloading CASE packages before running IBM Software Hub upgrade commands.
`<br>`

**Note:**

<br>

If the CASE packages have already been downloaded when mirroring the images, this step can be skipped.

<br>

[Downloading CASE packages](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-downloading-case-packages-1)

#### 1.1.5 Creating a profile for upgrading the service instances

Create a profile on the workstation from which you will upgrade the service instances.

The profile must be associated with a Cloud Pak for Data user who has either the following permissions:

- Create service instances (can_provision)
- Manage service instances (manage_service_instances)

Click this link and follow these steps for getting it done.

[Creating a profile to use the cpd-cli management commands](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=cli-creating-cpd-profile)

### 1.2 Health check OCP & CPD

Check and make sure the cluster operators, nodes, and machine configure pool are in healthy status.
`<br>`
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
`<br>`

```
oc get deployment -A |  grep ibm-licensing-operator
```

Make sure the project returned by the command matches the environment variable PROJECT_LICENSE_SERVICE in your environment variables script `cpd_vars_522.sh`.
`<br>`
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

**Production enironment**

Apply the IBM Cloud Pak for Data Enterprise Edition for the non-production environment.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cpd-enterprise \
--production=true
```

Apply the watsonx.ai license for the non-production environment.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-ai \
--production=true
```

Apply the watsonx.governance license for the non-production environment.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-gov-mm \
--production=true
```

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-gov-rc \
--production=true
```

Apply the watsonx Orchestrate license for the non-production environment.

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-orchestrate \
--production=true
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

Apply the Cognos Analytics license.

```
cpd-cli manage apply-entitlement
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
--entitlement=cognos-analytics
```

Reference: [Applying your entitlements](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=aye-applying-your-entitlements-without-node-pinning-3)

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

#### 2.2.1 Specify the parameters in the `override.yaml` file (Pending Babu's confirmation, the below is not valid)

Specify the following options in the `override.yaml` file in the `work` directory. Create the `override.yaml` file if it doesn't exist in the `work` directory.

```
################################################################################
# watsonx Orchestrate parameters
################################################################################ 
watsonxOrchestrate:
  installMode: agentic_assistant
  wxolite:
    enabled: false
  uab:
    enabled: false
  watsonxAI:
    watsonxaiifm: true
```

**Note:**
`<br>`
Make sure you edit or create the `override.yaml` file in the right `work` folder. You can identify the location of the `work` folder using below command.

```
podman inspect olm-utils-play-v4 | jq -r '.[0].Mounts' |jq -r '.[] | select(.Destination == "/tmp/work") | .Source'
```

##### Backup the Orchestrate Assistant Postgres

Identify the backup cronjob:

```
export STORE_CRONJOB_POD=$(oc get pods -l component=store-cronjob --no-headers | awk 'NR==1{print $1}')
echo $STORE_CRONJOB_POD
```

Create a Debug pod and check for the latest "store.dump" file.

```
oc debug ${STORE_CRONJOB_POD}


ls -l /store-backups/
```

Copy the name of the lastest backup file.

Open up a duplicate terminal while the debug pod is still live and copy the backup out onto your bastion.

```
export STORE_CRONJOB_DEBUG_POD=$(oc get pods | grep debug | awk '{print $1}')
echo ${STORE_CRONJOB_DEBUG_POD}

export STORE_DUMP_FILE=store.dump_<REPLACE WITH LATEST FILE TIME>
echo ${STORE_DUMP_FILE}

oc cp ${STORE_CRONJOB_DEBUG_POD}:/store-backups/${STORE_DUMP_FILE} ${STORE_DUMP_FILE}
```

Back up the secret for this database.

```
oc get secret wo-wa-auth-encryption -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > wo-wa-auth-encryption-backup.yaml
```

#### 2.2.2 Upgrade the watsonx Orchestrate service.

- Log in to the cluster

```
${CPDM_OC_LOGIN}
```

Reference (Steps Below): [https://github.ibm.com/watson-engagement-advisor/wea-backlog/issues/70312#issuecomment-158683395](https://github.ibm.com/watson-engagement-advisor/wea-backlog/issues/70312#issuecomment-158683395)

- Preview the helm command

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
  --upgrade=true \
   --preview=true
  ```
- Grab the helm command from the preview above

  ```
  cat cpd-cli-workspace/olm-utils-workspace/work/preview.sh  | grep watsonx-orchestrate-migration
  ```
- Run the helm comand that is outputted

  ```
  #Example helm upgrade --install --namespace cpd watsonx-orchestrate /root/cpd-cli-workspace/olm-utils-workspace/work/offline/5.3.0/.ibm-pak/data/cases/ibm-watson-assistant/5.10.0/charts/watsonx-orchestrate-migration-0.0.0.tgz --take-ownership --debug -f /root/cpd-cli-workspace/olm-utils-workspace/work/olm-utils-ansible-log/override_file_1765899129.3742213.yaml 
  ```
- Verify that helm label is added

  ```
  oc get wo wo -o jsonpath='{.metadata.labels.app\.kubernetes\.io/managed-by}'
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
  --param-file=/tmp/work/override.yml \
  --upgrade=true
```

**Note:** During the watsonx Orchestrate reconcilation, check if the UAB enabled in the custom resource.

```
oc get wo wo --namespace="${PROJECT_CPD_INST_OPERANDS}" -o yaml 
```

If UAB enabled, then disable it using below command.

```
oc patch wo wo \
 --namespace="${PROJECT_CPD_INST_OPERANDS}" \
 --type='merge' \
 -p '{"spec":{"uab":{"enabled":false}}}'
```

If the abInstance is not showing "wo-wa-assistantbuilder".

```
oc patch wo wo \
 --namespace="${PROJECT_CPD_INST_OPERANDS}" \
 --type='merge' \
 -p '{"spec":{"watsonAssistants":{"abInstance":{"config":{"name":"wo-wa-assistantbuilder"}}}}}'
```

If the if IFM is disabled.

```
oc patch wo wo --namespace="${PROJECT_CPD_INST_OPERANDS}" --type='merge' -p '{"spec":{"watsonAssistants":{"config":{"configOverrides":{"enabled_components":{"store":{"ifm":true}},"ifm":{}},"watsonx_enabled":true}}}}'

```

- Remove duplicate Postgres:

Note: 1.25.3 will not be removed during upgrade. There will be two Postgres operators (1.25.3 & 1.25.4). We need to remove and recreate some artifacts for the new Postgres operator.  This will also remove some secrets as well as service account for postgres.

* Back up and Delete Subsctiption and CSV for Old Postgres operator

```
oc get csv cloud-native-postgresql.v1.25.3 -n ${PROJECT_CPD_INST_OPERATORS} -oyaml > cloudnative1253_csv_backup.yaml
oc get subs cloud-native-postgresql-stable-v1.25-cloud-native-postgresql-catalog-cpd-operators   -n ${PROJECT_CPD_INST_OPERATORS} -oyaml > cloudnative_back.yaml

~~~~~~~~~~~~~~~~~~~~~~~~~~

oc delete csv cloud-native-postgresql.v1.25.3 -n ${PROJECT_CPD_INST_OPERATORS} 
oc delete subs cloud-native-postgresql-stable-v1.25-cloud-native-postgresql-catalog-cpd-operators   -n ${PROJECT_CPD_INST_OPERATORS} 
```

- Create the helm manifest yamls.

  ```
  helm get manifest postgresql -n ${PROJECT_CPD_INST_OPERANDS} > postgres_530_helmdeploy.yaml

  oc apply -f postgres_530_helmdeploy.yaml
  ```

##### **Patches to be applied during once Postgres + Redis + IFM are reconciled to 5.3.0**

**Patch 1 - Apply Day 0 Patch:**

Added as hyperlink as this critical patch can shift without notice.

- [watsonx Orchestrate 5.3.0 hot fix](https://www.ibm.com/support/pages/applying-watsonx-orchestrate-530-hotfix-hotfix-0)

**Patch 2 -Apply Custom Hotfix (Resolve creating archer conversation controller deployment):**

Added as hyperlink as this critical patch can shift without notice.

* [https://github.ibm.com/watson-engagement-advisor/wo-cpd-support/blob/main/wxo-support-docs/hotfix/5.3.0/5.3.0-hotfix0-with-llama-fix.md]([https://github.ibm.com/watson-engagement-advisor/wo-cpd-support/blob/main/wxo-support-docs/hotfix/5.3.0/5.3.0-hotfix0-with-llama-fix.md]())

 **Patch 3 -** **Additional RSI step to Hotfix to help bootstrap reconcile:**

```
mkdir cpd-cli-workspace/olm-utils-workspace/work/rsi

Create a file skill-seq.json with the content below in the rsi directory.

[{"op":"replace","path":"/spec/containers/0/resources/limits/memory","value":"3Gi"}]


cpd-cli manage create-rsi-patch --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --patch_name=skill-seq-resource-limit --patch_type=rsi_pod_spec --patch_spec=/tmp/work/rsi/skill-seq.json --spec_format=json --include_labels=wo.watsonx.ibm.com/component:wo-skill-sequencing --state=active

oc delete job wo-watson-orchestrate-bootstrap-job

```

- Validate the upgrade

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=watsonx_orchestrate
```


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

#### 2.5 Upgrade the Watson Speech services

##### Increase resources for MCG (Note: Already performed on Prod)

```
oc patch -n openshift-storage storageclusters.ocs.openshift.io ocs-storagecluster --type merge --patch '{"spec": {"resources": {"noobaa-agent": {"limits": {"memory": "8Gi"},"requests": {"memory": "8Gi"}},"noobaa-core": {"limits": {"memory":"8Gi"},"requests": {"memory": "8Gi"}},"noobaa-db": {"limits": {"memory": "8Gi"},"requests": {"memory": "8Gi"}},"noobaa-endpoint": {"limits": {"memory": "8Gi"},"requests": {"memory": "8Gi"}}}}}'

oc patch -n openshift-storage backingstore noobaa-default-backing-store --type=merge --patch='{"spec":{"pvPool":{"numVolumes": 1, "resources":{"limits":{"memory": "8Gi"}}}}}'
```

For StorageCluster ocs-storagecluster, add cpu: "3" for all resources as both requests and limits, add maxCount 8 and minCount 1 as multiCloudGateway.endpoints:

```yaml
apiVersion: ocs.openshift.io/v1
kind: StorageCluster
......
spec:
  ........
  multiCloudGateway:
    dbStorageClassName: ssd-csi
    disableLoadBalancerService: true
    endpoints:
      maxCount: 8
      minCount: 1
    reconcileStrategy: standalone
  resourceProfile: balanced
  resources:
    noobaa-agent:
      limits:
        cpu: "4"
        memory: 8Gi
      requests:
        cpu: "4"
        memory: 8Gi
    noobaa-core:
      limits:
        cpu: "4"
        memory: 8Gi
      requests:
        cpu: "4"
        memory: 8Gi
    noobaa-db:
      limits:
        cpu: "4"
        memory: 8Gi
      requests:
        cpu: "4"
        memory: 8Gi
    noobaa-endpoint:
      limits:
        cpu: "4"
        memory: 8Gi
      requests:
        cpu: "4"
        memory: 8Gi

......
```

For BackingStore noobaa-default-backing-store, set numVolumes to 4 and storage to 100Gi

```yaml
apiVersion: noobaa.io/v1alpha1
kind: BackingStore
......
spec:
  pvPool:
    numVolumes: 4
    resources:
      limits:
        memory: 8Gi
      requests:
        storage: 100Gi
    secret: {}
    storageClass: ssd-csi
  type: pv-pool

......
```

For NooBaa noobaa, add max_connections 2400 for dbConf

```yaml
apiVersion: noobaa.io/v1alpha1
kind: NooBaa
...
spec:
  ......
  coreResources:
    limits:
      cpu: "4"
      memory: 8Gi
    requests:
      cpu: "4"
      memory: 8Gi
  dbConf: |
    max_connections 2400
  dbImage: registry.redhat.io/rhel9/postgresql-15@sha256:4d707fc04f13c271b455f7b56c1fda9e232a62214ffc6213c02e41177dd4a13f
  dbResources:

......
```

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

Reference: [Cleaning up resources](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=u-upgrading-from-version-52-8#cli-upgrade__clean__title__1)

#### 2.6 Upgrade the Voice Gateway

Log in to the cluster

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

#### 2.7 Upgrade the DB2 with no service instances

DB2 is upgraded as a depdancy for WatsonxGoverance. However, just to be safe it is best to rerun the install component.

Note: [Additional steps are needed for upgrade the instances of DB2.](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=u-upgrading-from-version-52-39#cli-upgrade__svc-inst__title__1)

- Log in to the cluster

```
${CPDM_OC_LOGIN}
```

- Run the upgrade command

```
cpd-cli manage install-components \
--license_acceptance=true \
--components=db2oltp \
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
--components=db2oltp
```

#### 2.8 Upgrade the Cognos Analytics with no service instances

Note: [Additional steps are needed for upgrade the instances of Cognos.](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=u-upgrading-from-version-52-21#cli-upgrade__svc-inst__title__1)

- Log in to the cluster

```
${CPDM_OC_LOGIN}
```

- Run the upgrade command

```
cpd-cli manage install-components \
--license_acceptance=true \
--components=cognos_analytics \
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
--components=cognos_analytics
```

#### 2.9 Upgrade the cpdbr service

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

## Part 3 (Post-upgrade tasks)

### Swapping Models

**(Note: DO NOT PATCH FOR PROD)** Patch the IFM CR with updated models

```
oc patch watsonxaiifm watsonxaiifm-cr \
--namespace=${PROJECT_CPD_INST_OPERANDS} \
--type=merge \
--patch='{"spec":{"install_model_list": ["llama-4-maverick-17b-128e-instruct-fp8","ibm-slate-30m-english-rtrvr","granite-3-2-8b-instruct","mistral-medium-2505","granite-3-3-8b-instruct"]}}'

```

### Update the CPD Route:

Label will be missing from the cpd route post upgrade.

Edit the cpd route to add the label "**expose:external-regional**" to your cpd-route

### Groups Missing:

User groups in access control are missing from UI, but not erased.

Check the zen-metastore-edb postgred DB values:

```
oc rsh zen-metastore-edb-1 -n ${PROJECT_CPD_INST_OPERANDS} 


sh-5.1$ psql -U postgres -d zen

(Note: First 3 command should return the same consistent value. THe last should show only one cpadmin)
zen=# select * from accounts;
zen=# select account_id from user_groups;
zen=# select account_id from platform_users;
zen=# select * from platform_users where username='cpadmin';
```

If there is a difference  "1000" versus "999"

In the same session, back up the table and then update the values to what is shown in accounts (generally 999)

```
CREATE TABLE user_groups_backup AS SELECT * FROM user_groups;

UPDATE user_groups SET account_id = '999';
```

### WxO s3 storage routes

Refernece (Steps below): [https://github.ibm.com/watson-engagement-advisor/wo-cpd-support/blob/main/wxo-support-docs/5.2/5.2.2_storage_s3_route_config_issue_for_nooba.md](https://github.ibm.com/watson-engagement-advisor/wo-cpd-support/blob/main/wxo-support-docs/5.2/5.2.2_storage_s3_route_config_issue_for_nooba.md)

- Identify the s3 route

  ```
  oc get route s3 -n openshift-storage
  ```
- Create rsi patch "wo_custom_s3_route.json" in the "cpd-cli-workspace/olm-utils-workspace/work/rsi" directory

  ```
  [
    { "name": "STORAGE_S3_ROUTE", "value": "<REPLACE WITH YOUR ROUTE>"}
  ]
  ```
- Create and apply the RSI patch for runtime. conversation and archer components.

  ```
  #Create 
  cpd-cli manage create-rsi-patch \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --patch_type=rsi_pod_env_var \
  --patch_name=storage-s3-route-env-override-trm-patch \
  --patch_spec=/tmp/work/rsi/wo_custom_s3_route.json \
  --spec_format=set-env \
  --include_labels=wo.watsonx.ibm.com/component:wo-tools-runtime-manager \
  --state=active

  #Apply
  cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --patch_name=storage-s3-route-env-override-trm-patch
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  #Create 
  cpd-cli manage create-rsi-patch \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --patch_type=rsi_pod_env_var \
  --patch_name=storage-s3-route-env-override-cc-patch \
  --patch_spec=/tmp/work/rsi/wo_custom_s3_route.json \
  --spec_format=set-env \
  --include_labels=wo.watsonx.ibm.com/component:wo-conversation-controller \
  --state=active

  # Apply
  cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --patch_name=storage-s3-route-env-override-cc-patch
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  #Create
  cpd-cli manage create-rsi-patch \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --patch_type=rsi_pod_env_var \
  --patch_name=storage-s3-route-env-override-archersvr-patch \
  --patch_spec=/tmp/work/rsi/wo_custom_s3_route.json \
  --spec_format=set-env \
  --include_labels=wo.watsonx.ibm.com/component:wo-archer-server \
  --state=active

  #Apply
  cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --patch_name=storage-s3-route-env-override-archersvr-patch

  ```
- Verify the patch information is present

  ```
   cpd-cli manage get-rsi-patch-info   --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}   --all
  ```

## Summarize and close out the upgrade

1)Prepare for applying the TemporaryPatch if needed as a post-upgrade task.

2)Schedule a wrap-up meeting and review the upgrade procedure and lessons learned from it.

3)Evaluate the outcome of upgrade with pre-defined goals.

---

End of document
