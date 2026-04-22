# Offline Restore Runbook - 5.2.2
**Note:**
This runbook is not for a general offline restore.

## 1. Preparing the target cluster
Make sure that the target cluster meets the following requirements:
<br>
- The target cluster has the same storage classes as the source cluster.
- For environments that use a private container registry, such as air-gapped environmenicrts, the target cluster has the same image content source policy as the source cluster.
- The target cluster must be able to pull the 5.2.2 software images required by the restore.
- The target cluster is on the same OpenShift version as the source cluster.
- The target cluster allows for the same node configuration as the source cluster. For example, if the source cluster uses a custom KubeletConfig, the target cluster must allow the same custom KubeletConfig.
- IBM Software Hub & Services have been uninstalled including the clean up of CRs, operators and namespaces.
- IBM Scheduler has been uninstalled including the clean up of namespaces.
- IBM Licensing service and IBM Cert manager have been reinstalled including the clean up of namespaces.

### 1.1 Set up work statiion

copy the `cpd_vars.sh` `oadp_vars.sh` from the PROD environment
## Sourcing the environment variables

```bash
source ./cpd_vars.sh
source ./oadp_vars.sh
```

Download Version 14.2.2 of cpd-cli (cpd 5..2.2):
```
wget https://github.com/IBM/cpd-cli/releases/download/v14.2.2/cpd-cli-linux-EE-14.2.2.tgz

tar zxf cpd-cli-linux-EE-14.2.2.tgz
```
```
podman pull icr.io/cpopen/cpd/olm-utils-v3:5.2.2
podman tag icr.io/cpopen/cpd/olm-utils-v3:5.2.2 ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v3:5.2.2

podman login hub.fbond:5000 -u ocadmin -p ocadmin

podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v3:5.2.2 --tls-verify=false --remove-signatures 

export OLM_UTILS_IMAGE=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/olm-utils-v3:5.2.2
export OLM_UTILS_LAUNCH_ARGS=" --network=host"
./cpd-cli manage restart-container

./cpd-cli manage login-to-ocp --server=https://api.localcluster.fbond:6443 --token=$(cat /root/.sa/token)
```

### 1.2 Mirroring CPD 5.2.2 images (ETC 8 hours)

- Log in to the IBM Entitled Registry registry
```
./cpd-cli manage login-entitled-registry \
${IBM_ENTITLEMENT_KEY}
```
- Log in to the private container registry
```
./cpd-cli manage login-private-registry \
${PRIVATE_REGISTRY_LOCATION} \
${PRIVATE_REGISTRY_PUSH_USER} \
${PRIVATE_REGISTRY_PUSH_PASSWORD}
```
- Case download
```
./cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION}
```
- List images
```
./cpd-cli manage list-images \ 
--components=${COMPONENTS} \ 
--release=${VERSION} \ 
--inspect_source_registry=true
```

- run the following script

