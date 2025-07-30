# Pre-check Runbook for watsonx Orchestrate deployment

## 1. Set environment
1. Source the `cpd_vars.sh`.

```bash
source cpd_vars.sh
```

2. Create folder to put all the information gather.

```bash
mkdir -p /opt/ibm/wxo/pre-installation
cd /opt/ibm/wxo/pre-installation
```

**NOTE:**
<br>Please zip the folder created at the end of the session for further analysis of SWAT Team.

3. Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```bash
${CPDM_OC_LOGIN}
```

## 2. Get namespaces related resources and settings


1. Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```bash
${CPDM_OC_LOGIN}
```

2. Export variables.

```bash
export PROJECT_CPFS_OPS=cpd-operators        
export PROJECT_CPD_INST_OPERANDS=cpd
```

3. Collect the information about namespace metadata, resource quota, limit range, namespace scope, network policy, cluster service version, subscription and pods in each namespace.

```bash
oc project ${PROJECT_CPD_INST_OPERANDS}
```
```bash
for ns in ${PROJECT_CPFS_OPS} ${PROJECT_CPD_INST_OPERANDS}; do echo "==== Namespace:  $ns ====" ; oc get project $ns -o yaml > project-$ns.yaml;oc get ResourceQuota -o yaml -n $ns > quota-$ns.yaml;oc get LimitRange -o yaml -n $ns > limitrange-$ns.yaml;oc get namespacescope -o yaml -n $ns > namespacescope-$ns.yaml;oc get NetworkPolicy -o yaml -n $ns > networkpolicy-$ns.yaml;oc get csv -n $ns > csv-$ns.yaml;oc get sub -n $ns > sub-$ns.yaml;oc get pods -n $ns > pods-$ns.yaml;done
```

4. Check cert manager.

```bash
oc get csv | grep ibm-cert-manager > cert-manager.txt
```

## 3. Validate CR status

1. Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```bash
${CPDM_OC_LOGIN}
```

2. Run the following command.

```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} > cr_status.txt
```

## 4. Validate for Hot fix and Custom Configuration


1. Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```bash
${CPDM_OC_LOGIN}
```

2. Run the following command.

[hot fix]: <> (oc get <CR Kind> <CR Name> -n $PROJECT_CPD_INST_OPERANDS -o yaml) 


```bash
oc get ZenService lite-cr -n $PROJECT_CPD_INST_OPERANDS -o yaml > zen.yaml
oc get CommonService common-service -n $PROJECT_CPD_INST_OPERANDS -o yaml > cs.yaml
oc get WatsonAssistant wa -n $PROJECT_CPD_INST_OPERANDS -o yaml > wa.yaml
oc get WatsonDiscovery wd -n $PROJECT_CPD_INST_OPERANDS -o yaml > wd.yaml
```

3. Validate for hotfixes in CRs.

```bash
for i in $(oc get crd | grep watson.ibm.com | awk '{ print $1 }'); do echo "---------$i------------"; oc get $i $(oc get $i | grep -v "NAME" | awk '{ print $1 }') -o yaml > cr-$i.txt; if grep -q "image_digests" cr-$i.txt; then echo "Hot fix detected for cr-$i"; fi; done
```
```bash
for i in $(oc get crd | grep cpd.ibm.com | awk '{ print $1 }'); do echo "---------$i------------"; oc get $i $(oc get $i | grep -v "NAME" | awk '{ print $1 }') -o yaml > cr-$i.txt; if grep -q "image_digests" cr-$i.txt; then echo "Hot fix detected for cr-$i"; fi; done
```

## 5. Validate for RSI patches

1. Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```bash
${CPDM_OC_LOGIN}
```

2. Run following command.

```bash
cpd-cli manage get-rsi-patch-info \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--all
```

## 5. Validate CASE is Download to Workstation

1. Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```bash
${CPDM_OC_LOGIN}
```

2. Run following command.

```bash
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION}
```

## 6. Validate mirroring is completed

1. Check the log files in the work directory generated during the image mirroring.

```bash
grep "error" ${CPD_CLI_MANAGE_WORKSPACE}/work/mirror_*.log
```

**NOTE:**
<br>If the `CPD_CLI_MANAGE_WORKSPACE` is not set, we can export the variable by doing this:

```bash
podman inspect olm-utils-play-v3 | grep Source
export CPD_CLI_MANAGE_WORKSPACE=<value>
```

2. Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```bash
${CPDM_OC_LOGIN}
```

3. Log in to the private container registry.

```bash
cpd-cli manage login-private-registry \
${PRIVATE_REGISTRY_LOCATION} \
${PRIVATE_REGISTRY_PULL_USER} \
${PRIVATE_REGISTRY_PULL_PASSWORD}
```

4. Confirm that the images were mirrored to the private container registry: Inspect the contents of the private container registry:

```bash
cpd-cli manage list-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--case_download=false
```

5. The output is saved to the `list_images.csv` file in the `work/offline/${VERSION}` directory.
Check the output for errors:

```bash
grep "level=fatal" ${CPD_CLI_MANAGE_WORKSPACE}/work/offline/${VERSION}/list_images.csv
```

**NOTE:**
<br>The command returns images that are missing or that cannot be inspected which needs to be addressed.

## 7. Validate if Knative and Event Operator version

1. Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```bash
${CPDM_OC_LOGIN}
```

2. Verify Red Hat OpenShift Serverless Operator version.

```bash
oc get csv -n=openshift-serverless | grep serverless-operator
```

Output expected:

```bash
oc get csv -n=openshift-serverless | grep serverless-operator
serverless-operator.v1.35.0           Red Hat OpenShift Serverless   1.35.0    serverless-operator.v1.34.0   Succeeded
```

