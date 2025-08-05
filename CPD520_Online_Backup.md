# Online Backup Runbook for watsonx.AI


## Set environment
1. Source the `cpd_vars.sh`.

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

```
oc image mirror registry.redhat.io/openshift4/ose-cli:latest ${PRIVATE_REGISTRY_LOCATION}/openshift4/ose-cli:latest --insecure
oc image mirror registry.redhat.io/ubi9/ubi-minimal:latest ${PRIVATE_REGISTRY_LOCATION}/ubi9/ubi-minimal:latest --insecure
```

## Installing IBM Software Hub OADP backup and restore utility components

### Setting up object storage
An S3-compatible object storage that uses Signature Version 4 is needed to store the backups. A bucket must be created in object storage. 
<br>
The MinIO object storage can be used for the air-gapped cluster.

### Creating environment variables
Create an shell script named `oadp_br_vars.sh`.
<br>
Here's an example. Change the envirable variable value if needed.
```
export OADP_PROJECT=adp-openshift
export ACCESS_KEY_ID=<The access key ID to access the object store>
export SECRET_ACCESS_KEY=<The access key secret to access the object store>
export CPDBR_VELERO_PLUGIN_IMAGE_LOCATION=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/cpdbr-velero-plugin:${VERSION}.${IMAGE_ARCH}
export VELERO_POD_CPU_LIMIT=2
export NODE_AGENT_POD_CPU_LIMIT=2
export S3_URL=<The URL of the object store that you are using to store backups.If the object store is MinIO, you can get the s3url by running oc get route.>
export BUCKET_NAME=velero
export BUCKET_PREFIX=cpdbackup
export REGION=minio
```

Source the shell script.
```
source oadp_br_vars.sh
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

- Install the cpdbr-tenant service.
```
cpd-cli oadp install \
--component=cpdbr-tenant \
--namespace ${OADP_PROJECT} \
--tenant-operator-namespace ${PROJECT_CPD_INST_OPERATORS} \
--cpdbr-hooks-image-prefix=${PRIVATE_REGISTRY}/cpdbr-oadp:${VERSION} \
--cpfs-image-prefix=${PRIVATE_REGISTRY} \
--skip-recipes \
--log-level=debug \
--verbose
```
- Install the Red Hat OADP operator.

Check whether the OADP operator already installed or not.

```
oc get csv -A | grep "OADP Operator"

cpd-cli oadp version
```

If the OADP operator Not installed, then run below command.

```
cpd-cli oadp install \
  --component oadp-operator \
  --namespace oadp-operator \
  --oadp-version v1.4.4 \
  --log-level trace \
  --velero-cpu-limit 2 \
  --velero-mem-limit 2Gi \
  --velero-cpu-request 1 \
  --velero-mem-request 256Mi \
  --node-agent-pod-cpu-limit 2 \
  --node-agent-pod-mem-limit 2Gi \
  --node-agent-pod-cpu-request 0.5 \
  --node-agent-pod-mem-request 256Mi \
  --uploader-type ${UPLOADER_TYPE} \
  --bucket-name=velero \
  --prefix=cpdbackup \
  --access-key-id ${ACCESS_KEY_ID} \
  --secret-access-key ${SECRET_ACCESS_KEY} \
  --s3force-path-style=true \
  --region=minio \
  --s3url ${S3_URL} \
  --cpfs-oadp-plugin-image "icr.io/cpopen/cpfs/cpfs-oadp-plugins:4.10.0" \
  --swhub-velero-plugin-image "icr.io/cpopen/cpd/cpdbr-velero-plugin:5.2.0" \
  --cpdbr-velero-plugin-image "icr.io/cpopen/cpd/swhub-velero-plugin:5.2.0" \
  --verbose
