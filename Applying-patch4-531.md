# Applying Patch 4 to IBM Software Hub 5.3.1

---

## Context
- **OCP:** 4.20
- **CPD:** 5.3.1 → 5.3.1 Patch 4
- **Storage:** NetApp Trident
- **Components:** cpd_platform,watsonx_data,watsonx_ai,watsonx_orchestrate,spss,dods,datastage_ent
- **Airgapped:** Yes

# Table of Contents
- 1. Pre-patch procedures
- 2. Patch steps
- 3. Post-patch tasks

# Prerequisites

### 1. Backup of the cluster is done.

Backup your Cloud Pak for Data cluster before the upgrade.

**Note:**
Make sure there are no scheduled backups conflicting with the scheduled upgrade.

### 2. The image mirroring completed successfully

If a private container registry is in-use to host the IBM Software Hub software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry. 

### 3. The CASE files and cluster resource files downloaded successfully

Before upgrading IBM Software Hub platform or any services, you must download the required cluster‑scoped resources, such as ClusterRoles and ClusterRoleBindings—for the components you plan to upgrade. Ensure that these files are available on the bastion node for use during the upgrade.

### 4. The permissions required for the upgrade is ready

- Openshift cluster permissions
<br>An Openshift cluster administrator can complete all of the installation tasks.

- IBM Software Hub permissions
<br>The Cloud Pak for Data administrator role or permissions is required for upgrading the service instances.

- Permission to access the private image registry for pushing or pull images

- Red Hat pull secret for mirroring images of Red Hat Cert Manager

- Access to the Bastion node for executing the upgrade commands

### 5. A pre-patch health check is made to ensure the cluster's readiness for upgrade.

- The OpenShift cluster, persistent storage, IBM Software Hub platform and services are in healthy status.

## Pre-patch procedures

### Prepare cpd_vars.sh

Follow the IBM guidance at [Setting up installation environment variables](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=information-setting-up-installation-environment-variables) to prepare `cpd_vars.sh`. 

If an existing `cpd_vars.sh` file was used for the previous upgrade to 5.3.1, it can be reused.

Source the script in every new shell session:

```bash
source ./cpd_vars.sh
```

### Refresh the olm-utils container
Mirror the latest `olm-utils-v4` image.

```
podman pull icr.io/cpopen/cpd/olm-utils-v4:${VERSION} --tls-verify=false

podman login ${PRIVATE_REGISTRY_LOCATION} -u ${PRIVATE_REGISTRY_PUSH_USER} -p ${PRIVATE_REGISTRY_PUSH_PASSWORD}

podman tag icr.io/cpopen/cpd/olm-utils-v4:${VERSION} ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v4:${VERSION}

podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v4:${VERSION}
```

Restart the olm-utils container for using the latest `olm-utils-v4` image.

```bash
cpd-cli manage restart-container
```

### 2. Gather information about installed components

**Reference:** [Gathering information about installed components](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-gathering-information-about-installed-components)

Log in to OpenShift:

```bash
${CPDM_OC_LOGIN}
```

Generate the list of installed components:

```bash
cpd-cli manage list-deployed-components \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --all=true
```

Copy the generated `scan.yaml` for reference:

```bash
cp ${CPD_CLI_MANAGE_WORKSPACE}/work/scan/scan.yaml \
   ${CPD_CLI_MANAGE_WORKSPACE}/work/${PROJECT_CPD_INST_OPERANDS}-scan.yaml
```

### Step 4: List available patches

```bash
cpd-cli manage list-patch
```

Example output:

```
# To download CASE Packages for patches, run:
# cpd-cli manage case-download --components=cpd_platform,zen --release=5.3.1
```

### Step 5: Download CASE packages

**Reference:** [Downloading CASE packages](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-downloading-case-packages)

> ⚠️ **Important:** The command below is an example of the command **shape** only. The actual component list must be extracted from the output of the `list-patch` command in Step 4. Always append `--patch_download=false`.

```bash
cpd-cli manage case-download \
  --components=cpd_platform,zen \
  --release=5.3.1 \
  --patch_download=false
```

Set `COMPONENTS_TO_PATCH` to the same list of components used above:

```bash
export COMPONENTS_TO_PATCH=cpd_platform,zen
```

