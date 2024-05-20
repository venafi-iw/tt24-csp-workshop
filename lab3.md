# Lab 3 - SBOM Signing and Verification

### Pre-requisites
* Sigstore [cosign](https://github.com/sigstore/cosign) 1.3 or newer
* CodeSign Protect client
* Access to a container registry, e.g. Docker Hub
* Tools for generating Software Bill of Materials, e.g. [anchore/syft](https://github.com/anchore/syft)

### Signing Container Image SBOMs

1. Install Syft and Cosign
```bash
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
```
Install cosign via [Installation](https://docs.sigstore.dev/system_config/installation/) instructions.

2. Generate an SBOM in this example based on the Linux Foundation Project [SPDX](https://spdx.dev/learn/overview/) format:
```bash
syft docker-username/hello-container:latest -o spdx > hello-container-latest.spdx
```
3. Install and Configure CodeSign Protect client per online Venafi [documentation](https://docs.venafi.com)
4. Attach and Sign SBOM with cosign
```sbom
cosign attach sbom --sbom latest.spdx ${{ env.IMAGE }}
cosign sign --tlog-upload=false --key "pkcs11:slot-id=0;object={{ CERTIFICATE_LABEL }}?module-path=/opt/venafi/codesign/lib/venafipkcs11.so&pin-value=1234" {{ SBOM_IMAGE }}.sbom
```
5. Verify SBOM with cosign
```bash
cosign verify --insecure-ignore-tlog --key "pkcs11:slot-id=0;object={{ CERTIFICATE_LABEL }}?module-path=/opt/venafi/codesign/lib/venafipkcs11.so&pin-value=1234" {{ SBOM_IMAGE }}.sbom
cosign tree ${{ env.IMAGE }}
```

