### WKC 4.8.5 Hotfix April 2024 for Verizon Installation Instructions

Follow **Step 1** and the sub-steps to download and copy the images to a
local private registry for an air-gapped environment.

Follow **Step 2** and the sub-steps to go through applying the patch
using the online IBM entitled registry, or to apply the hotfix using the
images downloaded to the local private registry from Step 1.

**Procedure**

1)  To apply the patch in an air-gapped environment, proceed with the
    following steps.

    a.  Log in to the OpenShift console as the cluster admin.

    b.  Prepare the authentication credentials to access the IBM
        production repository. Use the same auth.json file used for CASE
        download and image mirroring. An example directory path:

> \${HOME}/.airgap/auth.json
>
> Or create an auth.json file that contains credentials to access icr.io
> and your local private registry. For example:
>
> {
>
> \"auths\": {
>
> \"cp.icr.io\":{\"email\":\"unused\",\"auth\":\"\<base64 encoded
> id:apikey\>\"},
>
> \"\<private registry
> hostname\>\":{\"email\":\"unused\",\"auth\":\"\<base64 encoded
> id:password\>\"}
>
> }
>
> }
>
> For more information about the auth.json file, see
> [[containers-auth.json - syntax for the registry authentication
> file]{.underline}](https://www.ibm.com/links?url=https%3A%2F%2Fman.archlinux.org%2Fman%2Fcontainers-common%2Fcontainers-auth.json.5.en).

c.  Install skopeo by running:

> yum install skopeo

d.  To confirm the path for the local private registry to copy the
    hotfix images to, run the following command:

> oc describe pod \<hotfix image pod\> \| grep -i \"image:\"
>
> *\<hotfix image pod\>* can be the pod name for any of the images which
> will be patched with this hotfix.
>
> For example:
>
> oc describe pod wdp-profiling-7855f7fd8f-lsvsj \| grep Image:
>
> ...
>
> Image:
> cp.icr.io/cp/cpd/wdp-profiling@sha256:03c88c69b986f24d39e4556731c0d171169d2bd91b0fb22f6367fd51c9020e64

e.  To get the local private registry source details, run the following
    commands:

> oc get imageContentSourcePolicy
>
> oc describe imageContentSourcePolicy \[cloud-pak-for-data-mirror\]
>
> The local private registry mirror repository and path details should
> be in the output of the describe command:
>
> \- mirrors:
>
> \- \${PRIVATE_REGISTRY_LOCATION}/cp/
>
> source: cp.icr.io/cp/cpd
>
> For more information about mirroring of images, see [[Configuring your
> cluster to pull Cloud Pak for ­Data
> images]{.underline}](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-configuring-your-cluster-pull-images&_ga=2.155880511.526964407.1664897771-1656206445.1664215600).

f.  Use the *skopeo* command to copy the patch images from the IBM
    production registry to the local private registry. Using the
    appropriate auth.json file, copy the patch images from the IBM
    production registry to the Openshift cluster registry:

> **NOTE: When copy/pasting each "skopeo" command below, it is
> recommended to copy the command into a text editor to ensure there are
> no additional newline characters after the '\\'. Remove any additional
> newline characters. Then copy/paste the command from the text editor
> to the command line. If these steps are not done, the command may
> fail.**
>
> **CCS**
>
> [skopeo copy \--all \--authfile \"\<folder path\>/auth.json\"
> \\]{.mark}
>
> [\--dest-tls-verify=false \--src-tls-verify=false \\]{.mark}
>
> [docker://cp.icr.io/cp/cpd/asset-files-api@sha256:a1525c29bebed6e9a982f3a06b3190654df7cf6028438f58c96d0c8f69e674c1
> \\]{.mark}
>
> [\<local private
> registry\>/cp/cpd/asset-files-api@sha256:a1525c29bebed6e9a982f3a06b3190654df7cf6028438f58c96d0c8f69e674c1 ]{.mark}
>
> [skopeo copy \--all \--authfile \"\<folder path\>/auth.json\"
> \\]{.mark}
>
> [\--dest-tls-verify=false \--src-tls-verify=false \\]{.mark}
>
> [docker://cp.icr.io/cp/cpd/portal-projects@sha256:d3722fb9a7e4a97f6f6de7d2b92837475e62cd064aa6d7590342e05620b16a6a
> \\]{.mark}
>
> [\<local private
> registry\>/cp/cpd/portal-projects@sha256:d3722fb9a7e4a97f6f6de7d2b92837475e62cd064aa6d7590342e05620b16a6a]{.mark}
>
> **WKC**
>
> [skopeo copy \--all \--authfile \"\<folder path\>/auth.json\"
> \\]{.mark}
>
> [\--dest-tls-verify=false \--src-tls-verify=false \\]{.mark}
>
> [docker://cp.icr.io/cp/cpd/wdp-profiling@sha256:1cedac257eb092c1a906046bd4ff8939856c1ef5fd624d1772469f795ee7236f
> \\]{.mark}
>
> [\<local private
> registry\>/cp/cpd/wdp-profiling@sha256:1cedac257eb092c1a906046bd4ff8939856c1ef5fd624d1772469f795ee7236f]{.mark}

2)  To install the patch using the online IBM entitled registry, or to
    apply the hotfix using the images downloaded to the local private
    registry from Step 1, proceed with the following commands. Note that
    \${PROJECT_CPD_INSTANCE} refers to the project name where WKC is
    installed. 

