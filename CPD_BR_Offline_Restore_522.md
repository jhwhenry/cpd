# Offline Restore Runbook - 5.2.2
**Note:**
This runbook is not for a general offline restore.

### Preparing the target cluster
Make sure that the target cluster meets the following requirements:
- The target cluster has the same storage classes as the source cluster.
- For environments that use a private container registry, such as air-gapped environments, the target cluster has the same image content source policy as the source cluster.
- The target cluster must be able to pull the 5.2.2 software images required by the restore.
- The target cluster is on the same OpenShift version as the source cluster.
- The target cluster allows for the same node configuration as the source cluster. For example, if the source cluster uses a custom KubeletConfig, the target cluster must allow the same custom KubeletConfig.

### Uninstall of the IBM Software Hub & Services
The corresponding namespaces should be deleted.

### Set up work statiion

- Make sure the `cpd-cli` command-line interface (5.2.2) and `oc` command-line interface (4.16) are available in the Bastion node.
- Update the shell script `cpd_vars.sh` and update the `VERSION`.
<br>
```
export VERSION=5.2.2
```
- Update the shell script `cpd_vars.sh` and add below environment variables like below.
<br>
Change the envirable variable value if needed.
<br>
**Note**
<br>
The `BUCKET_NAME` and `BUCKET_PREFIX` should be the same as that used by the offline backup.
```
export OADP_PROJECT=openshift-adp
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
export REGION=us-east
export UPLOADER_TYPE=kopia
```

### Sourcing the environment variables
```bash
source cpd_vars.sh
```

### Installing the Pre-requistes
- Node settings
- OADP Operator
### Installing the Licensing and IBM Cert Manager


### Restoring the scheduling service
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

Restore an offline backup by running one of the following commands.
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

### Restoring an IBM Software Hub instance.
**Note** You cannot restore a backup to a different project of the IBM Software Hub instance.
<br>
Log in to Red Hat OpenShift Container Platform as a cluster administrator:
```
${OC_LOGIN}
```

Restore IBM Software Hub by running one of the following commands.

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
### Completing post-restore tasks

#### Applying RSI patches to the control plane
```
cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} -vvv
```

#### Restarting IBM Knowledge Catalog lineage pods
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
