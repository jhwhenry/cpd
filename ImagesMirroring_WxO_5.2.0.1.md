# Images mirroring for watsonx Orchestrate 5.2.0.1

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

Download Version `14.2.0 Refresh 1` of the cpd-cli from the IBM/cpd-cli repository on GitHub.

#### 1.2.1 Download with wget

```
mkdir -p $WXO_INSTALL_DIR
cd $WXO_INSTALL_DIR
wget https://github.com/IBM/cpd-cli/releases/download/v14.2.0_refresh_1/cpd-cli-linux-EE-14.2.0.tgz
```

#### 1.2.2 Extract the tar file

```
tar -xvf cpd-cli-linux-EE-14.2.0.tgz
```

#### 1.2.3 Make the cpd-cli executable from any directory.

```
export PATH=$WXO_INSTALL_DIR/cpd-cli-linux-EE-14.2.0-2124:$PATH
```

Validate with the following command
```
cpd-cli version
```
The output looks like below. <br>

**Note**: The build number is `2124`

```
cpd-cli
        Version: 14.2.0
        Build Date: 2025-06-23T13:49:48
        Build Number: 2124
        CPD Release Version: 5.2.0
```

#### 1.2.4 Restart the olm-utils container
```
cpd-cli manage restart-container
```

## 2 Update the existing environment variables file

- Make sure the `VERSION` and the private image registry information are specified properly.
- Replace `watson_assistant` with `watsonx_orchestrate` in the `COMPONENTS` environment varibale.
- Add the 'IMAGE_GROUPS' environment variable

```
#Choose the Foundation models to be usedfor watsonx Orchestrate
export IMAGE_GROUPS=ibmwxSlate30mEnglishRtrvr
```
- Source the environment variable file.

```
source cpd_vars.sh
```

## 3 Mirroring images directly to the private container registry

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

### 3.3 Downloading CASE packages
- Backup existing watsonx Orchestrate CASE folder.
```
mv $(podman inspect olm-utils-play-v3 | jq -r '.[0].Mounts[0].Source')/offline/${VERSION}/.ibm-pak/data/cases/ibm-watsonx-orchestrate $(podman inspect olm-utils-play-v3 | jq -r '.[0].Mounts[0].Source')/offline/${VERSION}/.ibm-pak/data/cases/ibm-watsonx-orchestrate_5200
```
- Download the CASE files of watsonx Orchestrate 5.2.0.1.
```
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION}
```
- Validate the CASE files of watsonx Orchestrate 5.2.0.1 have been downloaded successfully.
```
ls -alrt $(podman inspect olm-utils-play-v3 | jq -r '.[0].Mounts[0].Source')/offline/${VERSION}/.ibm-pak/data/cases/ibm-watsonx-orchestrate/6.0.0 | grep -i ibm-watsonx-orchestrate-6.0.0+20250722.201300

```
The output looks like below.
```
-rw-r--r--. 1 itzuser root  37086 Aug  3 00:56 ibm-watsonx-orchestrate-6.0.0+20250722.201300-images.csv
-rw-r--r--. 1 itzuser root     32 Aug  3 00:56 ibm-watsonx-orchestrate-6.0.0+20250722.201300-charts.csv
-rw-r--r--. 1 itzuser root   1274 Aug  3 00:56 ibm-watsonx-orchestrate-6.0.0+20250722.201300-airgap-metadata.yaml
-rw-r--r--. 1 itzuser root 491147 Aug  3 00:56 ibm-watsonx-orchestrate-6.0.0+20250722.201300.tgz
```

### 3.4 Mirror the images to the private container registry

- Mirror the images without specifying the image group
<br>
```
cpd-cli manage mirror-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--case_download=false \
--retry_count=2 --retry_delay=5 -v
```

- Mirror the images for the foundation model (slate-30m-english-rtrvr)
```
cpd-cli manage mirror-images \
--components=watsonx_ai_ifm \
--groups=${IMAGE_GROUPS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--case_download=false \
--retry_count=2 --retry_delay=5 -v
```

- Validate the image mirroring
 
<br>

For each component, the command generates a log file in the `work` directory. 

<br>

1)Run the following command to validate if any errors in the log files:

```
grep "error" $(podman inspect olm-utils-play-v3 | jq -r '.[0].Mounts[0].Source')/mirror_*.log
```

2)Validate if the `slate-30m-english-rtrvr` foundation model image mirrored.

```
cat $(podman inspect olm-utils-play-v3 | jq -r '.[0].Mounts[0].Source')/mirror_ibm-watsonx-ai-ifm.log | grep -i a5e595fe75ae7d6a59bd9750fd88a283cd12fec5a643a2d1d16834298ea9f3b6
```

The output looks like below.

```
sha256:a5e595fe75ae7d6a59bd9750fd88a283cd12fec5a643a2d1d16834298ea9f3b6 -> 5.2.0-202505122243
```

3)Confirm that all the required images were mirrored to the private container registry:

```
cpd-cli manage list-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--case_download=false
```

The output is saved to the `list_images.csv` file in the `work/offline/${VERSION}` directory. 
<br>

Make sure the `list_images.csv` exists.

```
ls $(podman inspect olm-utils-play-v3 | jq -r '.[0].Mounts[0].Source')/offline/${VERSION}
```

Check the output for errors: 

```
grep "level=fatal" $(podman inspect olm-utils-play-v3 | jq -r '.[0].Mounts[0].Source')/offline/${VERSION}/list_images.csv
```

The command returns images that failed because of authorization errors or network errors. If no result returned, it means the images were mirrored successfully.

---

End of document