### Step 6: Mirror images directly to the private container registry

**Reference:** [Mirroring images directly to a private container registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-3)

Log in to the IBM Entitled Registry:

```bash
cpd-cli manage login-entitled-registry \
  ${IBM_ENTITLEMENT_KEY}
```

Log in to the private container registry:

```bash
cpd-cli manage login-private-registry \
  ${PRIVATE_REGISTRY_LOCATION} \
  ${PRIVATE_REGISTRY_PUSH_USER} \
  ${PRIVATE_REGISTRY_PUSH_PASSWORD}
```

Inspect the source (Entitled) registry to confirm image access.

> ⚠️ **Important:** Add `--patch_download=false` to the `list-images` command.

```bash
cpd-cli manage list-images \
  --components=${COMPONENTS_TO_PATCH} \
  --release=5.3.1 \
  --inspect_source_registry=true \
  --patch_download=false
```

Check for errors:

```bash
grep "level=fatal" ${CPD_CLI_MANAGE_WORKSPACE}/work/offline/${VERSION}/list_images.csv
```

Mirror the images to the private container registry.

> ⚠️ **Important:** Add `--patch_download=false` to the `mirror-images` command.

```bash
cpd-cli manage mirror-images \
  --components=${COMPONENTS_TO_PATCH} \
  --release=5.3.1 \
  --target_registry=${PRIVATE_REGISTRY_LOCATION} \
  --arch=${IMAGE_ARCH} \
  --case_download=false \
  --patch_download=false
```

Check mirror logs for errors:

```bash
grep "error" ${CPD_CLI_MANAGE_WORKSPACE}/work/mirror_*.log
```

Verify the images landed in the private registry.

> ⚠️ **Important:** Add `--patch_download=false` to the `list-images` command.

```bash
cpd-cli manage list-images \
  --components=${COMPONENTS_TO_PATCH} \
  --release=5.3.1 \
  --target_registry=${PRIVATE_REGISTRY_LOCATION} \
  --case_download=false \
  --patch_download=false

grep "level=fatal" ${CPD_CLI_MANAGE_WORKSPACE}/work/offline/${VERSION}/list_images.csv
```

### Step 7: Update cluster-scoped resources

**Reference:** [Updating cluster-scoped resources for the instance](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-updating-cluster-scoped-resources-instance)

> ⚠️ **Important:** Append `--patch_download=false`

Generate `cluster_scoped_resources.yaml`:

```bash
cpd-cli manage case-download \
  --components=${COMPONENTS_TO_PATCH} \
  --release=5.3.1 \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --case_download=false \
  --cluster_resources=true \
  --patch_download=false
```

Log in to OpenShift as a cluster administrator:

```bash
${OC_LOGIN}
```

Apply the cluster-scoped resources directly from the workspace path:

```bash
oc apply -f ${CPD_CLI_MANAGE_WORKSPACE}/work/cluster_scoped_resources.yaml \
  --server-side \
  --force-conflicts
```

> **Note:** IBM documentation references `cpd-cli-workspace/olm-utils-workspace/work` as the default location for the work directory. In practice, when `CPD_CLI_MANAGE_WORKSPACE` is set, the file is generated at `${CPD_CLI_MANAGE_WORKSPACE}/work/cluster_scoped_resources.yaml`. The command above applies it directly from that path.

(Optional) Archive the resource file for the record:

```bash
mv ${CPD_CLI_MANAGE_WORKSPACE}/work/cluster_scoped_resources.yaml \
   ${CPD_CLI_MANAGE_WORKSPACE}/work/${VERSION}-PATCH-${PROJECT_CPD_INST_OPERATORS}-cluster_scoped_resources.yaml
```

### Step 8: Verify the staged patch manifest before applying

Before running `apply-patch`, confirm that the staged `patch-5.3.1.yaml` corresponds to the intended patch level. For Patch 1, you should see `patch_id: 1`.

```bash
cat ${CPD_CLI_MANAGE_WORKSPACE}/work/offline/patch/patch-5.3.1.yaml
```

Expected output (excerpt) for Patch 1:

```yaml
patch_components_meta:
  patch_info:
    patch_id: 1
    patch_date: "2026-03-24"
```