```
export LOCAL_SECRET_JSON=/opt/ibm/appliance/storage/ips/cpd483/ocp/pull-secret.json

skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-acmesolver@sha256:2472053d912f693a5fcdd00b7d9f0a7ce98697c47eae955e31c0b602fd8a4a37 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-acmesolver@sha256:2472053d912f693a5fcdd00b7d9f0a7ce98697c47eae955e31c0b602fd8a4a37
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-acmesolver@sha256:24fca2376d04419c35a10aec50b085bbdcdbb10ea1bff2e1b1c573202d406129 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-acmesolver@sha256:24fca2376d04419c35a10aec50b085bbdcdbb10ea1bff2e1b1c573202d406129
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-acmesolver@sha256:807f2cd6fd3dc1d93d0973650fcbf9732ca91522ec375b64cd53edb6616ac74d docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-acmesolver@sha256:807f2cd6fd3dc1d93d0973650fcbf9732ca91522ec375b64cd53edb6616ac74d
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-acmesolver@sha256:9ced60f18740063ab281b836d4caf21035242ac3f659c6bb022ddb546770bff9 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-acmesolver@sha256:9ced60f18740063ab281b836d4caf21035242ac3f659c6bb022ddb546770bff9
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-acmesolver@sha256:c742e7fc33390135c3c63b61f8152f2c63594789567fd905bf9d2510ad7e12a5 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-acmesolver@sha256:c742e7fc33390135c3c63b61f8152f2c63594789567fd905bf9d2510ad7e12a5
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-acmesolver@sha256:e6c16348e532b1e716428ef46856ecb89a9c17ab380879da5ae05021238d3690 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-acmesolver@sha256:e6c16348e532b1e716428ef46856ecb89a9c17ab380879da5ae05021238d3690

skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-cainjector@sha256:2e70036d7299168331c94af70a22e3b2063c56160f616362b1a2187c12d003a4 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-cainjector@sha256:2e70036d7299168331c94af70a22e3b2063c56160f616362b1a2187c12d003a4
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-cainjector@sha256:9e80454a08e87d35eb5a16c74c4d170a8c13280adb0cdbbd85e951cd2a880a08 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-cainjector@sha256:9e80454a08e87d35eb5a16c74c4d170a8c13280adb0cdbbd85e951cd2a880a08
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-cainjector@sha256:b14602af498bc1877977b219217754ac1a497a669e27d69d7e218f12db649ba3 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-cainjector@sha256:b14602af498bc1877977b219217754ac1a497a669e27d69d7e218f12db649ba3
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-cainjector@sha256:ceee171c5ce0a8e580dfeccfe0dbfb73474c109becd45a562411678f3ab08595 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-cainjector@sha256:ceee171c5ce0a8e580dfeccfe0dbfb73474c109becd45a562411678f3ab08595
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-cainjector@sha256:fddadff3092b022630610f262b6c3389d30a97a2e95ee8e31d1ddfbdf3999c52 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-cainjector@sha256:fddadff3092b022630610f262b6c3389d30a97a2e95ee8e31d1ddfbdf3999c52
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-cainjector@sha256:ff3a99878fe11a889df5241b07ea07a84958fb50c25423bb6acb08e9f044c46d docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-cainjector@sha256:ff3a99878fe11a889df5241b07ea07a84958fb50c25423bb6acb08e9f044c46d

skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-controller@sha256:0dc719e58694fb501b8b36f3a375b3c0462624d27fe49afe6b835821e31393f3 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-controller@sha256:0dc719e58694fb501b8b36f3a375b3c0462624d27fe49afe6b835821e31393f3
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-controller@sha256:0f01a67524fa46829e455f654d3b9b00ceac44f3d870b21c022051dfcc8a00eb docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-controller@sha256:0f01a67524fa46829e455f654d3b9b00ceac44f3d870b21c022051dfcc8a00eb
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-controller@sha256:4ba109b496d0f1844605d775c0280610148a8f616c171d7e8110eed7e872d23e docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-controller@sha256:4ba109b496d0f1844605d775c0280610148a8f616c171d7e8110eed7e872d23e
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-controller@sha256:62319a856b5988284b34de1ee4044f62699c838811930d97fe58a24ffc054909 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-controller@sha256:62319a856b5988284b34de1ee4044f62699c838811930d97fe58a24ffc054909
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-controller@sha256:b041b9b033298042d168a98815e1a6c25337252e9dac6fcd7f00bfe0831a9073 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-controller@sha256:b041b9b033298042d168a98815e1a6c25337252e9dac6fcd7f00bfe0831a9073
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-controller@sha256:c6309ae6ae18916b8e7327c782db55f5c02f94079977d3895e7223e5f05fd6c4 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-controller@sha256:c6309ae6ae18916b8e7327c782db55f5c02f94079977d3895e7223e5f05fd6c4

skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-webhook@sha256:24f0169191cb707f27419226077180e35879503d72bd8c75342cc14e11db65ae docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-webhook@sha256:24f0169191cb707f27419226077180e35879503d72bd8c75342cc14e11db65ae
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-webhook@sha256:67a02a9264d9c6e82edc15ae36d4f99b3fa9099697c550088d27c76511189f37 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-webhook@sha256:67a02a9264d9c6e82edc15ae36d4f99b3fa9099697c550088d27c76511189f37
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-webhook@sha256:6cd127d3b327de9efdc0761bb3c769fe0f41e6aec58ca4ad0ea9940bdc903e16 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-webhook@sha256:6cd127d3b327de9efdc0761bb3c769fe0f41e6aec58ca4ad0ea9940bdc903e16
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-webhook@sha256:b646ea3330f3fd9bb3174224de69f20ef58589a464d50d1c9399211141f67bd5 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-webhook@sha256:b646ea3330f3fd9bb3174224de69f20ef58589a464d50d1c9399211141f67bd5
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-webhook@sha256:bced4ebd4a7ac776d6acec6785521fba51ecf21e026c87e8689c12d670a39b49 docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-webhook@sha256:bced4ebd4a7ac776d6acec6785521fba51ecf21e026c87e8689c12d670a39b49
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/cpfs/icp-cert-manager-webhook@sha256:e5e0fe6f3582ad85c4b1f2ad3a833f0c99d65b8aff674af2d41554f410e3482e docker://hub.fbond:5000/cpopen/cpfs/icp-cert-manager-webhook@sha256:e5e0fe6f3582ad85c4b1f2ad3a833f0c99d65b8aff674af2d41554f410e3482e

skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/ibm-cert-manager-operator-bundle@sha256:86f7c3d5deae86be4c0d33d76850623c46ba171bef5382c54a91e4bd3361ada2 docker://hub.fbond:5000/cpopen/ibm-cert-manager-operator-bundle@sha256:86f7c3d5deae86be4c0d33d76850623c46ba171bef5382c54a91e4bd3361ada2
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/ibm-cert-manager-operator-bundle@sha256:b9c8a65c1742c72c6351b1c8f4a858940c2bfa00ac5ec271c57b2ee7e5e7a66b docker://hub.fbond:5000/cpopen/ibm-cert-manager-operator-bundle@sha256:b9c8a65c1742c72c6351b1c8f4a858940c2bfa00ac5ec271c57b2ee7e5e7a66b
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/ibm-cert-manager-operator-bundle@sha256:d61b7b8e6a9785b59630d9c5776f72a29ffbf2e81eb753d7b11dcf2d9d77f1f0 docker://hub.fbond:5000/cpopen/ibm-cert-manager-operator-bundle@sha256:d61b7b8e6a9785b59630d9c5776f72a29ffbf2e81eb753d7b11dcf2d9d77f1f0

skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/ibm-cert-manager-operator-catalog@sha256:4bd876de3f52e0f88a43f8546b281672de9a8d7bc7c5ff5524d700a979b26344 docker://hub.fbond:5000/cpopen/ibm-cert-manager-operator-catalog@sha256:4bd876de3f52e0f88a43f8546b281672de9a8d7bc7c5ff5524d700a979b26344
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/ibm-cert-manager-operator-catalog@sha256:78e1242e197ed1a2db82ea29887ff85b77ce39d15868fd26505fcfe7db0c3bba docker://hub.fbond:5000/cpopen/ibm-cert-manager-operator-catalog@sha256:78e1242e197ed1a2db82ea29887ff85b77ce39d15868fd26505fcfe7db0c3bba
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/ibm-cert-manager-operator-catalog@sha256:c95a0e9261ea0afbfe03fff8fe86e2f9015aad50d125dd063de0e59c9815cf38 docker://hub.fbond:5000/cpopen/ibm-cert-manager-operator-catalog@sha256:c95a0e9261ea0afbfe03fff8fe86e2f9015aad50d125dd063de0e59c9815cf38
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/ibm-cert-manager-operator-catalog@sha256:de76aa967f3ea1c2bb7d56622111f8cb82f12ed99f5a627d8050149e770fd62d docker://hub.fbond:5000/cpopen/ibm-cert-manager-operator-catalog@sha256:de76aa967f3ea1c2bb7d56622111f8cb82f12ed99f5a627d8050149e770fd62d
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/ibm-cert-manager-operator-catalog@sha256:fc11d43b2597bcf1fac3c554a7841416caf3e1d14492b71777655b3aaff4a0c9 docker://hub.fbond:5000/cpopen/ibm-cert-manager-operator-catalog@sha256:fc11d43b2597bcf1fac3c554a7841416caf3e1d14492b71777655b3aaff4a0c9

skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/ibm-cert-manager-operator@sha256:107d784a51090b136fddfb6329e45e87d12bd4a1efeac258a2b817afacea4999 docker://hub.fbond:5000/cpopen/ibm-cert-manager-operator@sha256:107d784a51090b136fddfb6329e45e87d12bd4a1efeac258a2b817afacea4999
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/ibm-cert-manager-operator@sha256:4d8e20797eaf15fbe779f33d4cb9495ddc721dce2caf3898930066bd59ea384e docker://hub.fbond:5000/cpopen/ibm-cert-manager-operator@sha256:4d8e20797eaf15fbe779f33d4cb9495ddc721dce2caf3898930066bd59ea384e
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/ibm-cert-manager-operator@sha256:4e3ec61a34fc7c71d213999b68b1f5c3f866f97e19c12fb001d506d4166a50f1 docker://hub.fbond:5000/cpopen/ibm-cert-manager-operator@sha256:4e3ec61a34fc7c71d213999b68b1f5c3f866f97e19c12fb001d506d4166a50f1
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/ibm-cert-manager-operator@sha256:9af5ad5a81f23e9655235a42fa4ffef416dbbad7bff8911579d71cf030aaa71b docker://hub.fbond:5000/cpopen/ibm-cert-manager-operator@sha256:9af5ad5a81f23e9655235a42fa4ffef416dbbad7bff8911579d71cf030aaa71b
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/ibm-cert-manager-operator@sha256:b8c83fe5215b7412765181aaba84162b6cb188b5765e45972a6f29b2a9f19826 docker://hub.fbond:5000/cpopen/ibm-cert-manager-operator@sha256:b8c83fe5215b7412765181aaba84162b6cb188b5765e45972a6f29b2a9f19826
skopeo copy --all --authfile ${LOCAL_SECRET_JSON} --dest-tls-verify=false --src-tls-verify=false docker://icr.io/cpopen/ibm-cert-manager-operator@sha256:e3debea7176a592e51a0f6d8dd6b73530304476774224d206c15a296ade10e96 docker://hub.fbond:5000/cpopen/ibm-cert-manager-operator@sha256:e3debea7176a592e51a0f6d8dd6b73530304476774224d206c15a296ade10e96

```

