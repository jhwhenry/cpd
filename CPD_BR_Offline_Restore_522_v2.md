# Offline Restore Runbook - 5.2.2
**Note:**
This runbook is not for a general offline restore.

## 1. Preparing the target cluster
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

### 1.1 Set up work statiion

copy the `cpd_vars.sh` `oadp_vars.sh` from the PROD environment
## Sourcing the environment variables

```bash
source ./cpd_vars.sh
source ./oadp_vars.sh
```

Download Version 14.2.2 of cpd-cli (cpd 5..2.2):
```
wget https://github.com/IBM/cpd-cli/releases/download/v14.2.2/cpd-cli-linux-EE-14.2.2.tgz

tar zxf cpd-cli-linux-EE-14.2.2.tgz
```
```
podman pull icr.io/cpopen/cpd/olm-utils-v3:5.2.2
podman tag icr.io/cpopen/cpd/olm-utils-v3:5.2.2 ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v3:5.2.2

podman login hub.fbond:5000 -u ocadmin -p ocadmin

podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v3:5.2.2 --tls-verify=false --remove-signatures 

export OLM_UTILS_IMAGE=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v3:5.2.2
export OLM_UTILS_LAUNCH_ARGS=" --network=host"
./cpd-cli manage restart-container

./cpd-cli manage login-to-ocp --server=https://api.localcluster.fbond:6443 --token=$(cat /root/.sa/token)
```

### 1.2 Mirroring CPD 5.2.2 images (ETC 8 hours)

- Log in to the IBM Entitled Registry registry
```
./cpd-cli manage login-entitled-registry \
${IBM_ENTITLEMENT_KEY}
```
- Log in to the private container registry
```
./cpd-cli manage login-private-registry \
${PRIVATE_REGISTRY_LOCATION} \
${PRIVATE_REGISTRY_PUSH_USER} \
${PRIVATE_REGISTRY_PUSH_PASSWORD}
```
- Case download
```
./cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION}
```
- List images
```
./cpd-cli manage list-images \ 
--components=${COMPONENTS} \ 
--release=${VERSION} \ 
--inspect_source_registry=true
```
- Mirror images
```
./cpd-cli manage mirror-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--arch=${IMAGE_ARCH} \
--case_download=false
```

## 1.3 Configuring backup and restore components
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

**Moving images for backup and restore to a private container registry**

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
podman login ${PRIVATE_REGISTRY_LOCATION} \
-u ${PRIVATE_REGISTRY_PUSH_USER} \
-p ${PRIVATE_REGISTRY_PUSH_PASSWORD}

podman pull icr.io/cpopen/cpd/swhub-velero-plugin:${VERSION}
podman tag icr.io/cpopen/cpd/swhub-velero-plugin:${VERSION} ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/swhub-velero-plugin:${VERSION}
podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/swhub-velero-plugin:${VERSION} --tls-verify=false --remove-signatures

podman pull icr.io/cpopen/cpfs/cpfs-oadp-plugins:${CPFS_OADP_PLUGIN_VERSION}
podman tag icr.io/cpopen/cpfs/cpfs-oadp-plugins:${CPFS_OADP_PLUGIN_VERSION} ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs/cpfs-oadp-plugins:${CPFS_OADP_PLUGIN_VERSION}
podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs/cpfs-oadp-plugins:${CPFS_OADP_PLUGIN_VERSION} --tls-verify=false --remove-signatures

podman pull registry.redhat.io/ubi9/ubi-minimal:latest
podman tag registry.redhat.io/ubi9/ubi-minimal:latest ${PRIVATE_REGISTRY_LOCATION}/ubi9/ubi-minimal:latest 
podman push ${PRIVATE_REGISTRY_LOCATION}/ubi9/ubi-minimal:latest --tls-verify=false --remove-signatures

podman pull registry.redhat.io/openshift4/ose-cli:latest
podman tag registry.redhat.io/openshift4/ose-cli:latest ${PRIVATE_REGISTRY_LOCATION}/openshift4/ose-cli:latest 
podman push ${PRIVATE_REGISTRY_LOCATION}/openshift4/ose-cli:latest --tls-verify=false --remove-signatures

podman pull icr.io/db2u/db2u-velero-plugin:${VERSION}
podman tag icr.io/db2u/db2u-velero-plugin:${VERSION} ${PRIVATE_REGISTRY_LOCATION}/db2u/db2u-velero-plugin:${VERSION}
podman push ${PRIVATE_REGISTRY_LOCATION}/db2u/db2u-velero-plugin:${VERSION} --tls-verify=false --remove-signatures

podman pull icr.io/cpopen/cpd/cpdbr-velero-plugin:${VERSION}
podman tag icr.io/cpopen/cpd/cpdbr-velero-plugin:${VERSION} ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/cpdbr-velero-plugin:${VERSION} 
podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/cpdbr-velero-plugin:${VERSION} --tls-verify=false --remove-signatures

podman pull icr.io/cpopen/cpd/cpdbr-oadp:${VERSION}
podman tag icr.io/cpopen/cpd/cpdbr-oadp:${VERSION} ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/cpdbr-oadp:${VERSION} 
podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/cpdbr-oadp:${VERSION} --tls-verify=false --remove-signatures

podman pull icr.io/cpopen/cpfs/cpfs-utils:4.6.5 
podman tag icr.io/cpopen/cpfs/cpfs-utils:4.6.5 ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs/cpfs-utils:4.6.5
podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs/cpfs-utils:4.6.5 --tls-verify=false --remove-signatures
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

## 1.4 Installing shared cluster components for IBM Software Hub

https://www.ibm.com/docs/en/software-hub/5.2.x?topic=cluster-installing-shared-components

**Installing the Licensing and IBM Cert Manager**

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

**Cleaning up the target cluster after a previous restore**
This is only required if there was a previous restore.
[Cleaning up the target cluster after a previous restore](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=utility-offline-backup-restore-different-cluster#reference_acp_qwm_ddc__title__3)

**Installing the scheduling service**

```
cpd-cli manage apply-scheduler \
--release=${VERSION} \
--license_acceptance=true \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE}
```

## 2. Restoring the IBM Software Hub instance and services
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
