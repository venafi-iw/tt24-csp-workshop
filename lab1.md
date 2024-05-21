# Lab 1 - Container Signing and Verification with Sigstore cosign and Kyverno

This lab will walk through creating a basic image, having it signed with cosign and CodeSign Protect, and then enforcing signature policies when deployed with Kyverno

Prequisites:
- CodeSign Protect client is installed and configure with at least 1 active code signing certificate environment
- cosign installed

## Instructions

### Install Lighweight Kubernetes (K3s):

Let’s first start by setting up a lightweight and certified version of Kubernetes in your environment.  K3s is packaged as a single < 50MB binary that reduces the dependencies and steps needed to install, run and auto-update a production Kubernetes cluster. 

```bash
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get node
```

### Install CodeSign Protect client

Follow instructions per documentation as well as from client distribution endpoint of your CodeSign Protect deployment 

e.g.
```bash
curl -o "venafi-codesigningclients-24.1.1-linux-x86_64.deb" \
https://vh.venafilab.com/csc/clients/venafi-csc-latest-x86_64.deb
```

```bash
sudo dpkg -i "venafi-codesigningclients-24.1.1-linux-x86_64.deb"
```

### Configure CodeSign Protect client

Follow instructions per documentation with the goal of getting a valid grant for signing:

```bash
pkcs11config getgrant --authurl=https://tpp/vedauth --hsmurl=https://tpp/vedhsm --username=sample-cs-user --password=NewPassw0rd! --force
```

### Setup Local Registry

