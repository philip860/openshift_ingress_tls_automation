# Duncan Networks Ingress-Based TLS Automation

This project shows how to use cert-manager in OpenShift to build an internal PKI, issue application certificates from a Duncan Networks internal CA, and automate TLS for applications by using an annotated Ingress instead of manually pasting certificates into a Route.

## Recommended GitHub repository name

**openshift-ingress-tls-automation**

Why this name fits:
- `openshift` keeps the platform clear
- `ingress-tls` reflects the automation method being used
- `automation` matches the purpose of the repository

Other good options:
- `openshift-cert-manager-ingress-pki`
- `duncan-networks-openshift-pki`
- `openshift-internal-pki-automation`

## What this project does

This workflow creates:

1. A self-signed bootstrap issuer
2. A Duncan Networks root CA certificate
3. A CA-backed ClusterIssuer that uses that root CA
4. An example application certificate secret
5. An annotated Ingress that lets cert-manager manage TLS automatically

In OpenShift, this avoids manually embedding `tls.crt` and `tls.key` directly into a Route manifest.

## Why this method is recommended

Instead of managing TLS in a Route by hand, this project uses:
- cert-manager to issue and renew certificates
- a Kubernetes Ingress annotated with the cert-manager issuer
- OpenShift's ingress handling to expose the application

This gives you a cleaner GitOps workflow and automatic renewal.

## Important trust note

A self-signed certificate by itself does **not** remove browser trust warnings.

To fix browser trust issues, you must:

1. Create your own root CA
2. Use that CA to sign application certificates
3. Import the root CA certificate into the client OS or browser trust store

## File layout

```text
manifests/
├── 01-clusterissuer-selfsigned.yaml
├── 02-certificate-root-ca.yaml
├── 03-clusterissuer-ca.yaml
├── 04-certificate-app-example.yaml
└── 05-ingress-example.yaml
```

## Prerequisites

- OpenShift cluster
- cert-manager Operator installed
- Access to the `oc` CLI
- A target namespace for the example app, such as `demo-app`
- A Service named `demo-app` listening on port `8080`

## Process and order of operations

Apply the manifests in this order.

### Step 1: Create the self-signed bootstrap issuer

This issuer is used only to create the initial root CA.

```bash
oc apply -f manifests/01-clusterissuer-selfsigned.yaml
```

### Step 2: Create the Duncan Networks root CA certificate

This creates a root CA and stores it in the `duncan-networks-root-ca-secret` secret in the `cert-manager` namespace.

```bash
oc apply -f manifests/02-certificate-root-ca.yaml
```

Check that the secret was created:

```bash
oc get secret duncan-networks-root-ca-secret -n cert-manager
```

### Step 3: Create the CA-backed issuer

This issuer signs future application certificates using the Duncan Networks root CA.

```bash
oc apply -f manifests/03-clusterissuer-ca.yaml
```

### Step 4: Create the example application certificate resource

This tells cert-manager to issue a certificate for `demo-app.apps.duncan-networks.local` and store it in the `demo-app-tls` secret.

```bash
oc apply -f manifests/04-certificate-app-example.yaml
```

Check that the TLS secret exists:

```bash
oc get secret demo-app-tls -n demo-app
```

### Step 5: Create the annotated Ingress

This is the automation step that replaces manual Route TLS handling.

The Ingress:
- references the hostname
- references the TLS secret name
- tells cert-manager which issuer to use
- points to the `demo-app` Service on port `8080`

```bash
oc apply -f manifests/05-ingress-example.yaml
```

Verify it:

```bash
oc get ingress -n demo-app
```

## How the automated TLS flow works

1. You create the Ingress with the cert-manager annotation
2. cert-manager sees the Ingress and manages the certificate lifecycle
3. cert-manager writes the cert and key into `demo-app-tls`
4. OpenShift uses that ingress configuration to expose the app
5. When the certificate is renewed, the secret is updated automatically

This is the main advantage over manually embedding PEM blocks in a Route.

## Export the root CA for browser trust

Export the CA certificate:

```bash
oc get secret duncan-networks-root-ca-secret -n cert-manager \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > duncan-networks-root-ca.crt
```

## Install the CA into the client trust store

### Windows

- Open `duncan-networks-root-ca.crt`
- Choose **Install Certificate**
- Place it in **Trusted Root Certification Authorities**

### macOS

- Open **Keychain Access**
- Import `duncan-networks-root-ca.crt` into the **System** keychain
- Mark it as **Always Trust**

### RHEL / CentOS / Fedora

```bash
sudo cp duncan-networks-root-ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
```

### Debian / Ubuntu

```bash
sudo cp duncan-networks-root-ca.crt /usr/local/share/ca-certificates/duncan-networks-root-ca.crt
sudo update-ca-certificates
```

## Verification commands

Describe the certificate:

```bash
oc describe certificate demo-app-cert -n demo-app
```

Review the TLS secret:

```bash
oc get secret demo-app-tls -n demo-app -o yaml
```

Check the ingress:

```bash
oc describe ingress demo-app -n demo-app
```

## Notes

- Replace `demo-app.apps.duncan-networks.local` with your real application hostname.
- Replace the `demo-app` namespace with the namespace that hosts your application.
- Ensure the backend Service name and port match your application.
- If you want browsers to trust the certificate, the root CA must be installed on the client side.
- For production internet-facing apps, a public CA such as Let's Encrypt is usually a better fit.

## Suggested resource naming convention

- `duncan-networks-selfsigned-issuer`
- `duncan-networks-root-ca`
- `duncan-networks-root-ca-secret`
- `duncan-networks-ca-issuer`
- `demo-app-cert`
- `demo-app-tls`
- `demo-app`

## Quick apply sequence

```bash
oc apply -f manifests/01-clusterissuer-selfsigned.yaml
oc apply -f manifests/02-certificate-root-ca.yaml
oc apply -f manifests/03-clusterissuer-ca.yaml
oc apply -f manifests/04-certificate-app-example.yaml
oc apply -f manifests/05-ingress-example.yaml
```