> **NOTE: When copy/pasting each "oc patch" command below, please ensure
> that it is contained on a single line without line breaks. Also, if
> any spaces are introduced by the copy/paste, they must be removed. If
> these steps are not done, the command may fail.**

a.  Save a copy of the CRs currently deployed on the cluster

oc get wkc wkc-cr -o yaml -n \${PROJECT_CPD_INSTANCE} \>
wkc_cr_backup\_\<date\>.yaml

oc get ccs ccs-cr -o yaml -n \${PROJECT_CPD_INSTANCE} \>
ccs_cr_backup\_\<date\>.yaml

where \<date\> is the date you are applying the patch.

b.  Run the following command to apply the patch to the Common Core
    Services custom resource (ccs-cr):

> [oc patch ccs ccs-cr -n \${PROJECT_CPD_INSTANCE} \--type=merge -p
> \'{\"spec\":{\"asset_files_api_image\":{\"name\":\"asset-files-api@sha256\",\"tag\":\"a1525c29bebed6e9a982f3a06b3190654df7cf6028438f58c96d0c8f69e674c1\",\"tag_metadata\":\"4.6.5.4.155-amd64\"},\"portal_projects_image\":{\"name\":\"portal-projects@sha256\",\"tag\":\"d3722fb9a7e4a97f6f6de7d2b92837475e62cd064aa6d7590342e05620b16a6a\",\"tag_metadata\":\"4.6.5.4.2504-amd64\"}}}\']{.mark}

c.  Run the following command to apply the patch to the WKC custom
    resource (wkc-cr):

> [oc patch wkc wkc-cr -n \${PROJECT_CPD_INSTANCE} \--type=merge -p
> \'[{\"spec\":{\"wdp_profiling_image\":{\"name\":\"wdp-profiling@sha256\",\"tag\":\"ecc845503e45b4f8a0c83dce077d41c9a816cb9116d3aa411b000ec0eb916620\",\"tag_metadata\":\"4.6.5031-amd64](mailto:{"spec":{"wdp_profiling_image":{"name":"wdp-profiling@sha256","tag":"ecc845503e45b4f8a0c83dce077d41c9a816cb9116d3aa411b000ec0eb916620","tag_metadata":"4.6.5031-amd64)\"}}}\']{.mark}

d.  Run the following command to apply recommended retry configuration
    and divisor settings to the WKC custom resource (wkc-cr):

> oc patch wkc wkc-cr -n \${PROJECT_CPD_INSTANCE} \--type=merge -p
> \'{\"spec\":{\"wkc_term_assignment_rest_retry_config\":\"cams\\\\\\\\.write:400\|3\|2;cams\\\\\\\\.attachment\\\\\\\\.markcomplete:400\|3\|2,404\|3\|2;finley\\\\\\\\.predict:500\|18\|10,502\|18\|10,503\|18\|10,504\|1\|10,507\|18\|10;.\*:408\|3\|2,409\|3\|2,425\|3\|2,429\|3\|2,500\|6\|10,502\|6\|10,503\|6\|10,504\|6\|10,507\|6\|10,-1\|3\|2\"}}\'
>
> oc patch wkc wkc-cr -n \${PROJECT_CPD_INSTANCE} \--type=merge -p
> \'{\"spec\":{\"wkc_term_assignment_finley_page_size_reduction_divisor\":\"5\"}}\'

e.  Wait for the Common Core Services operator reconciliation to
    complete. Run the following command to\
    monitor the reconciliation status:\
    \
    oc get ccs ccs-cr -n \${PROJECT_CPD_INSTANCE}

> After a period of time, the asset-files-api and portal-projects pods
> in \${PROJECT_CPD_INSTANCE} should be up and running with the updated
> images, and the CCS custom resource should show a status of
> "Completed".

f.  Wait for the WKC operator reconciliation to complete. Run the
    following command to monitor the reconciliation status:

> oc get wkc wkc-cr -n \${PROJECT_CPD_INSTANCE}
>
> After a period of time, the wdp-profiling pod should be up and running
> with the updated image, and the WKC custom resource should show a
> status of "Completed".

3)  Unzip the contents of the MDI_Job_Reconciliation_Script.zip file
    into a folder. This may be needed in the event that the state of an
    MDI job does not match the state in the MDI job log. See the
    included readme.txt file for details on how to run the script.
    [Check with Sathis if still needed.]{.mark}

**To revert the hotfix changes**

Make sure to revert the image overrides before you install or upgrade to
a newer refresh or a major release of IBM® Cloud Pak for Data.

To revert the image overrides, proceed with the following steps. Note
that \${PROJECT_CPD_INSTANCE} refers to the project name where WKC is
installed. 

**NOTE: When copy/pasting each "oc patch" command below, please ensure
that it is contained on a single line without line breaks. Also, if any
spaces are introduced by the copy/paste, they must be removed. If these
steps are not done, the command may fail.**

a)  Run the following command to edit the Common Core Services custom
    resource:

