# Lab 2 - Container Image Signing and Verification with The Notary Project notation and Kyverno

This lab will walk through creating a basic image, having it signed with notation and CodeSign Protect, and then enforcing signature policies when deployed with Kyverno

Prequisites:
- CodeSign Protect configured with at least 1 active code signing certificate environment
- notation CLI
- CodeSign Protect plugin for [notation](https://github.com/venafi/notation-venafi-csp)

## Instructions

### Install Lighweight Kubernetes (K3s):

Let’s first start by setting up a lightweight and certified version of Kubernetes in your environment.  K3s is packaged as a single < 50MB binary that reduces the dependencies and steps needed to install, run and auto-update a production Kubernetes cluster. 

```bash
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get node
```

### Install notation CLI

Download the latest stable release of the [notation](https://notaryproject.dev/docs/user-guides/installation/cli/) CLI using Homebrew:

For example on MacOS:

```bash
brew install notation
```

Linux:

```bash
curl -LO https://github.com/notaryproject/notation/releases/download/v1.1.0/notation_1.1.0\_linux_amd64.tar.gz
sudo tar xzvf notation_1.1.0_linux_amd64.tar.gz -C /usr/bin/ notation
```

### Install CodeSign Protect notation plugin

To find out more about the Venafi Plugin, please refer to this [GitHub repository](https://github.com/venafi/notation-venafi-csp).

#### Install from URL:
```bash
notation plugin install --url https://github.com/Venafi/notation-venafi-csp/releases/download/v0.3.0/notation-venafi-csp-linux-amd64.tar.gz --sha256sum 03771794643f18c286b6db3a25a4d0b8e7c401e685b1e95a19f03c9356344f5a
Successfully installed plugin venafi-csp, version 0.3.0-release
```

#### Install from local file:
```bash
notation plugin install --file notation-venafi-csp-linux-amd64.tar.gz
Successfully installed plugin venafi-csp, version 0.3.0-release
```

To confirm you plugin is installed, run notation plugin list. For example:

```bash
notation plugin list
```

### Configure CodeSign Protect notation plugin

The CodeSign Protect notation plugin leverages the [vSign SDK](https://github.com/venafi/vsign), which requires a configuration file for secure connectivity via API to CodeSign Protect:

#### Create configuration file (config.ini)

Replace parameters with your specific instance details:

```
tpp_url = "https://tpp.example.com" 
access_token = "xxx"
tpp_project = "Project\\Environment"
```

You can use [vcert](https://github.com/venafi/vcert) to obtain the necessary access token:

```bash
vcert getcred --scope "codesignclient;codesign" -u https://tpp.example.com --client-id vsign-sdk --username sample-cs-user --password NewPassw0rd!
```

### Add Code Signing identity to notation

You should use the certificate label that matches the Venafi CodeSign Protect environment obtained using pkcs11config:

```bash
pkcs11config getcertificate <...>
```

This certificate label will be represented as the notation key ID

```bash
notation key add --default "sample-development-environment" --plugin venafi-csp --id "sample-development-environment" --plugin-config "config=config.ini"
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

### Sign with notation

We are now going to sign the alpine container image using the notation CLI.  Notation provides container signing, verification and storage in an OCI registry.  

Now sign with notation:

```bash
notation sign --key "sample-development-environment" localhost:5000/alpine:signed
```

### Install Kyverno

Kyverno is a policy engine designed for Kubernetes. It can validate, mutate, and generate configurations using admission controls and background scans. Kyverno policies are Kubernetes resources and do not require learning a new language. Kyverno is designed to work nicely with tools you already use like kubectl, kustomize, and Git.

For this part of the lab we will now be focusing on how to validate and enforce policy around container image signatures in a Kubernetes cluster.  Let’s get started by installing the Kyverno admission controller:

```bash
sudo kubectl create -f https://github.com/kyverno/kyverno/raw/main/config/install-latest-testing.yaml
```

### Install Kyverno Extension service for CodeSign Protect notation plugin

Complete instructions can be found [here](https://github.com/nirmata/kyverno-notation-venafi)

#### Install extension service

```bash
sudo kubectl create -f https://raw.githubusercontent.com/nirmata/kyverno-notation-venafi/main/configs/install.yaml
```

CRDs
```bash
sudo kubectl create -f https://raw.githubusercontent.com/nirmata/kyverno-notation-venafi/main/configs/crds/notation.nirmata.io_trustpolicies.yaml
sudo kubectl create -f https://raw.githubusercontent.com/nirmata/kyverno-notation-venafi/main/configs/crds/notation.nirmata.io_truststores.yaml
```

#### Create TrustStore policy

```yaml
apiVersion: notation.nirmata.io/v1alpha1
kind: TrustStore
metadata:
  name: venafi-signer-ts
spec:
  trustStoreName: venafidemo.com
  type: ca
  caBundle: |-
    -----BEGIN CERTIFICATE-----
    MIID7zCCAtegAwIBAgIQNOVKnFAQ5UeIsb+Gp9XjvDANBgkqhkiG9w0BAQsFADCB
    iTELMAkGA1UEBhMCVVMxDTALBgNVBAgTBFV0YWgxFzAVBgNVBAcTDlNhbHQgTGFr
    ZSBDaXR5MSgwJgYDVQQKEx9TYW1wbGUgQ29kZSBTaWduZXJzIEFyZSBVcywgTExD
    MSgwJgYDVQQDEx9TYW1wbGUgQ29kZSBTaWduZXJzIEFyZSBVcywgTExDMB4XDTI0
    MDUyMTE2MjU0OFoXDTI1MDUyMTE2MjU0OFowgYkxCzAJBgNVBAYTAlVTMQ0wCwYD
    VQQIEwRVdGFoMRcwFQYDVQQHEw5TYWx0IExha2UgQ2l0eTEoMCYGA1UEChMfU2Ft
    cGxlIENvZGUgU2lnbmVycyBBcmUgVXMsIExMQzEoMCYGA1UEAxMfU2FtcGxlIENv
    ZGUgU2lnbmVycyBBcmUgVXMsIExMQzCCASIwDQYJKoZIhvcNAQEBBQADggEPADCC
    AQoCggEBAL7Swjz/n2Ag9eCT71TdjjSNmgO9LRpsNtqEZAgyy44V+FXheYxshA92
    urgcrEeOEtNQ83NN0xFflwd/lTJJXLuwSb7oL6HZTuu03dqd+c2c4Eexr5U0KbXB
    E7FjrhGkds2JM/VgFybGZT3VIaX5kNRpRc/xZGOMbuKlb8AsjB2EPzuT7scbXpPO
    nC5zxycNXa10tK5Sr0sI1VTdzFpph8OBDIhIExEGI59t0JcaheOE0vpf6D1l6OVe
    vZF7HsKTQJ6BEAEtViFhEYc2v/l8Iw5+liw4NZdfcHTrJkDOE1XQnErHUFdbHm6t
    Bpt0huK4yhD+NZqa2KSYAML+U7WSbMkCAwEAAaNRME8wHQYDVR0OBBYEFFp4894m
    DK4gK44LDrnlfJ8GWqjtMAkGA1UdEwQCMAAwDgYDVR0PAQH/BAQDAgaAMBMGA1Ud
    JQQMMAoGCCsGAQUFBwMDMA0GCSqGSIb3DQEBCwUAA4IBAQBL9K2hzUYayPzvPlQC
    mw01kVfi8zR7HdjFTfxOQ6mgLrFDuYQG9vvdCyhHzz8SLfaaq2VoZ/ZpDeIb+B15
    H7WWTYDvrFJRHOQ9BUcZzeZGizanpR9XIQWCpXm9XvFmQ/AqxGEuptg5ialtdBPd
    hrDwfiOQEH9cfJiRsM5g9DiFjqHgoeUw6WD+bX3nGG9VeHHXhQ43upmDWFNvdskt
    pNQY7vB4yRsvkfzwdPmn5bQ+OfEhYqy8c4wKUtcmzsGpw3WAEOyGIT5Wm3tAo28s
    2vMjZph4Dr42DuljeYmJv38CKbnSHVtCSbZJ5rxbsq/QYE2bOeegqF3t8cXffOcE
    2Bsc
    -----END CERTIFICATE-----
```

```bash
kubectl apply -f truststore.yaml
```

#### Create Trust Policy

```yaml
apiVersion: notation.nirmata.io/v1alpha1
kind: TrustPolicy
metadata:
  name: venafi-trustpolicy-sample
spec:
  version: '1.0'
  trustPolicyName: tp-venafi-test-notation
  trustPolicies:
  - name: venafi-signer-tp
    registryScopes:
    - "*"
    signatureVerification:
      level: strict
      override: {}
    trustStores:
    - ca:venafidemo.com
    trustedIdentities:
    - "*"
```

```bash
kubectl apply -f trustpolicy.yaml
```

#### Apply Policy to Cluster

```bash
sudo kubectl create -f https://raw.githubusercontent.com/nirmata/kyverno-notation-venafi/main/configs/samples/kyverno-policy.yaml
```

#### Deploy a signed image:

Create the test namespace which the policy applies to:

```bash
kubectl create ns test-venafi
```

```bash
sudo kubectl -n test-venafi run signed --image=10.42.0.1:5000/alpine:signed
```

#### Deploy an unsigned/untrusted image:

Now let’s take a look at a scenario where we attempt to run an unsigned version of the container image and see what happens.

```bash
sudo kubectl -n test-venafi run unsigned --image=10.42.0.1:5000/alpine:unsigned
```

In this case we get a signature not found error since the version of alpine that we are attempting to run wasn’t signed.  If we signed with a different certificate/keypair then we might get a signature mismatch error.

#### View Kyverno Logs

While you are attempting to deploy signed or unsigned images with Kyverno policy management in action you may want to see the real-time logs, which you can obtain as follows:

```bash
sudo kubectl logs -n kyverno $(sudo kubectl get pod -n kyverno |  awk '/kyverno/{print $1}') -f
```

### Troubleshooting

* View kyverno per above as well as TPP logs in case of any signing issues
