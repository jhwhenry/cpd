# Images mirroring - OpenShift AI

---

## Setting up a client workstation

### Install the oc client and oc-mirror plugin

#### 1.Download with wget

```
mkdir /root/mirroring
cd /root/mirroring
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.16.40/openshift-client-linux-4.16.40.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.16.40/oc-mirror.tar.gz
```

#### 2.Extract the tar file

```
tar -xvf openshift-client-linux-4.16.40.tar.gz
tar -xvf oc-mirror.tar.gz
```

#### 3.Make the oc client executable everywhere 

```
cp oc /usr/bin/oc
chmod +x oc-mirror
cp oc-mirror /usr/bin/oc-mirror
```

Validate with the following command

```
oc version
oc mirror help
```


### Configure the oc-mirror credentials

[Configuring credentials that allow images to be mirrored](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/disconnected_installation_mirroring/installing-mirroring-disconnected#installation-adding-registry-pull-secret_installing-mirroring-disconnected)


## Build the ImageSetConfiguration file 

Create an ImageSetConfiguration file `ocp_ai_416.yaml` for OpenShift AI 2.16 as below:
***Note***: The `169.63.188.34:8080` in the `registry` section needs to be changed to your private image registry location accordingly.

```
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
archiveSize: 4                                                      
storageConfig:                                                      
  registry:
    imageURL: 169.63.188.34:8080/cp               
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
      - name: "stable-2.16"
        minVersion: "2.16.2"                                             
  additionalImages:
  - name: registry.redhat.io/ubi9/ubi:latest                        
  helm: {}
```

## Start the mirroring

***Note***: The `169.63.188.34:8080` in below commands needs to be changed to your private image registry location accordingly.

```
cd /root/mirroring
oc mirror --config=./ocp_ai_416.yaml docker://169.63.188.34:8080 --source-skip-tls --dest-skip-tls --verbose 9
```

Once complete, there’s a folder named oc-mirror-workspace created looks like below:

```
ls oc-mirror-workspace
# Output:
publish results-1750641947

ls oc-mirror-workspace/results-1750641947
# Output:
catalogSource-cs-redhat-operator-index.yaml  charts  imageContentSourcePolicy.yaml  mapping.txt  release-signatures  updateService.yaml
```

## Apply the ImageDigestMirrorSet

#### 1.Login the OCP cluster using the cluster admin role

```
oc login https://api.685849c881da5dd628552ad6.am1.xxx.yyy.com:6443 -u kubeadmin -p DP73S-iSmys-SuQ6j-9Ax8f
```

#### 2)Apply the ImageDigestMirrorSet

```
cd /root/mirroring/oc-mirror-workspace/results-1750641947

oc create -f $(oc adm migrate icsp imageContentSourcePolicy.yaml | cut -f 4 -d ' ')
```

#### 3)Validate

Check if the ImageDigestMirrorSet object created as expected.

```
oc get ImageDigestMirrorSet
```

Wait until all the node are up and running.
```
watch -n 10 "oc get nodes,mcp"
```

---

End of document