> oc edit ccs ccs-cr -n \${PROJECT_CPD_INSTANCE}

b)  Remove the following lines within the CCS custom resource and
    replace with the copy that you saved during the installation. Save
    the change.

> [asset_files_api_image:]{.mark}
>
> [name: asset-files-api@sha256]{.mark}
>
> [tag:
> a1525c29bebed6e9a982f3a06b3190654df7cf6028438f58c96d0c8f69e674c1]{.mark}
>
> [tag_metadata: 4.6.5.4.155-amd64]{.mark}
>
> [portal_projects_image:]{.mark}
>
> [name: portal-projects@sha256]{.mark}
>
> [tag:
> d3722fb9a7e4a97f6f6de7d2b92837475e62cd064aa6d7590342e05620b16a6a]{.mark}
>
> [tag_metadata: 4.6.5.4.2504-amd64]{.mark}

c)  Run the following command to edit the WKC custom resource:

oc edit wkc wkc-cr -n \${PROJECT_CPD_INSTANCE}

d)  Remove the following lines for the patched images within the WKC
    custom resource and replace with the copy that you saved during the
    installation. Save the change.

> [wdp_profiling_image:]{.mark}
>
> [name: wdp-profiling@sha256]{.mark}
>
> [tag:
> ecc845503e45b4f8a0c83dce077d41c9a816cb9116d3aa411b000ec0eb916620]{.mark}
>
> [tag_metadata: 4.6.5031-amd64]{.mark}

e)  Wait for the Common Core Services operator reconciliation to
    complete. Run the following command to monitor the reconciliation
    status:\
    \
    oc get ccs ccs-cr -n \${PROJECT_CPD_INSTANCE}

> After a period of time, the asset-files-api and portal-projects pods
> in \${PROJECT_CPD_INSTANCE} should be up and running with the original
> images, and the CCS custom resource should show a status of
> "Completed".

f)  Wait for the WKC operator reconciliation to complete. Run the
    following command to monitor the reconciliation status:\
    \
    oc get wkc wkc-cr -n \${PROJECT_CPD_INSTANCE}

> After a period of time, wdp-profiling pod in \${PROJECT_CPD_INSTANCE}
> should be up and running with the original image, and the WKC custom
> resource should show a status of "Completed".
