# kube-apisix-linkerd-ory-kustomize

This is an example to deploy apisix as gateway with linkerd and the ory stack using kustomize

# Introduction

## What is GitOps?

GitOps is a way to do Continuous Delivery, it works by using Git as a source of truth for declarative infrastructure and workloads. For Kubernetes this means using `git push` instead of `kubectl apply/delete` or `helm install/upgrade`.

I've used GitHub to host the config repository and Flux as the GitOps delivery solution.

# Prerequisites

You will need the following tools installed:

- [Docker](https://docs.docker.com/install/)
- [k3d](https://k3d.io/v5.6.0/#releases)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [terraform](https://developer.hashicorp.com/terraform/install)
- [jq](https://stedolan.github.io/jq/download/)
- [pnpm](https://pnpm.js.org/en/installation) or any other package manager that can read the `package.json` file.

If you are using MacOS, you can install all the tools using [Homebrew](https://brew.sh/):

```bash
brew bundle
```

The complete list of tools can be found in the `Brewfile`.

# Getting Started

## Provisioning a Kubernetes cluster

I've used [k3d](https://k3d.io/v5.6.0/#releases) to create a Kubernetes cluster with 1 master and 2 worker nodes.

You can also use other tools like [kind](https://kind.sigs.k8s.io/) or [minikube](https://minikube.sigs.k8s.io/docs/start/), but you will need to modify the terraform scripts (`terraform/provisioning-local-cluster`).

When the cluster is ready, the terraform will install [Flux](https://fluxcd.io/) automatically and it will start to sync the cluster with the config repository (`infrastructure/flux`).

This will deploy the following components:

- [Linkerd](https://linkerd.io/) as the service mesh
- [APISIX](https://apisix.apache.org) as the gateway and ingress controller
- [Vault](https://www.vaultproject.io/) as the secrets manager
- [Cert Manager](https://cert-manager.io/) as the certificate manager with a Vault issuer
- [Weave GitOps](https://docs.gitops.weave.works/docs/intro-weave-gitops/) as the GitOps UI for Flux

```bash
pnpm run kube:cluster:create
```

## Configuring Vault

During the provisioning of the Kubernetes cluster, the deployment of Vault will fail because it needs to be initialized and unsealed. But don't worry, Flux will retry the deployment until it succeeds.

You can use the following command to initialize and unseal Vault:

```bash
pnpm kube:vault:init
pnpm kube:vault:unseal
```

Once Vault is ready, you can use the following command to configure Vault:

```bash
kubectl port-forward -n vault service/vault 8200:8200 &
export VAULT_ADDR=http://localhost:8200
export VAULT_TOKEN=$(cat vault-keys.json | jq -r '.root_token')
pnpm kube:vault:config
```

This will configure Vault by enabling the Kubernetes auth method, create linkerd pki secrets mount and create the necessary policies.

## Waiting for the cluster to be ready

Once Vault is ready and configured, Flux will start to deploy the rest of the components. You can use your best tool to check the status of the cluster, I've used [Weave GitOps](https://docs.gitops.weave.works/docs/intro-weave-gitops/) to check the status of the cluster.

```bash
kubectl port-forward -n flux-system service/weave-gitops 9001:9001
```

Open your browser and go to http://localhost:9001 to check the status of the cluster.

> Default username and password is `admin` and `flux` respectively.

# Contributing

Contributions are welcome. Please follow the standard Git workflow - fork, branch, and pull request.

# License

This project is licensed under the Apache 2.0 - see the `LICENSE` file for details.
