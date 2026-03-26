# Offline Restore Runbook - 5.2.2
**Note:**
This runbook is not for a general offline restore.

## Preparing the target cluster
Make sure that the target cluster meets the following requirements:
<br>
- The target cluster has the same storage classes as the source cluster.
- For environments that use a private container registry, such as air-gapped environmenicrts, the target cluster has the same image content source policy as the source cluster.
- The target cluster must be able to pull the 5.2.2 software images required by the restore.
- The target cluster is on the same OpenShift version as the source cluster.
- The target cluster allows for the same node configuration as the source cluster. For example, if the source cluster uses a custom KubeletConfig, the target cluster must allow the same custom KubeletConfig.
- IBM Software Hub & Services have been uninstalled including the clean up of CRs, operators and namespaces.
- IBM Scheduler has been uninstalled including the clean up of namespaces.
- IBM Licensing service and IBM Cert manager have been reinstalled including the clean up of namespaces.

## Set up work statiion

- Make sure the `cpd-cli` command-line interface (5.2.2) and `oc` command-line interface (4.16) are available in the Bastion node.
- Update the shell script `cpd_vars.sh` and update the `VERSION`. <br>
```
export VERSION=5.2.2
```
- Update the shell script `cpd_vars.sh` and add below environment variables like below. <br>
Change the envirable variable value if needed. <br>
**Note** <br>
The `BUCKET_NAME` and `BUCKET_PREFIX` should be the same as that used by the offline backup.

```
export OADP_PROJECT=oadp-operator
export OADP_VERSION=v1.4.4
export S3_URL=<The URL of the object store that you are using to store backups.If the object store is MinIO, you can get the s3url by running oc get route.>
export ACCESS_KEY_ID=<The access key ID to access the object store>
export SECRET_ACCESS_KEY=<The access key secret to access the object store>
export CPDBR_VELERO_PLUGIN_IMAGE_LOCATION=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/cpdbr-velero-plugin:${VERSION}
export CPFS_OADP_PLUGIN_VERSION=4.14.1
export VELERO_POD_CPU_LIMIT=4
export NODE_AGENT_POD_CPU_LIMIT=4
export BUCKET_NAME=velero
export BUCKET_PREFIX=cpd522backup
export REGION=minio
export UPLOADER_TYPE=kopia
```

## Sourcing the environment variables
```bash
source cpd_vars.sh
```
## Configuring backup and restore components
- Create the `${OADP_PROJECT}` project where you want to install the OADP operator.
```
oc new-project ${OADP_PROJECT}
```

- Annotate the ${OADP_PROJECT} project so that Restic pods can be scheduled on all nodes.
```
oc annotate namespace ${OADP_PROJECT} openshift.io/node-selector=""
```

- Install the Red Hat OADP operator if it's not installed yet.

Check whether the OADP operator already installed or not.

```
oc get csv -A | grep -i oadp
```