- run the following script

```
export LOCAL_SECRET_JSON=/opt/ibm/appliance/storage/ips/cpd483/ocp/pull-secret.json
DEST_REG=docker://hub.fbond:5000

for img in \
icr.io/cpopen/cpfs/ibm-licensing-usage@sha256:8438c2229b500fc9c40629280701856345e228fc645a5b468bee2135c61b2387 \
icr.io/cpopen/cpfs/ibm-licensing-usage@sha256:fe5bc4a40472926ec0df3023777b2aa8105507128bd9ac02eac2c68ad08723c1 \
icr.io/cpopen/cpfs/ibm-licensing@sha256:14559058d4152de9513db49a22923e34a5bbb8870f1cc8d57c533c88bfb283d8 \
icr.io/cpopen/cpfs/ibm-licensing@sha256:2c172397873edbaa2d98108115f73ca4c05d7b742745d35de0b301804a24765a \
icr.io/cpopen/cpfs/ibm-licensing@sha256:af9901b0e96306708d5e741e5536450e9a7e138aae00385e49c53878a83dc5c2 \
icr.io/cpopen/cpfs/ibm-licensing@sha256:b87b6c32a2050f1d5ff311d163a7b45efa7d5de1d352984b2f70fa578301a952 \
icr.io/cpopen/cpfs/ibm-licensing@sha256:bcfb9b7d3b379ee2bb0b449cdf5af3fbc88f99c77641441ca392ebed04403e00 \
icr.io/cpopen/cpfs/ibm-licensing@sha256:dd76a8484628970b62238334e122768aeea820ab18e24d62d23a6328c83b868c \
icr.io/cpopen/ibm-licensing-catalog@sha256:42e8f5a12742742c1967568d497ff1c93aa7a78a3a7ad414da011d0c1eedb19a \
icr.io/cpopen/ibm-licensing-catalog@sha256:52671613bc09e45287f1585d3a5ce5f8f5072375646c32256bd06fb6317d6202 \
icr.io/cpopen/ibm-licensing-catalog@sha256:6519a8a3c26ebf4fe5d74eff2093004ac5e8de23079cc069c2500e98a559db1c \
icr.io/cpopen/ibm-licensing-catalog@sha256:8af0feccfb1cedf035b9dd509d5b484bd17cf93757c19f995e56a26e1faa59b0 \
icr.io/cpopen/ibm-licensing-catalog@sha256:e0e5c9098b477d1ffb241c9eb328d1df9013a22e07769585ecc87d47def66283 \
icr.io/cpopen/ibm-licensing-operator-bundle@sha256:cab91a231068a6bbef03ee8db32fadf11654317c4573c6714721a15d0e2a98dd \
icr.io/cpopen/ibm-licensing-operator-bundle@sha256:d2680c5f05c886b44df644aef6cb2fdc9534bb6a238066f6ebbb97bbeaac431e \
icr.io/cpopen/ibm-licensing-operator-bundle@sha256:f0c49fdefce70b5125627618a948edb17ac57ccc1fb542c51e0f02dc111233b6 \
icr.io/cpopen/ibm-licensing-operator@sha256:03a9b9a9fc0672a8a33d5095233e026e1f8a41562e0890624a44283c1eb2a266 \
icr.io/cpopen/ibm-licensing-operator@sha256:24ca69c2f80a4369e29aa382a94d64fd3546b110787df1142c45a20b8246e844 \
icr.io/cpopen/ibm-licensing-operator@sha256:6a107eef68f984d0cc0a79478b11bab9402ba4b0366ae3156e5263c5d42ca58d \
icr.io/cpopen/ibm-licensing-operator@sha256:7c84f39b9ec603f5943c3c8efe1355c15140d0b7cb1842cf6c80024661ebb929 \
icr.io/cpopen/ibm-licensing-operator@sha256:932825fc7af8b96c381d504cd0976c751661ce078257430e5c79d565f416bacb \
icr.io/cpopen/ibm-licensing-operator@sha256:da38999d66199df0eeabd7c9e934205fb858cf2c52be8bbaf6b34adb894dd38e

do
  skopeo copy --all --authfile ${LOCAL_SECRET_JSON} \
    --dest-tls-verify=false --src-tls-verify=false \
    docker://${img} ${DEST_REG}/${img#icr.io/}
done
 
```

