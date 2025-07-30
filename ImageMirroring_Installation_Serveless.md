# Images mirroring and Installation - OpenShift Serverless


---

## Setting up a client workstation

### Set the installation directory

***Note***:
<br>
You can change the directory path as you want.

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

Create an ImageSetConfiguration file `ocp_serverless_416.yaml` for Serverless Knative Eventing as below:
***Note***: The `159.33.188.34:8080` in the `registry` section needs to be changed to your private image registry location accordingly.

```
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
archiveSize: 4                                                      
storageConfig:                                                      
  registry:
    imageURL: 159.33.188.34:8080/cp               
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
    - name: serverless-operator                                     
      channels:
      - name: stable                                        
  additionalImages:
  - name: registry.redhat.io/ubi9/ubi:latest                        
  helm: {}
```

## Mirror images

**Note:** The `169.63.179.172:8080` in below commands needs to be changed to your private image registry location accordingly.

```
cd $WXO_INSTALL_DIR
oc mirror --config=./ocp_serverless_416.yaml docker://169.63.179.172:8080 --source-skip-tls --dest-skip-tls --verbose 9
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
## Installing the OpenShift Serverless Knative Eventing

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

### Install IBM Events Operator

1. Log into OpenShift cluster for the cpd-cli.

```bash
${CPDM_OC_LOGIN}
```
2.Authorize the projects where the software will be installed to communicate:
```
cpd-cli manage authorize-instance-topology \
--release=${VERSION} \
--cpd_operator_ns=ibm-knative-events \
--cpd_instance_ns=knative-eventing
```
3.Install the IBM Events Operator in the ibm-knative-events project:
```
cpd-cli manage setup-instance-topology \
--release=${VERSION} \
--cpd_operator_ns=ibm-knative-events \
--cpd_instance_ns=knative-eventing \
--block_storage_class=${STG_CLASS_BLOCK} \
--license_acceptance=true
```

4.Install the Red Hat OpenShift Serverless Knative Eventing and IBM Events software.

```bash
cpd-cli manage deploy-knative-eventing \
--release=${VERSION} \
--block_storage_class=${STG_CLASS_BLOCK}
```

## Troubleshooting
#### 1. "knativeeventings.operator.knative.dev" not found
**Symptom**
<br>
During running the step 4 `Install the Red Hat OpenShift Serverless Knative Eventing and IBM Events software` in the `Install IBM Events Operator`, you may encounter the command failure with the error message like below.
```
Error from server (NotFound): customresourcedefinitions.apiextensions.k8s.io "knativeeventings.operator.knative.dev" not found
```
**Resolution**
<br>
Check whether the serverless csv was created successfully.

```
oc get csv -n openshift-serverless | grep -i serverless-operator
```

The result should look like below.

```
serverless-operator.v1.36.0         Red Hat OpenShift Serverless   1.36.0    serverless-operator.v1.35.0   Succeeded
```

If no result found, then it means there's something wrong when creating the Serverless operator. It's recommended [installing the Serverless operator from the OpenShift web console](https://docs.redhat.com/en/documentation/red_hat_openshift_serverless/1.35/html/installing_openshift_serverless/install-serverless-operator#serverless-install-web-console_install-serverless-operator).
<br>
**Note**:
<br>
- Ensure the correct catalog source is used accordingly.
- Check if the installplan is set to be Automatic. If not, you may have to approval it.

After the operator installed successfully, then rerun the command `cpd-cli manage deploy-knative-eventing`


---

End of document