If the OADP operator Not installed, then follow below link for the OADP installation.
<br>
[Install the OADP operator and configure it to work with your object store](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/backup_and_restore/oadp-application-backup-and-restore#installing-oadp)

If the OADP operator already installed, check that the OADP operator version is 1.4.x

```
oc get csv -A | grep -i oadp
```

Upgrade the the OADP operator if the version is lower than 1.4.

- Create a secret in the ${OADP_PROJECT} project with the credentials of the S3-compatible object store that you are using to store the backups.

<br>
Create a file named `credentials-velero` that contains the credentials for the object store:

```
cat << EOF > credentials-velero
[default]
aws_access_key_id=${ACCESS_KEY_ID}
aws_secret_access_key=${SECRET_ACCESS_KEY}
EOF
```

Create the secret.The name of the secret must be `cloud-credentials`.
```
oc create secret generic cloud-credentials \
--namespace ${OADP_PROJECT} \
--from-file cloud=./credentials-velero
```

- Create or update the DataProtectionApplication (DPA) custom resource

```
cat << EOF | oc apply -f -
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: dpa-sample
  namespace: ${OADP_PROJECT}
spec:
  backupImages: false
  backupLocations:
    - velero:
        accessMode: ReadWrite
        config:
          region: ${REGION}
          s3ForcePathStyle: "true"
          s3Url: ${S3_URL}
        credential:
          key: cloud
          name: cloud-credentials
        default: true
        objectStorage:
          bucket: ${BUCKET_NAME}
          prefix: ${BUCKET_PREFIX}
        provider: aws
  configuration:
    nodeAgent:
      enable: true
      podConfig:
        resourceAllocations:
          limits:
            cpu: "${NODE_AGENT_POD_CPU_LIMIT}"
            memory: 32Gi
          requests:
            cpu: 500m
            memory: 256Mi
        tolerations:
        - effect: NoSchedule
          key: icp4data
          operator: Exists
      timeout: 72h
      uploaderType: ${UPLOADER_TYPE}
    velero:
      customPlugins:
      - image: ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs/cpfs-oadp-plugins:${CPFS_OADP_PLUGIN_VERSION}
        name: cpfs-oadp-plugin
      - image: ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/cpdbr-velero-plugin:${VERSION}
        name: cpdbr-velero-plugin
      - image: ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/swhub-velero-plugin:${VERSION}
        name: swhub-velero-plugin
      - image: ${PRIVATE_REGISTRY_LOCATION}/db2u/db2u-velero-plugin:${VERSION}
        name: db2u-velero-plugin
      defaultPlugins:
      - aws
      - openshift
      - csi
      podConfig:
        resourceAllocations:
          limits:
            cpu: "${VELERO_POD_CPU_LIMIT}"
            memory: 4Gi
          requests:
            cpu: 500m
            memory: 256Mi
      resourceTimeout: 60m
EOF
```

After you create the DPA, do the following checks.

<br>

Check that the velero pods are running in the ${OADP_PROJECT} project.

```
oc get po -n ${OADP_PROJECT}
```

The node-agent daemonset creates one node-agent pod for each worker node. Example output:

```
NAME                                                    READY   STATUS    RESTARTS   AGE
openshift-adp-controller-manager-678f6998bf-fnv8p       2/2     Running   0          55m
node-agent-455wd                                        1/1     Running   0          49m
node-agent-5g4n8                                        1/1     Running   0          49m
node-agent-6z9v2                                        1/1     Running   0          49m
node-agent-722x8                                        1/1     Running   0          49m
node-agent-c8qh4                                        1/1     Running   0          49m
node-agent-lcqqg                                        1/1     Running   0          49m
node-agent-v6gbj                                        1/1     Running   0          49m
node-agent-xb9j8                                        1/1     Running   0          49m
node-agent-zjngp                                        1/1     Running   0          49m
velero-7d847d5bb7-zm6vd                                 1/1     Running   0          49m
```

Verify that the backup storage location PHASE is Available.

```
cpd-cli oadp client config set namespace=${OADP_PROJECT}
```

```
cpd-cli oadp backup-location list
```

Example output:

```
NAME           PROVIDER    BUCKET             PREFIX              PHASE        LAST VALIDATED      ACCESS MODE
dpa-sample-1   aws         ${BUCKET_NAME}     ${BUCKET_PREFIX}    Available    <timestamp>
```

Check that the cpd-cli oadp version is 5.2.2:
```
cpd-cli oadp version
```

## Installing the Licensing and IBM Cert Manager

Log the cpd-cli in to the Red Hat OpenShift Container Platform cluster:
```
${CPDM_OC_LOGIN}
```

Install the License Service for CPD 5.2.2 :
```
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```

**Note**
<br>
If the Red Hat OpenShift Container Platform cert-manager Operator is not installed, the IBM Certificate manager is automatically installed or upgraded when you run the `apply-cluster-components` command.

<br>

Wait for the cpd-cli to return the following message before proceeding to the next step:
```
[SUCCESS]... The apply-cluster-components command ran successfully.
```

## Cleaning up the target cluster after a previous restore
This is only required if there was a previous restore.
[Cleaning up the target cluster after a previous restore](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=utility-offline-backup-restore-different-cluster#reference_acp_qwm_ddc__title__3)

## Restoring the scheduling service
Log in to Red Hat OpenShift Container Platform as a cluster administrator:
```
${OC_LOGIN}
```

Configure the OADP client to set the IBM Software Hub project to the scheduling service project:

```
cpd-cli oadp client config set cpd-namespace=${PROJECT_SCHEDULING_SERVICE}
```

Configure the OADP client to set the OADP project to the project where the OADP operator is installed:

```
cpd-cli oadp client config set namespace=${OADP_PROJECT}
```

Restore an offline backup by running below command.
```
 cpd-cli oadp restore create ${PROJECT_SCHEDULING_SERVICE}-restore \
 --from-backup=${PROJECT_SCHEDULING_SERVICE}-offline \
 --include-resources='operatorgroups,configmaps,catalogsources.operators.coreos.com,subscriptions.operators.coreos.com,customresourcedefinitions.apiextensions.k8s.io,scheduling.scheduler.spectrumcomputing.ibm.com' \
 --include-cluster-resources=true \
 --skip-hooks \
 --log-level=debug \
 --verbose \
 --image-prefix=${PRIVATE_REGISTRY_LOCATION}
```

## Restoring the IBM Software Hub instance and services
**Note** You cannot restore a backup to a different project of the IBM Software Hub instance.
<br>
Log in to Red Hat OpenShift Container Platform as a cluster administrator:
```
${OC_LOGIN}
```

Restore IBM Software Hub by running below command.

```
 cpd-cli oadp tenant-restore create ${TENANT_OFFLINE_BACKUP_NAME}-restore \
 --from-tenant-backup ${TENANT_OFFLINE_BACKUP_NAME} \
 --image-prefix=${PRIVATE_REGISTRY_LOCATION}/ubi9 \
 --verbose \
 --log-level=debug &> ${TENANT_OFFLINE_BACKUP_NAME}-restore.log&
```

Get the status of the installed components:
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Ensure that the status of all of the services is `Completed` or `Succeeded`.

To view a list of restores, run the following command:
```
cpd-cli oadp tenant-restore list
```

To view the detailed status of the restore, run the following command:
```
cpd-cli oadp tenant-restore status ${TENANT_BACKUP_NAME}-restore --details
```

The command shows a varying number of sub-restores in the following form: `cpd-tenant-r-xxx`. You can view more information about these sub-restores by running the following command:
```
cpd-cli oadp restore status <SUB_RESTORE_NAME> --details
```

To view logs of the tenant restore, run the following command:
```
cpd-cli oadp tenant-restore log ${TENANT_BACKUP_NAME}-restore
```
## Completing post-restore tasks

### Applying RSI patches to the control plane
```
cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} -vvv
```

### Restarting IBM Knowledge Catalog lineage pods
Log in to Red Hat OpenShift Container Platform as a cluster administrator:
```
${OC_LOGIN}
```

Restart the wkc-data-lineage-service-xxx pod:
```
oc delete -n ${PROJECT_CPD_INST_OPERANDS} "$(oc get pods -o name -n ${PROJECT_CPD_INST_OPERANDS} | grep wkc-data-lineage-service)"
```

Restart the wdp-kg-ingestion-service-xxx pod:
```
oc delete -n ${PROJECT_CPD_INST_OPERANDS} "$(oc get pods -o name -n ${PROJECT_CPD_INST_OPERANDS} | grep wdp-kg-ingestion-service)"
```

## Reference
[Offline backup and restore to a different cluster with the IBM Software Hub OADP utility](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=utility-offline-backup-restore-different-cluster)
