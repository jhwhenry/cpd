# Pre-check Runbook for watsonx Orchestrate deployment

## Set environment
1. Source the `cpd_vars.sh`.

```bash
source cpd_vars.sh
```

2. Create folder to put all the information gather.

```bash
mkdir -p /opt/ibm/wxo/pre-installation
cd /opt/ibm/wxo/pre-installation
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

## Collect PVC size

```bash
oc get pvc -A > pvc_list.txt
```
