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

## Start the mirroring

**Note:** The `159.33.188.34:8080` in below commands needs to be changed to your private image registry location accordingly.

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

End of document
