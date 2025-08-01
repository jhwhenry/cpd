# Images mirroring for watsonx Orchestrate 5.2.0

---

## 1 Setting up a client workstation

### 1.1 Set the installation directory

**Note**:
<br>
You can change the directory path if necessary.

```
export WXO_INSTALL_DIR=/opt/ibm/wxo
```
### 1.2 Installing the IBM Cloud Pak for Data command-line interface

Download Version 14.2.0 of the cpd-cli from the IBM/cpd-cli repository on GitHub <br>

#### 1.2.1 Download with wget

```
mkdir -p $WXO_INSTALL_DIR
cd $WXO_INSTALL_DIR
wget https://github.com/IBM/cpd-cli/releases/download/v14.2.0/cpd-cli-linux-EE-14.2.0.tgz
```

#### 1.2.2 Extract the tar file

```
tar -xvf cpd-cli-linux-EE-14.2.0.tgz
```

#### 1.2.3 Make the cpd-cli executable from any directory.

```
export PATH=$WXO_INSTALL_DIR/cpd-cli-linux-EE-14.2.0-2081:$PATH
```

Validate with the following command
```
cpd-cli version
```

#### 1.2.4 Restart the olm-utils container
```
cpd-cli manage restart-container
```

## 2 Creating an environment variables file

Create the cpd_vars.sh shell script with below file content.

```

#===============================================================================

# ------------------------------------------------------------------------------

export VERSION=5.2.0

# ------------------------------------------------------------------------------

export IBM_ENTITLEMENT_KEY=<your IBM Entitlement key>

# ------------------------------------------------------------------------------
# Private image registry information
# ------------------------------------------------------------------------------

export PRIVATE_REGISTRY_LOCATION=<your private image registry location>
export PRIVATE_REGISTRY_USER=<The user name for logging into the private image registry>
export PRIVATE_REGISTRY_PASSWORD=<The password for logging into the private image registry>

# ------------------------------------------------------------------------------

# Components

# ------------------------------------------------------------------------------

export COMPONENTS=ibm-licensing,scheduler,cpfs,cpd_platform,watsonx_orchestrate

#Choose the Foundation models to be usedfor watsonx Orchestrate
#https://www.ibm.com/docs/en/software-hub/5.2.x?topic=information-determining-which-models-optional-images-mirror#mirror-model-list__orchestrate-models__title__1
export IMAGE_GROUPS=ibmwxSlate30mEnglishRtrvr
```

Source the environment variable file.

```
chmod +x cpd_vars.sh
source cpd_vars.sh
```

## 3 Mirroring images directly to a private container registry

From a client workstation that can connect to the internet: 

### 3.1 Log in to the IBM Entitled Registry

```
cpd-cli manage login-entitled-registry \
${IBM_ENTITLEMENT_KEY}
```

### 3.2 Log in to the private container registry:

```
cpd-cli manage login-private-registry \
${PRIVATE_REGISTRY_LOCATION} \
${PRIVATE_REGISTRY_USER} \
${PRIVATE_REGISTRY_PASSWORD}
```

### 3.3 Mirror the images to the private container registry

```
cpd-cli manage mirror-images \
--components=${COMPONENTS} \
--groups=${IMAGE_GROUPS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--arch=amd64 \
--case_download=false
```

For each component, the command generates a log file in the `work` directory. 

<br>

Run the following command to print out any errors in the log files:

```
grep "error" $(podman inspect olm-utils-play-v3 | jq -r '.[0].Mounts[0].Source')/mirror_*.log
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
