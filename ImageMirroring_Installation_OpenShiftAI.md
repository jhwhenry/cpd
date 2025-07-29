# Images mirroring for the OpenShift AI installation

---

## Setting up a client workstation

### Set the installation directory

***Note***:
<br>
You can change the directory path if needed.

```
export WXO_INSTALL_DIR=/opt/ibm/wxo
```
### Install the oc client and oc-mirror plugin

#### 1.Download with wget

```
mkdir -p $WXO_INSTALL_DIR
cd $WXO_INSTALL_DIR
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.16.40/openshift-client-linux-4.16.40.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.16.40/oc-mirror.tar.gz

```

#### 2.Extract the tar file

```
tar -xvf openshift-client-linux-4.16.40.tar.gz
tar -xvf oc-mirror.tar.gz
```

#### 3.Make the oc client executable from any directory.

```
cp oc /usr/bin/oc
chmod +x oc-mirror
cp oc-mirror /usr/bin/oc-mirror
```

Validate with the following commands:
```
oc version
oc mirror help
```

#### 4.Configure the oc-mirror credentials
Configure the oc-mirror credentials by following below documentation.
[Configuring credentials that allow images to be mirrored](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/disconnected_installation_mirroring/installing-mirroring-disconnected?_gl=1*1w781l1*_ga*Nzg0NTU2NDczLjE3NTM0MDU0Nzc.*_ga_FYECCCS21D*czE3NTM3NDM3MjQkbzExJGcxJHQxNzUzNzQ2MzM1JGo0NCRsMCRoMA..#installation-adding-registry-pull-secret_installing-mirroring-disconnected)

## Build the ImageSetConfiguration file

Create an ImageSetConfiguration file `openshift_ai_416.yaml` for OpenShift AI 2.19 as below:

**Note:** The `169.63.179.172:8080` in the registry section needs to be changed to your private image registry location accordingly.

```
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
archiveSize: 4                                                      
storageConfig:                                                      
  registry:
    imageURL: 169.63.179.172:8080/cp               
    skipTLS: true
mirror:
  platform:
    channels:
    - name: stable-4.16                                           
      type: ocp
    graph: true                                                     
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.16
    packages:
    - name: "rhods-operator"
      channels:
      - name: "stable-2.19"
        minVersion: "2.19.2"                                             
  additionalImages:
  - name: registry.redhat.io/ubi9/ubi:latest                        
  helm: {}
```

## Mirror images

**Note:** The `169.63.179.172:8080` in below commands needs to be changed to your private image registry location accordingly.

```
cd $WXO_INSTALL_DIR
oc mirror --config=./openshift_ai_416.yaml docker://169.63.179.172:8080 --source-skip-tls --dest-skip-tls --verbose 9
```
Once complete, there’s a folder named `oc-mirror-workspace` created looks like below:

```
ls oc-mirror-workspace
```

The output looks like below:

```
publish results-1750641947
```

Multipel resurce files created in the results folder:
```
ls oc-mirror-workspace/results-1750641947
```

The output looks like below:
```
catalogSource-cs-redhat-operator-index.yaml  charts  imageContentSourcePolicy.yaml  mapping.txt  release-signatures  updateService.yaml
```


## Apply the ImageDigestMirrorSet
### 1.Login the OCP cluster using the cluster admin role.
For example:
```
oc login https://api.685849c881da5dd628552ad6.am1.xxx.yyy.com:6443 -u kubeadmin -p DP73S-iSmys-SuQ6j-9Ax8f
```

### 2.Apply the ImageDigestMirrorSet
For example:
```
cd /root/mirroring/oc-mirror-workspace/results-1750641947
oc create -f $(oc adm migrate icsp imageContentSourcePolicy.yaml | cut -f 4 -d ' ')
```

Wait until the image content source policy or image digest mirror set applied successfully.
All the nodes should be `Ready` before you proceed to the next step. For example, if you see Ready,SchedulingDisabled, wait for the process to complete.

```
watch -n 5 "oc get nodes"
```
## Installing the OpenShift AI

### Disable the default OperatorHub sources
```
oc patch OperatorHub cluster --type json -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'
```

### Create the catalog source
Create the catalog source with the yaml file generated in the `Mirror images` step.
```
oc apply -f $WXO_INSTALL_DIR/oc-mirror-workspace/results-1750641947/catalogSource-cs-redhat-operator-index.yaml
```

Validate whether catalog source created successfully.
```
oc get pods -n openshift-marketplace | grep -i cs-redhat-operator-index
```
Make sure the pod of the cs-redhat-operator-index catalog source is up and running.

### Create the redhat-ods-operator project
```
oc new-project redhat-ods-operator
```
### Create the rhods-operator operator group
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: rhods-operator
  namespace: redhat-ods-operator
EOF
```

### Create the rhods-operator operator subscription
**Note:**
You may have to change the catalog source name accordingly.
```
export CHANNEL_VERSION=stable-2.19

cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhods-operator
  namespace: redhat-ods-operator
spec:
  name: rhods-operator
  channel: ${CHANNEL_VERSION}
  source: cs-redhat-operator-index
  sourceNamespace: openshift-marketplace
  config:
     env:
        - name: "DISABLE_DSC_CONFIG"
EOF
```

Check the status of the rhods-operator-* pod in the redhat-ods-operator project:
```
oc get pods -n redhat-ods-operator
```

Confirm that the pod is Running. The command returns a response with the following format:
```
NAME                              READY   STATUS    RESTARTS   AGE
rhods-operator-56c85d44c9-vtk74   1/1     Running   0          3h57m
```

### Create a DSC Initialization (DSCInitialization) object in the redhat-ods-monitoring project

```
cat <<EOF |oc apply -f -
apiVersion: dscinitialization.opendatahub.io/v1
kind: DSCInitialization
metadata:
  name: default-dsci
spec:
  applicationsNamespace: redhat-ods-applications
  monitoring:
    managementState: Managed
    namespace: redhat-ods-monitoring
  serviceMesh:
    managementState: Removed
  trustedCABundle:
    managementState: Managed
    customCABundle: ""
EOF
```

Check the phase of the DSC Initialization (DSCInitialization) object:
```
oc get dscinitialization
```

Confirm that the object is Ready. The command returns a response with the following format:
```
NAME           AGE     PHASE
default-dsci   4d18h   Ready
```
### Create a Data Science Cluster (DataScienceCluster) object
```
cat <<EOF |oc apply -f -
apiVersion: datasciencecluster.opendatahub.io/v1
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  components:
    codeflare:
      managementState: Removed
    dashboard:
      managementState: Removed
    datasciencepipelines:
      managementState: Removed
    kserve:
      managementState: Managed
      defaultDeploymentMode: RawDeployment
      serving:
        managementState: Removed
        name: knative-serving
    kueue:
      managementState: Removed
    modelmeshserving:
      managementState: Removed
    ray:
      managementState: Removed
    trainingoperator:
      managementState: Managed
    trustyai:
      managementState: Removed
    workbenches:
      managementState: Removed
EOF
```
The Red Hat OpenShift AI Operator installs and manages the services that are listed as Managed. Services that are Removed are not installed.

Wait for the Data Science Cluster object to be Ready.

To check the status of the object, run:
```
oc get datasciencecluster default-dsc -o jsonpath='"{.status.phase}" {"\n"}'
```

Confirm that the status of the following pods in the redhat-ods-applications project are Running: 
<br>

- kserve-controller-manager-* pod
- kubeflow-training-operator-* pod
- odh-model-controller-* pod

```
oc get pods -n redhat-ods-applications
```
The command returns a response with the following format:
```
NAME                                         READY   STATUS      RESTARTS   AGE
kserve-controller-manager-57796d5b44-sh9n5   1/1     Running     0          4m57s
kubeflow-training-operator-7b99d5584c-rh5hb  1/1     Running     0          4m57s
```

### Edit the inferenceservice-config configuration map in the redhat-ods-applications project:
- Log in to the Red Hat OpenShift Container Platform web console as a cluster administrator.
- From the navigation menu, select Workloads > Configmaps.
- From the Project list, select redhat-ods-applications.
- Click the inferenceservice-config resource. Then, open the YAML tab.
<br>
In the metadata.annotations section of the file, add `opendatahub.io/managed: 'false'`:
```
    metadata:
      annotations:
        internal.config.kubernetes.io/previousKinds: ConfigMap
        internal.config.kubernetes.io/previousNames: inferenceservice-config
        internal.config.kubernetes.io/previousNamespaces: opendatahub
        opendatahub.io/managed: 'false'
```

Find the following entry in the file:
```
"domainTemplate": "{{ .Name }}-{{ .Namespace }}.{{ .IngressDomain }}",
```

Update the value of the domainTemplate field to "example.com":
```
"domainTemplate": "example.com",
```

Click Save.

End of document