- Mirror images
```
./cpd-cli manage mirror-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--arch=${IMAGE_ARCH} \
--case_download=false
```

## 1.3 Configuring backup and restore components
- Create the `${OADP_PROJECT}` project where you want to install the OADP operator.
```
oc new-project ${OADP_PROJECT}
```

- Annotate the ${OADP_PROJECT} project so that Restic pods can be scheduled on all nodes.
```
oc annotate namespace ${OADP_PROJECT} openshift.io/node-selector=""
```

- Install the Red Hat OADP operator if it's not installed yet.

Check whether the OADP operator already installed or not.

```
oc get csv -A | grep -i oadp
```

If the OADP operator Not installed, then follow below link for the OADP installation.
<br>
[Install the OADP operator and configure it to work with your object store](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/backup_and_restore/oadp-application-backup-and-restore#installing-oadp)

If the OADP operator already installed, check that the OADP operator version is 1.4.x

```
oc get csv -A | grep -i oadp
```

Upgrade the the OADP operator if the version is lower than 1.4.

- Create a secret in the ${OADP_PROJECT} project with the credentials of the S3-compatible object store that you are using to store the backups.

<br>
Create a file named `credentials-velero` that contains the credentials for the object store:

```
cat << EOF > credentials-velero
[default]
aws_access_key_id=${ACCESS_KEY_ID}
aws_secret_access_key=${SECRET_ACCESS_KEY}
EOF
```

Create the secret.The name of the secret must be `cloud-credentials`.
```
oc create secret generic cloud-credentials \
--namespace ${OADP_PROJECT} \
--from-file cloud=./credentials-velero
```

**Moving images for backup and restore to a private container registry**

- Log in to the private container registry
```
podman login ${PRIVATE_REGISTRY_LOCATION} \
-u ${PRIVATE_REGISTRY_PUSH_USER} \
-p ${PRIVATE_REGISTRY_PUSH_PASSWORD}
```
- Log in to the Red Hat entitled registry.

Set the `REDHAT_USER` environment variable to the username of a user who can pull images from `registry.redhat.io`:

```
export REDHAT_USER=<enter-your-username>
```

Set the `REDHAT_PASSWORD` environment variable to the password for the specified user:
```
export REDHAT_PASSWORD=<enter-your-password>
```

Log in to `registry.redhat.io`:

```
podman login registry.redhat.io -u ${REDHAT_USER} -p ${REDHAT_PASSWORD}
```

- Run the following commands to mirror the images the private container registry.

```
podman login ${PRIVATE_REGISTRY_LOCATION} \
-u ${PRIVATE_REGISTRY_PUSH_USER} \
-p ${PRIVATE_REGISTRY_PUSH_PASSWORD}

podman pull icr.io/cpopen/cpd/swhub-velero-plugin:${VERSION}
podman tag icr.io/cpopen/cpd/swhub-velero-plugin:${VERSION} ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/swhub-velero-plugin:${VERSION}
podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/swhub-velero-plugin:${VERSION} --tls-verify=false --remove-signatures

podman pull icr.io/cpopen/cpfs/cpfs-oadp-plugins:${CPFS_OADP_PLUGIN_VERSION}
podman tag icr.io/cpopen/cpfs/cpfs-oadp-plugins:${CPFS_OADP_PLUGIN_VERSION} ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs/cpfs-oadp-plugins:${CPFS_OADP_PLUGIN_VERSION}
podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs/cpfs-oadp-plugins:${CPFS_OADP_PLUGIN_VERSION} --tls-verify=false --remove-signatures

podman pull registry.redhat.io/ubi9/ubi-minimal:latest
podman tag registry.redhat.io/ubi9/ubi-minimal:latest ${PRIVATE_REGISTRY_LOCATION}/ubi9/ubi-minimal:latest 
podman push ${PRIVATE_REGISTRY_LOCATION}/ubi9/ubi-minimal:latest --tls-verify=false --remove-signatures

podman pull registry.redhat.io/openshift4/ose-cli:latest
podman tag registry.redhat.io/openshift4/ose-cli:latest ${PRIVATE_REGISTRY_LOCATION}/openshift4/ose-cli:latest 
podman push ${PRIVATE_REGISTRY_LOCATION}/openshift4/ose-cli:latest --tls-verify=false --remove-signatures

podman pull icr.io/db2u/db2u-velero-plugin:${VERSION}
podman tag icr.io/db2u/db2u-velero-plugin:${VERSION} ${PRIVATE_REGISTRY_LOCATION}/db2u/db2u-velero-plugin:${VERSION}
podman push ${PRIVATE_REGISTRY_LOCATION}/db2u/db2u-velero-plugin:${VERSION} --tls-verify=false --remove-signatures

podman pull icr.io/cpopen/cpd/cpdbr-velero-plugin:${VERSION}
podman tag icr.io/cpopen/cpd/cpdbr-velero-plugin:${VERSION} ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/cpdbr-velero-plugin:${VERSION} 
podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/cpdbr-velero-plugin:${VERSION} --tls-verify=false --remove-signatures

podman pull icr.io/cpopen/cpd/cpdbr-oadp:${VERSION}
podman tag icr.io/cpopen/cpd/cpdbr-oadp:${VERSION} ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/cpdbr-oadp:${VERSION} 
podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/cpdbr-oadp:${VERSION} --tls-verify=false --remove-signatures

podman pull icr.io/cpopen/cpfs/cpfs-utils:4.6.5 
podman tag icr.io/cpopen/cpfs/cpfs-utils:4.6.5 ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs/cpfs-utils:4.6.5
podman push ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs/cpfs-utils:4.6.5 --tls-verify=false --remove-signatures
```

- Create or update the DataProtectionApplication (DPA) custom resource

```
cat << EOF | oc apply -f -
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: dpa-sample
  namespace: ${OADP_PROJECT}
spec:
  backupImages: false
  backupLocations:
    - velero:
        accessMode: ReadWrite
        config:
          region: ${REGION}
          s3ForcePathStyle: "true"
          s3Url: ${S3_URL}
        credential:
          key: cloud
          name: cloud-credentials
        default: true
        objectStorage:
          bucket: ${BUCKET_NAME}
          prefix: ${BUCKET_PREFIX}
        provider: aws
  configuration:
    nodeAgent:
      enable: true
      podConfig:
        resourceAllocations:
          limits:
            cpu: "${NODE_AGENT_POD_CPU_LIMIT}"
            memory: 32Gi
          requests:
            cpu: 500m
            memory: 256Mi
        tolerations:
        - effect: NoSchedule
          key: icp4data
          operator: Exists
      timeout: 72h
      uploaderType: ${UPLOADER_TYPE}
    velero:
      customPlugins:
      - image: ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs/cpfs-oadp-plugins:${CPFS_OADP_PLUGIN_VERSION}
        name: cpfs-oadp-plugin
      - image: ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/cpdbr-velero-plugin:${VERSION}
        name: cpdbr-velero-plugin
      - image: ${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/swhub-velero-plugin:${VERSION}
        name: swhub-velero-plugin
      - image: ${PRIVATE_REGISTRY_LOCATION}/db2u/db2u-velero-plugin:${VERSION}
        name: db2u-velero-plugin
      defaultPlugins:
      - aws
      - openshift
      - csi
      podConfig:
        resourceAllocations:
          limits:
            cpu: "${VELERO_POD_CPU_LIMIT}"
            memory: 4Gi
          requests:
            cpu: 500m
            memory: 256Mi
      resourceTimeout: 60m
EOF
```

After you create the DPA, do the following checks.

<br>

Check that the velero pods are running in the ${OADP_PROJECT} project.

```
oc get po -n ${OADP_PROJECT}
```

The node-agent daemonset creates one node-agent pod for each worker node. Example output:

```
NAME                                                    READY   STATUS    RESTARTS   AGE
openshift-adp-controller-manager-678f6998bf-fnv8p       2/2     Running   0          55m
node-agent-455wd                                        1/1     Running   0          49m
node-agent-5g4n8                                        1/1     Running   0          49m
node-agent-6z9v2                                        1/1     Running   0          49m
node-agent-722x8                                        1/1     Running   0          49m
node-agent-c8qh4                                        1/1     Running   0          49m
node-agent-lcqqg                                        1/1     Running   0          49m
node-agent-v6gbj                                        1/1     Running   0          49m
node-agent-xb9j8                                        1/1     Running   0          49m
node-agent-zjngp                                        1/1     Running   0          49m
velero-7d847d5bb7-zm6vd                                 1/1     Running   0          49m
```

Verify that the backup storage location PHASE is Available.

```
cpd-cli oadp client config set namespace=${OADP_PROJECT}
```

```
cpd-cli oadp backup-location list
```

Example output:

```
NAME           PROVIDER    BUCKET             PREFIX              PHASE        LAST VALIDATED      ACCESS MODE
dpa-sample-1   aws         ${BUCKET_NAME}     ${BUCKET_PREFIX}    Available    <timestamp>
```

Check that the cpd-cli oadp version is 5.2.2:
```
cpd-cli oadp version
```

## 1.4 Installing shared cluster components for IBM Software Hub

https://www.ibm.com/docs/en/software-hub/5.2.x?topic=cluster-installing-shared-components

**Installing the Licensing and IBM Cert Manager**

Log the cpd-cli in to the Red Hat OpenShift Container Platform cluster:
```
${CPDM_OC_LOGIN}
```

Install the License Service for CPD 5.2.2 :
```
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```

**Note**
<br>
If the Red Hat OpenShift Container Platform cert-manager Operator is not installed, the IBM Certificate manager is automatically installed or upgraded when you run the `apply-cluster-components` command.

<br>

Wait for the cpd-cli to return the following message before proceeding to the next step:
```
[SUCCESS]... The apply-cluster-components command ran successfully.
```

**Cleaning up the target cluster after a previous restore**
This is only required if there was a previous restore.
[Cleaning up the target cluster after a previous restore](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=utility-offline-backup-restore-different-cluster#reference_acp_qwm_ddc__title__3)

**Installing the scheduling service**

```
cpd-cli manage apply-scheduler \
--release=${VERSION} \
--license_acceptance=true \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE}
```

## 2. Restoring the IBM Software Hub instance and services
**Note** You cannot restore a backup to a different project of the IBM Software Hub instance.
<br>
Log in to Red Hat OpenShift Container Platform as a cluster administrator:
```
${OC_LOGIN}
```

Restore IBM Software Hub by running below command.

```
nohup ./cpd-cli oadp tenant-restore create ${TENANT_OFFLINE_BACKUP_NAME}-restore \
--from-tenant-backup ${TENANT_OFFLINE_BACKUP_NAME} \
--image-prefix=${PRIVATE_REGISTRY_LOCATION}/ubi9 \
--verbose \
--log-level=debug &> ${TENANT_OFFLINE_BACKUP_NAME}-restore.log&
```

Get the status of the installed components:
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Ensure that the status of all of the services is `Completed` or `Succeeded`.

To view a list of restores, run the following command:
```
cpd-cli oadp tenant-restore list
```

To view the detailed status of the restore, run the following command:
```
cpd-cli oadp tenant-restore status ${TENANT_BACKUP_NAME}-restore --details
```

The command shows a varying number of sub-restores in the following form: `cpd-tenant-r-xxx`. You can view more information about these sub-restores by running the following command:
```
cpd-cli oadp restore status <SUB_RESTORE_NAME> --details
```

To view logs of the tenant restore, run the following command:
```
cpd-cli oadp tenant-restore log ${TENANT_BACKUP_NAME}-restore
```
## Completing post-restore tasks

### Applying RSI patches to the control plane
```
cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} -vvv
```

### Restarting IBM Knowledge Catalog lineage pods
Log in to Red Hat OpenShift Container Platform as a cluster administrator:
```
${OC_LOGIN}
```

Restart the wkc-data-lineage-service-xxx pod:
```
oc delete -n ${PROJECT_CPD_INST_OPERANDS} "$(oc get pods -o name -n ${PROJECT_CPD_INST_OPERANDS} | grep wkc-data-lineage-service)"
```

Restart the wdp-kg-ingestion-service-xxx pod:
```
oc delete -n ${PROJECT_CPD_INST_OPERANDS} "$(oc get pods -o name -n ${PROJECT_CPD_INST_OPERANDS} | grep wdp-kg-ingestion-service)"
```

## Reference
[Offline backup and restore to a different cluster with the IBM Software Hub OADP utility](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=utility-offline-backup-restore-different-cluster)
