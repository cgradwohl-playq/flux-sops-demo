# FluxCD SOPS Demo

In this repository we present a workflow for encrypting and decrypting secrets
using SOPS and FluxCD.

## Overview

In order for Flux to be able to decrypt locally encrypted secrets, it needs
access to the private encryption key. See
[Configuring Flux for SOPS decryption](#configuring-flux-for-sops-decryption).

Once flux has the key, the workflow to add secrets consists of three steps.

1. Create and encrypt the secret locally using the public key.
2. Add the encrypted secret file to `secretGenerator`
3. Push the changes to main

Flux will then create Kubernetes Secret Objects from the encrypted secrets.
Kubernetes only receives decrypted secrets and Flux always decrypts them before
applying. The cluster never stores encrypted SOPS data in any Secret object.

If you want to encrypt secrets at rest we have three options:

1. SOPS + Kubernetes Secrets Encryption
   https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/

- Keeps SOPS encryption at the Git/repo layer, before Flux applies.
- KMS encryption is at the cluster layer, after Flux applies.

2. SealedSecrets + Kubernetes Secrets Encryption
   https://fluxcd.io/flux/guides/sealed-secrets/

- encrypt secrets using the SealedSecrets public key instead of SOPS
- Git stores SealedSecrets, not SOPS-encrypted files
- Flux does not need to handle SOPS decryption anymore — it just applies the
  SealedSecret YAML

3. External Secrets Operator + Kubernetes Secrets Encryption

- a Kubernetes controller that lets your cluster consume secrets from external
  secret stores, such as AWS Secrets Manager or AWS SSM Parameter Store
- Instead of storing secrets in Git, ESO fetches them at runtime and creates
  Kubernetes Secret objects in your cluster

## Demo

Repo Overview: https://www.loom.com/share/26831d942e6243c1aa32cbfee78ab164

Cluster Demo: https://www.loom.com/share/49f0802986374e54baacb957fdf05e4d

Workflow Demo: https://www.loom.com/share/97f722fe90754ff5abcbc15051e8195a

## Prerequisites

If you would like to run this code locally you will need the following
installed:

- Docker - https://www.docker.com/products/docker-desktop/
- minikube - https://minikube.sigs.k8s.io/docs/
- kubectl - https://kubernetes.io/docs/tasks/tools/
- flux cli - https://fluxcd.io/flux/installation/
- github PAT with full repo privileges -
  https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens

## Documentation and References

- https://fluxcd.io/flux/component
- https://fluxcd.io/flux/guides/mozilla-sops/
- https://fluxcd.io/flux/installation/bootstrap/github/

## Folder Structure

We are going to bootstrap our minikube cluster from scratch using a monorepo
style folder structure.

All of our FluxCD configuration and gitops pipeline definitions will live inside
the `flux-cd/` folder.

```bash
flux-cd
├── apps
│   ├── demo-app-1
│   ├──── gitrepository.yaml # Flux GitRepository CRD for demo-app-1
│   ├──── kustomization.yaml # Flux Kustomization CRD for demo-app-1
└── clusters
    ├── minikube
```

Inside our `flux-cd/` folder we will configure a unique gitops pipeline for
`demo-app-1` using the GitRepository and Kustomization CRDs.

Adjacent to our flux infrastructure repository we will create the kubenetes
manifest for `demo-app-1`.

```bash
demo-app-1/
├── deploy
│   ├── pod.yaml
│   ├── secrets
│   ├──── kustomization.yaml # secrets generator
│   ├──── sops-age.key # private encryption key (.gitignore)
│   ├──── password.txt
│   ├──── username.txt
```

Overall our repo will look something like this:

```bash
flux-sops-demo/
├── flux-cd/
│   ├── apps/
│   │   └── demo-app-1/
│   │       ├── gitrepository.yaml       # Flux GitRepository CRD for demo-app-1
│   │       └── kustomization.yaml       # Flux Kustomization CRD for demo-app-1
│   └── clusters/
│       └── minikube/                     # Flux system manifests (gotk-components.yaml, etc.)
├── apps/
│   └── demo-app-1/
│       ├── deploy/
│       │   ├── pod.yaml                  # Example pod consuming secrets
│       │   └── secrets/
│       │       ├── kustomization.yaml    # secretGenerator
│       │       ├── username.txt          # SOPS-encrypted
│       │       ├── password.txt          # SOPS-encrypted
│       │       ├── sops-age.key          # private encryption key
│       │       └── .sops.yaml            # local encryption config
```

## Bootstrapping

First ensure you have a fresh minikube cluster:

```bash
minikube delete # beware this nukes your minikube cluster
minikube start
```

To bootstrap Flux onto our cluster we can run the following commands:

```bash
export GITHUB_TOKEN=your-token
export GITHUB_USER=your-username
```

followed by

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=your-repo-name \ # consider changing this!
  --branch=main \
  --path=./flux-cd/clusters/minikube \
  --personal
```

Finally clone the repo:

```bash
git clone https://github.com/$GITHUB_USER/your-repo-name
cd your-repo-name
```

## Creating the GitOps pipeline

```bash
mkdir flux-cd/apps
mkdir flux-cd/apps/demo-app-1

touch flux-cd/apps/demo-app-1/gitrepository.yaml
touch flux-cd/apps/demo-app-1/kustomization.yaml
```

The contents of these files are found in this repo.

Once the pipeline has been configured we can apply these CRDs to the cluster.

```bash
kubectl apply -f flux-cd/apps/demo-app-1/gitrepository.yaml
kubectl apply -f flux-cd/apps/demo-app-1/kustomization.yaml

```

We can now describe these resource:

```bash
kubectl -n flux-system get gitrepository demo-app-1
kubectl -n flux-system get kustomization demo-app-1
```

## Creating demo-app-1 and secrets This demo application will be a single pod that reads in two secrets.

```bash
#
mkdir apps
mkdir apps/demo-app-1
mkdir apps/demo-app-1/deploy
mkdir apps/demo-app-1/deploy/secrets

touch apps/demo-app-1/deploy/pod.yaml
touch apps/demo-app-1/deploy/secrets/kustomization.yaml
touch apps/demo-app-1/deploy/secrets/username.txt
touch apps/demo-app-1/deploy/secrets/password.txt
touch apps/demo-app-1/deploy/secrets/.sops.yaml
```

The contents of these files are found in this repo.

### Creating the encrypted secret locally

First we configure sops to use our public key for encryption via our .sops.yaml
file.

Next we create and encrypt our two secrets:

```bash
cd apps/demo-app-1/deploy/secrets/

sops --encrypt --in-place username.txt
sops --encrypt --in-place password.txt
```

Finally we create the Kubernetes Secret Object using `secretsGenerator` in our
kustomization.yaml file.

### Configuring Flux for SOPS decryption

First we make the age private key available in-cluster in the flux-system
namespace so that Flux can use it to decrypt. Note, that this is part of the
Kustomization CRD definition we created in `flux-cd\` folder.

```bash
kubectl -n flux-system create secret generic sops-age \
  --from-file=age.agekey=./apps/demo-app-1/deploy/secrets/sops-age.key \
  --dry-run=client -o yaml | kubectl apply -f -
```

## validation

First we can confirm that the secrets are created and decrypted:

```bash
# show secret exists
kubectl get secret demo-app-1-secrets -n default

# show the resource (data is base64)
kubectl get secret demo-app-1-secrets -n default -o yaml

# decode and print the username/password — expected: plaintext (not SOPS JSON)
kubectl get secret myapp-secrets -n default -o jsonpath='{.data.username}' | base64 --decode ; echo
kubectl get secret myapp-secrets -n default -o jsonpath='{.data.password}' | base64 --decode ; echo
```

Now we can exec into the running pod and verify the secrets are available as
plain text:

```bash
kubectl exec -it pod/demo-app-1 -n default -- sh -c 'echo "DB_USERNAME=$DB_USERNAME"; echo "DB_PASSWORD=$DB_PASSWORD"'
```

## age.agekey

This repo uses an age kehy as an exmaple.

Public key: age1caenensw6jcpfzldye9hsrexyglua2xm3lgjdrkz2743s4hsj4zs7psnqw
