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

```bash
wget https://github.com/sigstore/cosign/releases/download/v2.2.4/cosign-linux-pivkey-pkcs11key-amd64
sudo chmod +x cosign-linux-pivkey-pkcs11key-amd64
sudo mv cosign-linux-pivkey-pkcs11key-amd64 /usr/bin/cosign
```

Now sign with cosign:

```bash
cosign sign --key "pkcs11:slot-id=0;object=Sample-Development-Environment?module-path=/opt/venafi/codesign/lib/venafipkcs11.so&pin-value=1234" localhost:5000/alpine:signed
```

### Install Kyverno

### Configure Kyverno Image Verification Policy

### View Kyverno Logs

### Troubleshooting