Version 1.35 is the latest version

3. IBM Cloud Pak foundational services CASE package should be available, if not, please download it from the following link:
https://www.ibm.com/docs/en/software-hub/5.1.x?topic=ups-upgrading-red-hat-openshift-serverless-knative-eventing-1

4. Verify IBM Events Operator version.

```bash
oc get csv -n=${PROJECT_IBM_EVENTS} | grep ibm-events
```

Output expected:

```bash
oc get csv -n=${PROJECT_IBM_EVENTS} | grep ibm-events
ibm-events-operator.v5.0.1                    IBM Events Operator                    5.0.1                                   Succeeded
```

## 8. Check PVC size

1. Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```bash
${CPDM_OC_LOGIN}
```

2. Run following command.

```bash
oc get pvc -A > pvc_list.txt
```

## 9. Backing up and retaining temporary patches

1. Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```bash
${CPDM_OC_LOGIN}
```

2. Run following command.

```bash
oc get temporarypatch -n ${PROJECT_CPD_INST_OPERANDS}  > patches_list.txt
```

3. Save the yaml file for your existing temporary patches.

```bash
oc get temporarypatch -o yaml > old_patch_backups.yaml
```

## 10. Run Health Checks


1. General OCP

```bash
oc get nodes,mcp,co > ocp_status.txt
oc version > oc_version.txt
oc get pods -A > pod_list.txt
oc describe nodes > nodes_desc.txt
```

2. Validate Cluster, Nodes, Operands and Operators

```bash
cpd-cli health runcommand \
--commands=cluster,nodes,operands,operators \
--control_plane_ns=${PROJECT_CPD_INST_OPERANDS} \
--log-level=debug \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--save \
--verbose
```

3. Network Performance.

[network]: <> (IBM Entitled Registry)

```bash
cpd-cli health network-performance \
--log-level=debug \
--minbandwidth=350 \
--save \
--verbose
```

4. Network Connectivity.

```bash
cpd-cli health network-connectivity \
--control_plane_ns=${PROJECT_CPD_INST_OPERANDS} \
--save \
--verbose
```
5. Storage status.

```bash
# Get more details of the ocs cluster

oc describe cephcluster ocs-storagecluster-cephcluster -n openshift-storage > odf-status.txt

# Verify that the provisioner was created and that the corresponding daemon sets were created by:

oc get all -n openshift-local-storage > local-storage.txt

# Verify the created storage class:

oc get sc
```

6. Storage performance.

Create param.yaml, client have to replace values:
- ocp URL
- ocp credentials
- storageclass used

```bash
# OCP Parameters
ocp_url: https://<required>:6443
ocp_username: <required>  # a cluster admin user
ocp_password: <required>
ocp_token: <required if user/password not available>
ocp_apikey: <required if neither user/password or token not available>

storageClass_ReadWriteOnce: ocs-storagecluster-ceph-rbd # eg "ocs-storagecluster-ceph-rbd"
storageClass_ReadWriteMany: ocs-storagecluster-cephfs  # eg "ocs-storagecluster-cephfs"
arch: amd64  # required, valid values: amd64, ppc64le, s390x
# default image location shown, update as needed if using a private image registry
imageurl: icr.io/cpopen/cpd/k8s-storage-perf:5.1.0

run_storage_perf: true # true = run the "storage-performance" tests

############################ STORAGE PERFORMANCE PARAMETERS START #######################

# project to create test resources in, will be created if not already present
storage_perf_namespace: ibmtigerstorageperf

# used internally by the k8s-storage-perf container (changing not recommended)
logfolder: '.logs' 

cluster_infrastructure: <optional> # env. e.g.: ibmcloud, aws, azure, vmware - "Environment" in result.csv
cluster_name: <optional> # cluster name - "Cluster Name" in result.csv
storage_type: <optional> # storage vendor e.g.: portworx, ocs, nfs - "Storage Type" in result.csv

# To run the performace jobs on a dedicated compute nodes, set the node label which meet the criteria.
# The idea is to gather performance data when the jobs are running remotely from a storage node.
# A cluster administrator can label a node by running this query with appropriate label key/value: 
# oc label node <node name> "<label_key>=<label_value>" --overwrite
dedicated_compute_node:
   label_key: "<optional>"
   label_value: "<optional>"

# size of pvcs created for storage tests
rwx_storagesize: 10Gi
rwo_storagesize: 10Gi

# IO flags to use: direct|dsync|sync|none. 
# Use direct for Cloud and VMware based environments when storage infrastructure is network attached.
file_extra_flags: dsync     # IO flags to use. dsync |none. 

#sysbench random read
sysbench_random_read: true
rread_threads: 1,4,8,16
rread_fileTotalSize: 128m
rread_fileNum: 128
rread_fileBlockSize: 4k,8k,16k

#sysbench random write
sysbench_random_write: true
rwrite_threads: 1,4,8,16
rwrite_fileTotalSize: 4096m
rwrite_fileNum: 4
rwrite_fileBlockSize: 4k,8k,16k

#sysbench sequential read
sysbench_sequential_read: true
sread_threads: 1,2
sread_fileTotalSize: 4096m
sread_fileNum: 4
sread_fileBlockSize: 512m,1g

#sysbench sequential write
sysbench_sequential_write: true
swrite_threads: 1,2
swrite_fileTotalSize: 4096m
swrite_fileNum: 4
swrite_fileBlockSize: 512m,1g

############################ STORAGE PERFORMANCE PARAMETERS END #########################
```

```bash
cpd-cli health storage-performance \
--param <path of paramperf.yml file> \
--save \
--verbose
```
