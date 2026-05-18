# Applying Patch4 to IBM Software Hub 5.3.1

> Pinning to a chosen patch level (for example, Patch 1) when a newer patch has already been released.

| | |
|---|---|
| **Audience** | CP4D / Software Hub administrators, IBM Software Support, Expert Labs |
| **Scope** | IBM Software Hub 5.3.1 GA → 5.3.1 Patch 1 (applies to any patch level by tag) |
| **Version** | 1.2 |
| **Authored by** | IBM Data & AI Tiger Team |

---

## Contents

- [Overview](#overview)
- [Background](#background)
  - [Solution: --patch_download=false](#solution---patch_downloadfalse)
  - [Scope of this document](#scope-of-this-document)
- [Assumptions](#assumptions)
- [Prerequisites](#prerequisites)
  - [1. Prepare cpd_vars.sh](#1-prepare-cpd_varssh)
  - [2. Identify the correct patch YAML URL](#2-identify-the-correct-patch-yaml-url)
- [Procedure](#procedure)
  - [Step 1: Gather information about installed components](#step-1-gather-information-about-installed-components)
  - [Step 2: Stage the pinned patch manifest](#step-2-stage-the-pinned-patch-manifest)
  - [Step 3: Refresh the olm-utils container](#step-3-refresh-the-olm-utils-container)
  - [Step 4: List available patches (pinned)](#step-4-list-available-patches-pinned)
  - [Step 5: Download CASE packages](#step-5-download-case-packages)
  - [Step 6: Mirror images directly to the private container registry](#step-6-mirror-images-directly-to-the-private-container-registry)
  - [Step 7: Update cluster-scoped resources](#step-7-update-cluster-scoped-resources)
  - [Step 8: Verify the staged patch manifest before applying](#step-8-verify-the-staged-patch-manifest-before-applying)
  - [Step 9: Apply the patch to the instance](#step-9-apply-the-patch-to-the-instance)
  - [Step 10: Update service instances (Patch 1 specific)](#step-10-update-service-instances-patch-1-specific)
- [Reference: Commands requiring --patch_download=false](#reference-commands-requiring---patch_downloadfalse)
- [References](#references)

---

## Overview

This guide describes how to apply a specific patch level (for example, Patch 1) to an existing IBM Software Hub 5.3.1 instance, rather than always jumping to the latest available patch.

## Background

By default, the `cpd-cli manage patch` commands resolve the patch manifest (`patch-5.3.1.yaml`) from the `main` branch of the IBM/software-hub GitHub repo, which always points to the latest patch. Most patch-related commands will also overwrite any locally staged `patch-5.3.1.yaml` with the latest version during execution.

This makes it impossible, using public documentation alone, to upgrade to a specific patch level (for example, Patch 1) after a newer patch (for example, Patch 2) has been released.

### Solution: --patch_download=false

To pin to a specific patch level, you must:

- Manually download the `patch-5.3.1.yaml` file from the Git tag corresponding to the desired patch (for example, `swhub-5.3.1-p1`).
- Pass `--patch_download=false` to every patch-related command so the manifest is not re-fetched from `main`.

The `--patch_download=false` flag must be added to all of the following commands:

- `list-patch`
- `case-download`
- `list-images`
- `mirror-images`
- `apply-cluster-components` (install-time only)
- `apply-scheduler` (install-time only)

> **Note:** This guide covers applying a patch to an existing 5.3.1 instance. The same principle applies for a fresh install pinned to a specific patch level. In that case, make sure `--patch_download=false` is added to `apply-cluster-components`, `apply-scheduler`, and `setup-instance` as well.

### Scope of this document

This document walks through applying 5.3.1 Patch 1 on top of an existing 5.3.1 GA instance, in a scenario where Patch 2 or later has already been released. The same process applies for any patch level, only the Git tag and YAML URL change.

## Assumptions

- Existing IBM Software Hub 5.3.1 GA instance is installed and healthy.
- Target patch level is 5.3.1 Patch 1 (tag: `swhub-5.3.1-p1`).
- The client workstation is set up per IBM documentation and has `cpd-cli`, `oc`, `helm`, and `skopeo` available.
- A private container registry is used, and images are mirrored directly from the IBM Entitled Registry to the private registry (no intermediary registry). For the intermediary-registry flow, see the IBM docs link at the end of this document.
- The `olm-utils-v4` image is pulled directly from the IBM Entitled Registry (`icr.io/cpopen/cpd/olm-utils-v4:5.3.1`). If your environment pulls `olm-utils-v4` from a private registry, you must re-mirror the image from the IBM Entitled Registry to your private registry to pick up the latest `olm-utils-v4` image before running `cpd-cli manage restart-container`.
- NamespaceScope operator and instance admin user already have sufficient RBAC from the 5.3.1 GA install. The minimum-RBAC re-authorization steps for the NSS operator and for the instance admin user are skipped in this guide.
- If you installed with minimum RBAC, complete those IBM-documented steps before applying the patch:
  - [Reauthorizing a user with minimum RBAC](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-reauthorizing-user-minimum-rbac)
  - [Reauthorizing the NamespaceScope operator with minimum RBAC](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-reauthorizing-namespacescope-operator-minimum-rbac)
- `cpd_vars.sh` has been prepared per the IBM documentation and is sourced in every new shell session.

## Prerequisites

### 1. Prepare cpd_vars.sh

Follow the IBM guidance at [Setting up installation environment variables](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=information-setting-up-installation-environment-variables) to prepare `cpd_vars.sh`. The variables most relevant to this patch workflow are shown below. Adjust `CPD_CLI_MANAGE_WORKSPACE` to match the existing `cpd-cli-workspace` directory in the customer environment.

```bash
# Client workstation
export CPD_CLI_MANAGE_WORKSPACE=/root/cpd-cli-workspace

# IBM Software Hub version
export VERSION=5.3.1
export OLM_UTILS_IMAGE=icr.io/cpopen/cpd/olm-utils-v4:${VERSION}
```

Source the script in every new shell session:

```bash
source ./cpd_vars.sh
```

### 2. Identify the correct patch YAML URL

Before running any commands, determine the raw GitHub URL for `patch-5.3.1.yaml` at the desired patch tag.

**Step 1.** Open <https://github.com/IBM/software-hub/tree/main/patch> in a browser. You should see the `patch-5.3.1.yaml` file listed. Note that `main` always reflects the latest patch, which is why we need to switch to a tag.

![Browse to the patch directory on main](images/01-github-patch-directory.png)
*Figure 1. Browse to the patch directory on main*

**Step 2.** Click the branch/tag selector (labeled `main`), then switch to the **Tags** tab. Select the tag that corresponds to the patch you want (for example, `swhub-5.3.1-p1` for Patch 1, `swhub-5.3.1-p2` for Patch 2).

![Switch from the main branch to the Tags tab](images/02-tags-selector.png)
*Figure 2. Switch from the main branch to the Tags tab*

**Step 3.** Click on `patch-5.3.1.yaml` to open the file at that tag. Then click the **Raw** button in the upper right of the file view.

![Open the YAML file at the tag and click Raw](images/03-raw-button.png)
*Figure 3. Open the YAML file at the tag and click Raw*

**Step 4.** Copy the raw URL from the browser address bar. It will follow this pattern:

```
https://raw.githubusercontent.com/IBM/software-hub/refs/tags/<tag>/patch/patch-5.3.1.yaml
```

![Copy the raw URL from the address bar](images/04-raw-url.png)
*Figure 4. Copy the raw URL from the address bar*

For Patch 1, the URL is:

```
https://raw.githubusercontent.com/IBM/software-hub/refs/tags/swhub-5.3.1-p1/patch/patch-5.3.1.yaml
```

## Procedure

### Step 1: Gather information about installed components

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

### Step 2: Stage the pinned patch manifest

**Reference:** [Checking for new patches](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=patches-checking-new)

Ensure no stale `patch-5.3.1.yaml` exists. Delete any found:

```bash
find ${CPD_CLI_MANAGE_WORKSPACE} -name "patch-5.3.1.yaml"
```

Download the patch manifest for the specific tag identified in the Prerequisites section. Run from the directory that contains `cpd-cli-workspace`:

```bash
wget -O ${CPD_CLI_MANAGE_WORKSPACE}/work/offline/patch/patch-5.3.1.yaml \
  https://raw.githubusercontent.com/IBM/software-hub/refs/tags/swhub-5.3.1-p1/patch/patch-5.3.1.yaml
```

> **Note:** For other patch levels, replace `swhub-5.3.1-p1` with the appropriate tag (for example, `swhub-5.3.1-p2`). See the Prerequisites section for how to identify the tag.

### Step 3: Refresh the olm-utils container

```bash
cpd-cli manage restart-container
```

### Step 4: List available patches (pinned)

```bash
cpd-cli manage list-patch --patch_download=false
```

Review the output. It will print the `case-download` command that would be used. Use the component list from that output, but always append `--patch_download=false` to the actual command.

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
