# Offline Backup Runbook - 5.2.2

## Assumption
- The `cpd-cli` command-line interface (5.2.2) is available in the Bastion node
- The `oc` command-line interface (4.16) is available in the Bastion node
 
## Sourcing the environment variables

```bash
source cpd_vars.sh
```

## Moving images for backup and restore to a private container registry
- Log in to the private container registry
```
podman login ${PRIVATE_REGISTRY_LOCATION} \
-u ${PRIVATE_REGISTRY_PUSH_USER} \
-p ${PRIVATE_REGISTRY_PUSH_PASSWORD}
```
- Log in to the Red Hat entitled registry.

Set the `REDHAT_USER` environment variable to the username of a user who can pull images from `registry.redhat.io`:

```
export REDHAT_USER=<enter-your-username>
```

Set the `REDHAT_PASSWORD` environment variable to the password for the specified user:
```
export REDHAT_PASSWORD=<enter-your-password>
```

Log in to `registry.redhat.io`:

```
podman login registry.redhat.io -u ${REDHAT_USER} -p ${REDHAT_PASSWORD}
```

- Run the following commands to mirror the images the private container registry.
<br>

**ose-cli**
  
```
oc image mirror registry.redhat.io/openshift4/ose-cli:latest ${PRIVATE_REGISTRY_LOCATION}/openshift4/ose-cli:latest --insecure
```

**ubi-minimal**

```
oc image mirror registry.redhat.io/ubi9/ubi-minimal:latest ${PRIVATE_REGISTRY_LOCATION}/ubi9/ubi-minimal:latest --insecure
```

**db2u-velero-plugin**
```
cpd-cli manage copy-image \
--from=icr.io/db2u/db2u-velero-plugin:${VERSION} \
--to=${PRIVATE_REGISTRY_LOCATION}/db2u/db2u-velero-plugin:${VERSION}
```

## Installing IBM Software Hub OADP backup and restore utility components

### Setting up object storage
An S3-compatible object storage that uses Signature Version 4 is needed to store the backups. A bucket must be created in object storage. 
<br>

The IBM Software Hub OADP backup and restore utility supports the following S3-compatible object storage:
- AWS S3
- IBM Cloud Object Storage
- MinIO
- Ceph® Object Gateway

A S3-compatible object storage shared between the source and target cluster is required for the offline restore to a different cluster.

### Setting environment variables required by OADP
#### Update the shell script `cpd_vars.sh` and add below environment variables.

<br>

Here's an example. Change the envirable variable value if needed.

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

#### Sourcing the environment variables

```bash
source cpd_vars.sh
```

### Installing backup and restore components
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

- Create the DataProtectionApplication (DPA) custom resource

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

- Install the cpdbr-tenant service if it's not installed yet.

```
cpd-cli oadp install \
  --component=cpdbr-tenant \
  --namespace ${OADP_PROJECT} \
  --tenant-operator-namespace ${PROJECT_CPD_INST_OPERATORS} \
  --cpdbr-hooks-image-prefix=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd \
  --cpfs-image-prefix=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs \
  --skip-recipes \
  --log-level=debug \
  --verbose
```

## Installing the jq JSON command-line utility

```
wget -O jq https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64
chmod +x ./jq
cp jq /usr/local/bin
```

## Configuring the OADP backup and restore utility
Log in to Red Hat OpenShift Container Platform as a cluster administrator.

```
${OC_LOGIN}
```

Configure the client to set the OADP project:

```
cpd-cli oadp client config set namespace=${OADP_PROJECT}
```

## Preparation for the Offline backup
### Check whether the services that you are using support platform backup and restore.
```
cpd-cli oadp service-registry check \
--tenant-operator-namespace ${PROJECT_CPD_INST_OPERATORS} \
--verbose \
--log-level debug
```

### Check that you installed the correct version of OADP components.
- Check that the OADP operator version is 1.4.x:
```
oc get csv -A | grep -i oadp
```

- Check that the cpd-cli oadp version is 5.2.2:
```
cpd-cli oadp version
```
### Estimating how much storage to allocate for backups
Install the `cpdbr-agent` by running the following command:
```
cpd-cli oadp install --component=cpdbr-agent --namespace=${OADP_PROJECT} --cpd-namespace=${PROJECT_CPD_INST_OPERANDS}
```

Export the following environment variable:
```
export CPDBR_ENABLE_FEATURES=volume-util
```

Estimate how much storage you need to allocate to a backup by running the following command:
```
cpd-cli oadp du-pv
```
### Removing MongoDB-related ConfigMaps
Ensure that these ConfigMaps do not exist in the operand project by running the following commands:
```
oc get cm zen-cs-aux-br-cm
oc get cm zen-cs-aux-ckpt-cm
oc get cm zen-cs-aux-qu-cm
oc get cm zen-cs2-aux-ckpt-cm
```
Delete them if any of them exist.

### Checking the primary instance of every PostgreSQL cluster is in sync with its replicas
- To check the status of the database cluster:

```
oc get clusters.postgresql.k8s.enterprisedb.io \
-n ${PROJECT_CPD_INST_OPERANDS}
```

- To check whether the database cluster is in fenced mode or not:

```
oc get cluster <cluster-name> -o yaml | grep -i fence -A 5
```

- To check the status of the database pods:

```
oc get pods \
-n ${PROJECT_CPD_INST_OPERANDS} \
-l k8s.enterprisedb.io/podRole=instance
```


### Checking if any volumes or pods labeled have special labels
```
oc get pods,pvc -l velero.io/exclude-from-backup=true
oc get pods,pvc -l icpdsupport/ignore-on-nd-backup=true
oc get pods,pvc -l icpdsupport/empty-on-backup=true
```