In this part of the lab we will now setup a local stateless Docker-based [registry](https://docs.docker.com/registry/) so that we can store as well as distribute signed and unsigned container images that we’ve obtained from Docker hub.

```bash
sudo docker run -d -p 5000:5000 --restart always --name registry registry:2
sudo docker pull alpine:latest
sudo docker pull alpine:3.14.3
sudo docker image tag alpine:latest localhost:5000/alpine:signed
sudo docker image tag alpine:3.14.3 localhost:5000/alpine:unsigned
sudo docker image push localhost:5000/alpine:signed
sudo docker image push localhost:5000/alpine:unsigned
curl -X GET http://localhost:5000/v2/alpine/tags/list
```

### Sign with Sigstore cosign

We are now going to sign the alpine container image using the [GitHub - sigstore/cosign: Code signing and transparency for containers and binaries CLI](https://github.com/sigstore/cosign).  Cosign provides container signing, verification and storage in an OCI registry.  We will download a specific version that has support for PKCS#11, which is how we integrate with the Venafi CodeSign Protect client.

Verify that the OS has the required PKCS#11 libraries, such as on Ubuntu:

```bash
sudo apt install libpcsclite-dev
```

```bash
wget https://github.com/sigstore/cosign/releases/download/v2.2.4/cosign-linux-pivkey-pkcs11key-amd64
sudo chmod +x cosign-linux-pivkey-pkcs11key-amd64
sudo mv cosign-linux-pivkey-pkcs11key-amd64 /usr/bin/cosign
```

Now sign with cosign:

```bash
cosign sign --tlog-upload=false --key "pkcs11:slot-id=0;object=Sample-Development-Environment?module-path=/opt/venafi/codesign/lib/venafipkcs11.so&pin-value=1234" localhost:5000/alpine:signed
```

### Install Kyverno

Kyverno is a policy engine designed for Kubernetes. It can validate, mutate, and generate configurations using admission controls and background scans. Kyverno policies are Kubernetes resources and do not require learning a new language. Kyverno is designed to work nicely with tools you already use like kubectl, kustomize, and Git.

For this part of the lab we will now be focusing on how to validate and enforce policy around container image signatures in a Kubernetes cluster.  Let’s get started by installing the Kyverno admission controller:

```bash
sudo kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.11.1/install.yaml
```

### Configure Kyverno Image Verification Policy

We will now setup a Kyverno image verification policy that will leverage A Venafi CodeSign Protect code signing certificate. 

#### Get Certificate:

```bash
pkcs11config getcertificate -label:Sample-Development-Environment -file:sample.crt
```

Let’s now create a sample image verification policy that will enforce signature verification against the alpine images you distributed in the earlier part of this lab.

```bash
vi venafi_csp_verify_policy.yaml
```
Paste in the following template and make sure to add in the certificate you obtained earlier:

```bash
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature
      match:
        any:
        - resources:
            kinds:
              - Pod
      verifyImages:
      - imageReferences:
        - "10.1.0.12:5000/alpine:*"
        attestors:
        - entries:
          - certificates:
              cert: |-
                -----BEGIN CERTIFICATE-----
                MIIDuzCCAqOgAwIBAgIUDG7gFB8RMMOMGkDm6uEusOE8FWgwDQYJKoZIhvcNAQEL
                BQAwbDELMAkGA1UEBhMCVVMxCzAJBgNVBAgMAkNBMQwwCgYDVQQHDANTSkMxEDAO
                BgNVBAoMB05pcm1hdGExEDAOBgNVBAMMB25pcm1hdGExHjAcBgkqhkiG9w0BCQEW
                D2ppbUBuaXJtYXRhLmNvbTAeFw0yMjA0MjgxOTU0NDFaFw0yNDA3MzExOTU0NDFa
                MGwxCzAJBgNVBAYTAlVTMQswCQYDVQQIDAJDQTEMMAoGA1UEBwwDU0pDMRAwDgYD
                VQQKDAdOaXJtYXRhMRAwDgYDVQQDDAduaXJtYXRhMR4wHAYJKoZIhvcNAQkBFg9q
                aW1AbmlybWF0YS5jb20wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDP
                LObWc4VM4CULHrjScdjAXwdeJ6o1SwS9Voz9wTYAASp54EDqgzecWGjtn409NF9o
                4tqd5LotEFscoMXGpmm7dBpv76MQhGym7JBhlYaBksmnKp17nTfAmsgiDiUnjnG6
                BQ5/FIdZYHtpJmMZ/SZqQ3ehXLaGj2qogPrEsObN1S/1b+0guLC/gVi1fiuUgd4Z
                SDEmDaLjSuIQBrtba08vQnl5Ihzrag3A85+JNNxk9WBDFnLHMsRvlrUMU4565FS9
                X57epDZakKvLATAK0/gKI2ZvWfY0hoO3ngEk4Rkek6Qeh1vXFBc8Rsym8W0RXjux
                JDkye5RTsYrlXxSavP/xAgMBAAGjVTBTMB8GA1UdIwQYMBaAFBF3uwHovsxj7WxS
                vDDKBTwuR+oaMAkGA1UdEwQCMAAwCwYDVR0PBAQDAgTwMBgGA1UdEQQRMA+CDWhl
                bGxmaXNoLnRlc3QwDQYJKoZIhvcNAQELBQADggEBAHtn9KptJyHYs45oTsdmXrO0
                Fv0k3jZnmqxHOX7OiFyAkpcYUTezMYKHGqLdme0p2VE/TdQmGPEq1dqlQbF7UMb/
                o+SrvFpmAJ1iAVjLYQ7KDCE706NgnVkxaPfU8UBOw2vF5nsgIxcheOyxTplbVOVM
                vcYYwAWXxkNhrQ4sYygXuNgZawruxY1HdUgGWlh9XY0J5OBrXyinh2YGBUGQJgQR
                NEmM+GQjdquPqAgDsb3kvWgFDrcbBZJBc/CyZU8GH9uIuPDgfVhDTqFtiz9W/F5s
                Hh8yD7VAIWgL9TkGWRwWdD6Qx/BAu7dMdpjAxdGpMLn3O4SDAZDnQneaHx6qr/I=
                -----END CERTIFICATE-----
```

#### Apply Policy and deploy a signed image:

```bash
sudo kubectl apply -f venafi_csp_verify_policy.yaml
```

```bash
sudo kubectl run signed --image=10.1.0.12:5000/alpine:signed
```

#### Deploy an unsigned/untrusted image:

Now let’s take a look at a scenario where we attempt to run an unsigned version of the container image and see what happens.

```bash
sudo kubectl run unsigned --image=10.1.0.12:5000/alpine:unsigned
```

In this case we get a signature not found error since the version of alpine that we are attempting to run wasn’t signed.  If we signed with a different certificate/keypair then we might get a signature mismatch error.

#### View Kyverno Logs

While you are attempting to deploy signed or unsigned images with Kyverno policy management in action you may want to see the real-time logs, which you can obtain as follows:

```bash
sudo kubectl logs -n kyverno $(sudo kubectl get pod -n kyverno |  awk '/kyverno/{print $1}') -f
```

### Troubleshooting
