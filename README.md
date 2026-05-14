# DevPod on AKS

This repository is an AKS-first blueprint for running [DevPod workspaces](https://devpod.sh) on
Azure Kubernetes Service by using DevPod's built-in `kubernetes` provider.

- Run devcontainers on your private kubernetes cluster
- Run AI agents in remote sandboxes preconfigured with devcontainers.json files

## What This Repo Contains

- `infra/aks/`: AKS infrastructure definition in Bicep
- `hack/provision_aks.sh`: creates or updates the AKS cluster
- `hack/devpod_up_aks_smoke.sh`: configures DevPod against AKS and runs the
  smoke workspace
- `samples/aks-smoke/`: smallest supported workspace for first validation
- `samples/dotnet-hello-world/`: richer sample app for follow-up validation
- `docs/`: architecture notes, contributor guidance, runbooks, and roadmap

## SSH Over Kubernetes HTTPS/WebSocket

DevPod can let IDEs such as VS Code use the standard SSH protocol without
requiring workspace pods to expose SSH ports. The IDE speaks SSH to the local
SSH client, the SSH client uses DevPod as a `ProxyCommand`, and DevPod carries
the SSH bytes through `kubectl exec`.

The important enterprise networking detail is that `kubectl exec` talks to the
Kubernetes API server over HTTPS and upgrades that connection to a bidirectional
stream, usually WebSocket. In other words: SSH remains the IDE-facing protocol,
while the underlying transport is Kubernetes API HTTPS/WebSocket traffic. There
is normally no need to open port 22 to workspaces or manage workspace-specific
SSH firewall rules.

See [docs/devpod-ssh-byte-paths.md](docs/devpod-ssh-byte-paths.md) for the byte
path diagrams and security notes.

## Quick Start

Set the required Azure variables:

```bash
export AZURE_SUBSCRIPTION_ID="<subscription-id>"
export AZURE_REGION="westus2"
export AKS_RESOURCE_GROUP="devpod-aks-rg"
export AKS_NAME="devpod-aks"
```

Create a temporary SSH key for the AKS node pool:

```bash
ssh-keygen -q -t ed25519 -f /tmp/devpod-aks-ssh -N '' -C devpod-aks
export AKS_SSH_PUBLIC_KEY_FILE=/tmp/devpod-aks-ssh.pub
```

Provision the cluster:

```bash
./hack/provision_aks.sh
```

Run the first smoke workspace:

```bash
./hack/devpod_up_aks_smoke.sh
```

Verify the workspace:

```bash
DEVPOD_HOME="${DEVPOD_HOME:-/tmp/devpod-aks-home}" devpod list
DEVPOD_HOME="${DEVPOD_HOME:-/tmp/devpod-aks-home}" devpod ssh aks-smoke
kubectl --kubeconfig /tmp/devpod-aks-kubeconfig -n devpod-workspaces get pods,pvc
```

The detailed walkthrough lives in [docs/runbooks/aks-smoke.md](docs/runbooks/aks-smoke.md).

## Repo Layout

- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md): repo architecture and control
  flow
- [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md): contributor guide and validation
  commands
- [docs/roadmap.md](docs/roadmap.md): next improvements for the AKS path
- [docs/adrs/0001-aks-kubernetes-first.md](docs/adrs/0001-aks-kubernetes-first.md):
  architectural decision record for the pivot
- [samples/aks-smoke/README.md](samples/aks-smoke/README.md): smoke workspace
  notes
- [samples/dotnet-hello-world/README.md](samples/dotnet-hello-world/README.md):
  optional richer sample

## Operating Model

- The primary workflow is `DevPod CLI -> kubernetes provider -> AKS`.
- The repository owns only the AKS bootstrap assets and helper scripts.
- No custom DevPod provider is shipped from this repo anymore.

## Archived ACI Material

Historical ACI research is still available under `docs/archive/aci/` for
context, but it is no longer part of the supported path.
