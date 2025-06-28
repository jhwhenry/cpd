# IKC Premium Installation 5.2.0

---

## Setting up a client workstation
### Set the installation directory

***Note***:
<br>
You can change the directory path if needed.

```
export IKC_PREMIUM_INSTALL_DIR=/opt/ibm/ikc_premium
```
### Installing the IBM Cloud Pak for Data command-line interface

Download Version 14.2.0 of the cpd-cli from the IBM/cpd-cli repository on GitHub <br>

#### 1.Download with wget

```
mkdir -p $IKC_PREMIUM_INSTALL_DIR
cd $IKC_PREMIUM_INSTALL_DIR
wget https://github.com/IBM/cpd-cli/releases/download/v14.2.0/cpd-cli-linux-EE-14.2.0.tgz
```

#### 2.Extract the tar file

```
tar -xvf cpd-cli-linux-EE-14.2.0.tgz
```

#### 3.Make the cpd-cli executable from any directory.

```
export PATH=$IKC_PREMIUM_INSTALL_DIR/cpd-cli-linux-EE-14.2.0-2081:$PATH
```

Validate with the following command
```
cpd-cli version
```

## Creating an environment variables file

Create the cpd_vars.sh shell script with file content like below. <br>

***Note***:
<br>
Change the Cluster information 


```
#===============================================================================
# Cloud Pak for Data installation variables
#===============================================================================

# ------------------------------------------------------------------------------
# Cluster
# ------------------------------------------------------------------------------

export OCP_URL=<enter your Red Hat OpenShift Container Platform URL>
export OPENSHIFT_TYPE=self-managed
export IMAGE_ARCH=amd64
export OCP_USERNAME=<enter your username>
export OCP_PASSWORD=<enter your password>
# export OCP_TOKEN=sha256~JkYN46_DwXlQ4bWCOwu_rfWq_RghLsUqMCY_rkGkv4U
export SERVER_ARGUMENTS="--server=${OCP_URL}"
export LOGIN_ARGUMENTS="--username=${OCP_USERNAME} --password=${OCP_PASSWORD}"
export CPDM_OC_LOGIN="cpd-cli manage login-to-ocp ${SERVER_ARGUMENTS} ${LOGIN_ARGUMENTS}"
export OC_LOGIN="oc login ${OCP_URL} ${LOGIN_ARGUMENTS}"


# ------------------------------------------------------------------------------
# Projects
# ------------------------------------------------------------------------------

export PROJECT_CERT_MANAGER=ibm-cert-manager
export PROJECT_LICENSE_SERVICE=ibm-licensing
export PROJECT_SCHEDULING_SERVICE=ibm-cpd-scheduler
export PROJECT_IBM_EVENTS=ibm-knative-events
# export PROJECT_PRIVILEGED_MONITORING_SERVICE=<enter your privileged monitoring service project>
export PROJECT_CPD_INST_OPERATORS=cpd-operators
export PROJECT_CPD_INST_OPERANDS=cpd

# ------------------------------------------------------------------------------
# Storage
# ------------------------------------------------------------------------------

export STG_CLASS_BLOCK=ocs-storagecluster-ceph-rbd
export STG_CLASS_FILE=ocs-storagecluster-cephfs

export IBM_ENTITLEMENT_KEY=<Your IBM Entitlement Key>

# ------------------------------------------------------------------------------
# Private container registry
# ------------------------------------------------------------------------------
# Set the following variables if you mirror images to a private container registry.
#
# To export these variables, you must uncomment each command in this section.

export PRIVATE_REGISTRY_LOCATION=<enter the location of your private container registry>
export PRIVATE_REGISTRY_PUSH_USER=<enter the username of a user that can push to the registry>
export PRIVATE_REGISTRY_PUSH_PASSWORD=<enter the password of the user that can push to the registry>
export PRIVATE_REGISTRY_PULL_USER=<enter the username of a user that can pull from the registry>
export PRIVATE_REGISTRY_PULL_PASSWORD=<enter the password of the user that can pull from the registry>

# ------------------------------------------------------------------------------
# IBM Software Hub version
# ------------------------------------------------------------------------------

export VERSION=5.2.0

# ------------------------------------------------------------------------------

# Components

# ------------------------------------------------------------------------------

# Set the following variable if you want to install or upgrade multiple components at the same time.

#

# To export the variable, you must uncomment the command.

export COMPONENTS=ibm-licensing,cpfs,cpd_platform,ikc_premium,datastage_ent,datalineage

```

## Source the environment variable

```
sourc cpd_vars.sh
```

## Configuring image content source policy

Run the following command to get the list of image content source policy resources:

```
oc get imageContentSourcePolicy -o yaml > icsp.yaml
oc get ImageDigestMirrorSet -o yaml > idms.yaml
```

If the command returns No resources found, then run below commands:
```
${CPDM_OC_LOGIN}

cpd-cli manage apply-icsp \
--registry=${PRIVATE_REGISTRY_LOCATION}
```

Get the status of the nodes:
```
oc get nodes
```

Wait until all the nodes are Ready before you proceed to the next step. 

## Updating the global image pull secret
This step may take minutes to complete.

```
cpd-cli manage add-cred-to-global-pull-secret \
--registry=${PRIVATE_REGISTRY_LOCATION} \
--registry_pull_user=${PRIVATE_REGISTRY_PULL_USER} \
--registry_pull_password=${PRIVATE_REGISTRY_PULL_PASSWORD}
```

Get the status of the nodes:

```
oc get nodes
```

Wait until all the nodes are Ready before you proceed to the next step. 


## Changing the process IDs limit

Check whether there is an existing kubeletconfig on the cluster:

```
oc get kubeletconfig
```

If the above command returns the name of a kubeletconfig:

```
export KUBELET_CONFIG=<kubeletconfig-name>

oc patch kubeletconfig ${KUBELET_CONFIG} \
--type=merge \
--patch='{"spec":{"kubeletConfig":{"podPidsLimit":16384}}}'
```

If no kubeletconfig found:

```
oc apply -f - << EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: cpd-kubeletconfig
spec:
  kubeletConfig:
    podPidsLimit: 16384
  machineConfigPoolSelector:
    matchExpressions:
    - key: pools.operator.machineconfiguration.openshift.io/worker
      operator: Exists
EOF
```

## Changing load balancer timeout settings
Increasing the load balancer timeout settings prevents connections from being closed before processes complete. <br>

The minimum recommended timeout is:
```
    Client timeout: 1800s (30m)
    Server timeout: 1800s (30m)
```

[Reference] (https://www.ibm.com/docs/en/software-hub/5.2.x?topic=settings-changing-load-balancer)




## Mirroring images directly to a private container registry

From a client workstation that can connect to the internet: 

### Log in to the IBM Entitled Registry : 

```
cpd-cli manage login-entitled-registry \
${IBM_ENTITLEMENT_KEY}
```

### Log in to the private container registry:

```
cpd-cli manage login-private-registry \
${PRIVATE_REGISTRY_LOCATION} \
${PRIVATE_REGISTRY_USER} \
${PRIVATE_REGISTRY_PASSWORD}
```

### Mirror the images to the private container registry

```
cpd-cli manage mirror-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--arch=amd64 \
--case_download=true
```

For each component, the command generates a log file in the work directory. 

<br>

Run the following command to print out any errors in the log files:

```
grep "error" mirror_*.log
```

Confirm that the images were mirrored to the private container registry:

```

cpd-cli manage list-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--case_download=false
```

Check the output for errors: 

```
grep "level=fatal" list_images.csv
```


---

End of document