> ⚠️ **Important:** If `patch_id` does not match the target patch level, stop and re-stage the correct manifest from the correct Git tag (see Step 2). Do not run `apply-patch` with the wrong manifest.
>

## Pre-patch procedures

### Step 9: Apply the patch to the instance

**Reference:** [Applying the patch to an instance](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-applying-patch)

Apply the patch. Use the variant that matches your instance:

#### Without tethered projects

```bash
cpd-cli manage apply-patch \
  --release=5.3.1 \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET}
```

#### With tethered projects

```bash
cpd-cli manage apply-patch \
  --release=5.3.1 \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --tethered_instance_ns=${PROJECT_CPD_INSTANCE_TETHERED_LIST} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET}
```

Wait for:

```
[SUCCESS] ... The apply-patch command ran successfully.
```

Confirm all operands report `Completed`:

```bash
cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

### Step 10: Update service instances (Patch 1 specific)

**Reference:** [Updating service instances after applying the patch](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-updating-service-instances)

Based on the Patch 1 manifest, the following services require a manual service-instance upgrade after `apply-patch` completes.

> ⚠️ **Important: Only run the subset that applies to services installed in the customer's environment.**

> **Note:** Create a `cpd-cli` profile for a user with `can_provision` or `manage_service_instances` permissions, then export `CPD_PROFILE_NAME` before running the commands below.

#### Analytics Engine powered by Apache Spark

```bash
cpd-cli service-instance upgrade \
  --service-type=spark \
  --profile=${CPD_PROFILE_NAME} \
  --all
```

#### Data Gate

Upgrade all Data Gate service instances at the same time:

```bash
cpd-cli service-instance upgrade \
  --service-type=dg \
  --profile=${CPD_PROFILE_NAME} \
  --all
```

Verify the upgrade completed (status should be `UPGRADED`):

```bash
cpd-cli service-instance list \
  --service-type=dg \
  --profile=${CPD_PROFILE_NAME}
```

#### OpenPages

Upgrade all OpenPages service instances (repeat per-instance if you prefer individual control):

```bash
cpd-cli service-instance upgrade \
  --service-type=openpages \
  --profile=${CPD_PROFILE_NAME} \
  --all \
  --force-version-upgrade=true
```

#### watsonx BI

For Patch 1, the target service instance version is 3.4.1.

Get the watsonx BI service instance name:

```bash
cpd-cli service-instance list \
  --service-type=watsonx-bi-assistant \
  --profile=${CPD_PROFILE_NAME}
```

Set the instance name and version, then upgrade:

```bash
export BI_INSTANCE_NAME=<service-instance-name>
export BI_INSTANCE_VERSION=3.4.1

cpd-cli service-instance upgrade \
  --service-type=watsonx-bi-assistant \
  --instance-name=${BI_INSTANCE_NAME} \
  --version=${BI_INSTANCE_VERSION} \
  --profile=${CPD_PROFILE_NAME}
```

> **Note:** Cognos Analytics, Data Virtualization, Db2, Db2 Warehouse, and Db2 Big SQL are **not** included in Patch 1 and do not require a service-instance upgrade for this patch level.

## Reference: Commands requiring --patch_download=false

| Command | Phase | Required |
|---|---|---|
| `list-patch` | Patch discovery | Yes |
| `case-download` | Patch download | Yes |
| `list-images` | Patch mirror | Yes |
| `mirror-images` | Patch mirror | Yes |
| `apply-cluster-components` | Install only | Yes |
| `apply-scheduler` | Install only | Yes |
| `apply-patch` | Patch apply | No (reads staged manifest) |

## References

- [Applying patches to IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading-applying-patches)
- [Gathering information about installed components](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-gathering-information-about-installed-components)
- [Checking for new patches](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-checking-new)
- [Downloading CASE packages](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-downloading-case-packages)
- [Mirroring images to a private container registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-mirroring-images-private-container-registry)
- [Mirroring images directly to a private container registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-3)
- [Mirroring images using an intermediary container registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-using-intermediary-container-registry-3)
- [Updating cluster-scoped resources](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-updating-cluster-scoped-resources-instance)
- [Applying the patch to an instance](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-applying-patch)
- [Updating service instances after applying the patch](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-updating-service-instances)
- [Setting up installation environment variables](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=information-setting-up-installation-environment-variables)
