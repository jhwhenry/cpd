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
export OADP_PROJECT=<The project where you want to install the OADP operator. The default project is openshift-adp>
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
-Install the Red Hat OADP operator.
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

2. Create folder to put all the information gather.

```bash
mkdir -p /opt/ibm/wxo/pre-installation
cd /opt/ibm/wxo/pre-installation
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

**NOTE**

<br>
Please zip the folder created at the end of the session for further analysis of SWAT Team.

## Collect namespaces related resources and settings

1. Log in to the OpenShift cluster

```bash
${OC_LOGIN}
```

2. Export variables
<br>

**Note**: Change the namesapces as needed

```bash
export PROJECT_CPD_INST_OPERATORS=cpd-operators        
export PROJECT_CPD_INST_OPERANDS=cpd
```

3. Collect the information about namespace metadata, resource quota, limit range, namespace scope, network policy, cluster service version, subscription and pods in each namespace.

```bash
oc project ${PROJECT_CPD_INST_OPERANDS}
```

```bash
for ns in ${PROJECT_CPD_INST_OPERATORS} ${PROJECT_CPD_INST_OPERANDS}; do echo "==== Namespace:  $ns ====" ; oc get project $ns -o yaml > project-$ns.yaml;oc get ResourceQuota -o yaml -n $ns > quota-$ns.yaml;oc get LimitRange -o yaml -n $ns > limitrange-$ns.yaml;oc get NetworkPolicy -o yaml -n $ns > networkpolicy-$ns.yaml; oc get pods -n $ns > pod-list-$ns.txt;done
```

## Check the certificate manager
Check whether IBM Certificate Manager is used in this cluster.

```bash
oc get csv -A | grep ibm-cert-manager > cert-manager.txt
```

## Check the Knative and Event Operator version

1. Verify Red Hat OpenShift Serverless Operator version.

```bash
oc get csv -n=openshift-serverless | grep serverless-operator > serverless-operator.txt
```

2. Verify IBM Events Operator version.

```bash
oc get csv -n=${PROJECT_IBM_EVENTS} | grep ibm-events > ibm-events.txt
```

## Check the OpenShift AI Operator

1. Verify Red Hat OpenShift Serverless Operator version.

```bash
oc get pods -A | grep -i rhods-operator > openshift-ai.txt
```

```bash
oc get subs -A | grep -i rhods-operator >> openshift-ai.txt
```

```bash
oc get DSCInitialization -A | grep -i dsci >> openshift-ai.txt
```

## Check the secrets used for connecting to the MCG
Get the names of the secrets that contain the NooBaa account credentials and certificate

```bash
oc get secrets --namespace=openshift-storage > ocp-storage-secrets.txt
```

Get the watson-assistant secrets were created in the operands project for the watson assistant instance:
```
oc get secrets --namespace=${PROJECT_CPD_INST_OPERANDS} \
noobaa-account-watson-assistant \
noobaa-cert-watson-assistant \
noobaa-uri-watson-assistant > wa-secrects.txt
```

## Check the node resources

```bash
oc describe nodes > nodes_desc.txt
```

## Collect CR status

1. Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```bash
${CPDM_OC_LOGIN}
```

2. Run the following command.

```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} > cr_status.txt
```

## Run Health Checks for the Cluster, Nodes, Operands and Operators

```bash
cpd-cli health runcommand \
--commands=cluster,nodes,operands,operators \
--control_plane_ns=${PROJECT_CPD_INST_OPERANDS} \
--log-level=debug \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--save \
--verbose
```

## Run Health Checks for the Network
1. Network Performance.

```bash
cpd-cli health network-performance \
--log-level=debug \
--minbandwidth=350 \
--save \
--verbose
```

2. Network Connectivity.

```bash
cpd-cli health network-connectivity \
--control_plane_ns=${PROJECT_CPD_INST_OPERANDS} \
--save \
--verbose
```
## Run Health Checks for the Storage

1. Login to the private image registry depends on the container runtime.

* 1.1 Podman login.

```bash
podman login ${PRIVATE_REGISTRY_LOCATION} \
-u ${PRIVATE_REGISTRY_PULL_USER} \
-p ${PRIVATE_REGISTRY_PULL_PASSWORD}
```

2. Create parameter file for storage performance health check.

* 2.1 Export variable for namespace.

```bash
export STOR_PERF=ibm-storage-performance
```

* 2.2 Create file.

```bash
cat <<EOF > storage_perf.yml
# OCP Parameters
ocp_url: ${OCP_URL}
ocp_username: ${OCP_USERNAME}
ocp_password: ${OCP_PASSWORD}
ocp_token: ${OCP_TOKEN}
ocp_apikey: <required if neither user/password or token not available>

storageClass_ReadWriteOnce: ${STG_CLASS_BLOCK}
storageClass_ReadWriteMany: ${STG_CLASS_FILE}
arch: ${IMAGE_ARCH}

imageurl: ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/k8s-storage-perf:${VERSION}.${IMAGE_ARCH}

run_storage_perf: true

storage_perf_namespace: ${STOR_PERF}
logfolder: '.logs'

cluster_infrastructure: <optional>
cluster_name: <optional>
storage_type: <optional>

dedicated_compute_node:
   label_key: "<optional>"
   label_value: "<optional>"

rwx_storagesize: 10Gi
rwo_storagesize: 10Gi

file_extra_flags: dsync

sysbench_random_read: false
rread_threads: 8
rread_fileTotalSize: 128m
rread_fileNum: 128
rread_fileBlockSize: 4k

sysbench_random_write: true
rwrite_threads: 8
rwrite_fileTotalSize: 4096m
rwrite_fileNum: 4
rwrite_fileBlockSize: 4k

sysbench_sequential_read: false
sread_threads: 2
sread_fileTotalSize: 4096m
sread_fileNum: 4
sread_fileBlockSize: 1g

sysbench_sequential_write: true
swrite_threads: 2
swrite_fileTotalSize: 4096m
swrite_fileNum: 4
swrite_fileBlockSize: 1g
EOF
```

3. Run command.
<br>

**NOTE:**
Remember that the `image-tag` option, it needs to match what it is in the `imageurl` in the `param.yml` file.

```bash
cpd-cli health storage-performance \
--image-prefix=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd \
--image-tag=${VERSION}.${IMAGE_ARCH} \
--param=storage_perf.yml \
--verbose \
--save
```
4. Run the cleanup
Run a [cleanup script](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=health-storage-performance#health-storage-perf__cleanup__title__1) to remove any resources that were created after you ran the cpd-cli health storage-performance command. 