### Preparing Db2
Log in to Red Hat OpenShift Container Platform as a cluster administrator.
```
${OC_LOGIN}
```

Retrieve the names of the IBM Software Hub deployment's Db2U clusters:
```
oc get db2ucluster -A -ojsonpath='{.items[?(@.spec.environment.dbType=="db2oltp")].metadata.name}'
```

For each Db2U cluster, do the following substeps:
- 1) Export the Db2U cluster name:
```
export DB2UCLUSTER=<db2ucluster_name>
```
- 2) Label the cluster:
```
oc label db2ucluster ${DB2UCLUSTER} db2u/cpdbr=db2u --overwrite
```
- 3) Verify that the Db2U cluster now contains the new label:
```
oc get db2ucluster ${DB2UCLUSTER} --show-labels
```

For each Db2U cluster, check if Q Replication is enabled. 
```
oc get po -n ${PROJECT_CPD_INST_OPERANDS} | grep ${DB2UCLUSTER} | grep qrep
```
Stop the Q Replication if enabled.

### Preparing Db2 Warehouse
Log in to Red Hat OpenShift Container Platform as a cluster administrator.
```
${OC_LOGIN}
```

Retrieve the names of the IBM Software Hub deployment's Db2U clusters:
```
oc get db2ucluster -A -ojsonpath='{.items[?(@.spec.environment.dbType=="db2wh")].metadata.name}'
```

For each Db2U cluster, do the following substeps:
- 1)Export the Db2U cluster name:
```
export DB2UCLUSTER=<db2ucluster_name>
```
- 2)Label the cluster:
```
oc label db2ucluster ${DB2UCLUSTER} db2u/cpdbr=db2u --overwrite
```
- 3)Verify that the Db2U cluster now contains the new label:
```
oc get db2ucluster ${DB2UCLUSTER} --show-labels
```

For each Db2U cluster, if Q Replication is enabled, stop Q Replication by doing the following steps.
```
oc get po -n ${PROJECT_CPD_INST_OPERANDS} | grep ${DB2UCLUSTER} | grep qrep
```

Stop the Q Replication if enabled.

### Stopping SPSS Modeler runtimes and jobs
Log in to Red Hat OpenShift Container Platform as a cluster administrator.
```
${OC_LOGIN}
```

To stop all active SPSS Modeler runtimes and jobs
```
oc delete rta -l type=service,job -l component=spss-modeler
```

To check whether any SPSS Modeler runtime sessions are still running:
```
oc get pod -l type=spss-modeler
```

When no pods are running, no output is produced for this command.

### Checking the status of installed services
```
${CPDM_OC_LOGIN}

cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

## Creating an offline backup
### Log in to Red Hat OpenShift Container Platform as a cluster administrator:
```
${OC_LOGIN}
```
### Backing up the scheduling service
Configure the OADP client to set the IBM Software Hub project to the scheduling service project:
```
cpd-cli oadp client config set cpd-namespace=${PROJECT_SCHEDULING_SERVICE}
```
Configure the OADP client to set the OADP project to the project where the OADP operator is installed:
```
cpd-cli oadp client config set namespace=${OADP_PROJECT}
```

Run service backup prechecks:

```
cpd-cli oadp backup precheck \
--backup-type singleton \
--include-namespaces=${PROJECT_SCHEDULING_SERVICE} \
--log-level=debug \
--verbose \
--hook-kind=br
```

Back up the IBM Software Hub scheduling service:

```
cpd-cli oadp backup create ${PROJECT_SCHEDULING_SERVICE}-offline \
--backup-type singleton \
--include-namespaces ${PROJECT_SCHEDULING_SERVICE} \
--include-resources='operatorgroups,configmaps,catalogsources.operators.coreos.com,subscriptions.operators.coreos.com,customresourcedefinitions.apiextensions.k8s.io,scheduling.scheduler.spectrumcomputing.ibm.com' \
--prehooks=true \
--posthooks=true \
--log-level=debug \
--verbose \
--hook-kind=br \
--selector 'velero.io/exclude-from-backup notin (true)' \
--image-prefix=${PRIVATE_REGISTRY_LOCATION}/ubi9
```

Validate the backup:

```
cpd-cli oadp backup validate \
--backup-type singleton \
--include-namespaces=${PROJECT_SCHEDULING_SERVICE} \
--backup-names ${PROJECT_SCHEDULING_SERVICE}-offline \
--log-level trace \
--verbose \
--hook-kind=br
```
### Backing up IBM Software Hub instance

Specify a backup name

```
export TENANT_BACKUP_NAME=oadp_br_20260401_01
```

Create a backup:

```
cpd-cli oadp tenant-backup create ${TENANT_OFFLINE_BACKUP_NAME} \
--namespace ${OADP_PROJECT} \
--vol-mnt-pod-mem-request=1Gi \
--vol-mnt-pod-mem-limit=4Gi \
--tenant-operator-namespace ${PROJECT_CPD_INST_OPERATORS} \
--mode offline \
--image-prefix=${PRIVATE_REGISTRY_LOCATION}/ubi9 \
--log-level=debug \
--verbose &> ${TENANT_OFFLINE_BACKUP_NAME}.log&
```

Confirm that the tenant backup was created and has a Completed status:

```
cpd-cli oadp tenant-backup list
```

To view the detailed status of the backup, run the following command:

```
cpd-cli oadp tenant-backup status ${TENANT_BACKUP_NAME} --details
```

To view logs of the tenant backup and all sub-backups, run the following command:

```
cpd-cli oadp tenant-backup log ${TENANT_BACKUP_NAME}
```

## Reference
[Offline backup and restore to a different cluster with the IBM Software Hub OADP utility](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=utility-offline-backup-restore-different-cluster)