```

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
### Create the DataProtectionApplication (DPA) custom resource, and specify a name for the instance.
Here's an example of the DPA configuration.

```
cat << EOF | oc apply -f -
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: dpa-sample
spec:
  configuration:
    velero:
      customPlugins:
      - image: icr.io/cpopen/cpfs/cpfs-oadp-plugins:4.10.0
        name: cpfs-oadp-plugin
      - image: icr.io/cpopen/cpd/cpdbr-velero-plugin:5.2.0
        name: cpdbr-velero-plugin
      - image: icr.io/cpopen/cpd/swhub-velero-plugin:55.2.0
        name: swhub-velero-plugin
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
    nodeAgent:
      enable: true
      uploaderType: kopia
      timeout: 72h
      podConfig:
        resourceAllocations:
          limits:
            cpu: "${NODE_AGENT_POD_CPU_LIMIT}"
            memory: 32Gi
          requests:
            cpu: 500m
            memory: 256Mi
        tolerations:
        - key: icp4data
          operator: Exists
          effect: NoSchedule
  backupImages: false
  backupLocations:
    - velero:
        provider: aws
        default: true
        objectStorage:
          bucket: ${BUCKET_NAME}
          prefix: ${BUCKET_PREFIX}
        config:
          region: ${REGION}
          s3ForcePathStyle: "true"
          s3Url: ${S3_URL}
        credential:
          name: cloud-credentials
          key: cloud
EOF
```
After you create the DPA, do the following checks.

<br>
1)Check that the velero pods are running in the ${OADP_PROJECT} project.
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
cpd-cli oadp backup-location list
```

Example output:

```
NAME           PROVIDER    BUCKET             PREFIX              PHASE        LAST VALIDATED      ACCESS MODE
dpa-sample-1   aws         ${BUCKET_NAME}     ${BUCKET_PREFIX}    Available    <timestamp>
```

## Installing the jq JSON command-line utility

```
wget -O jq https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64
chmod +x ./jq
cp jq /usr/local/bin
```
## Configuring the OADP backup and restore utility
Configure the client to set the OADP project:
```
cpd-cli oadp client config set namespace=${OADP_PROJECT}
```

## Creating volume snapshot classes on the source cluster

If you are backing up IBM Software Hub on NetApp Trident storage, create the following volume snapshot class.
```
cat << EOF | oc apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapclass
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: csi.trident.netapp.io
deletionPolicy: Retain
EOF
```
## Preparation for the Online backup
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
oc get csv -A | grep "OADP Operator"
```

- Check that the cpd-cli oadp version is 5.1.0:
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
### Checking the primary instance of every PostgreSQL cluster is in sync with its replicas
```
oc get clusters.postgresql.k8s.enterprisedb.io \
-n ${PROJECT_CPD_INST_OPERANDS}
```

### Checking the status of installed services
```
${CPDM_OC_LOGIN}

cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

## Creating an online backup
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
--exclude-checks ValidVolumeSnapshotClass \
--hook-kind=checkpoint \
--log-level=debug \
--verbose
```

Back up the IBM Software Hub scheduling service:

```
cpd-cli oadp backup create ${PROJECT_SCHEDULING_SERVICE}-online \
--backup-type singleton \
--include-namespaces=${PROJECT_SCHEDULING_SERVICE} \
--include-resources='operatorgroups,configmaps,catalogsources.operators.coreos.com,subscriptions.operators.coreos.com,customresourcedefinitions.apiextensions.k8s.io,scheduling.scheduler.spectrumcomputing.ibm.com' \
--prehooks=false \
--posthooks=false \
--with-checkpoint \
--log-level=debug \
--verbose \
--hook-kind=checkpoint \
--selector 'icpdsupport/ignore-on-nd-backup notin (true)'
```

Validate the backup:

```
cpd-cli oadp backup validate \
--backup-type singleton \
--include-namespaces=${PROJECT_SCHEDULING_SERVICE} \
--backup-names ${PROJECT_SCHEDULING_SERVICE}-online \
--log-level trace \
--verbose \
--hook-kind checkpoint
```
### Backing up IBM Software Hub instance

Specify a backup name

```
export TENANT_BACKUP_NAME=oadp_br_20250805_01
```

Create a backup:

```
cpd-cli oadp tenant-backup create ${TENANT_BACKUP_NAME} \
--tenant-operator-namespace ${PROJECT_CPD_INST_OPERATORS} \
--log-level=debug \
--verbose
```

Confirm that the tenant backup was created and has a Completed status:

```
cpd-cli oadp tenant-backup list
```

To view the detailed status of the backup, run the following command:

```
cpd-cli oadp tenant-backup status ${TENANT_BACKUP_NAME} \
--details
```

To view logs of the tenant backup and all sub-backups, run the following command:

```
cpd-cli oadp tenant-backup log ${TENANT_BACKUP_NAME}
```

## Reference
[IBM Software Hub online backup and restore to the same cluster with the OADP utility](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=cluster-backup-restore-software-hub-oadp-utility)
