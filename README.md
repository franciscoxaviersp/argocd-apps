# ArgoCD Apps

This repository contains ArgoCD Application manifests for managing Kubernetes deployments using GitOps principles.

## Repository Structure

```
.
├── clusters/          # Cluster-specific App of Apps
│   └── homelab/      # Homelab cluster App of Apps
├── infrastructure/   # Infrastructure applications
│   ├── homelab/     # Homelab infrastructure apps
│   └── other/       # Shared infrastructure apps
└── media/           # Media server applications
    └── homelab/     # Homelab media apps
```

## Applications

### Infrastructure (Homelab)

- **cert-manager**: Certificate management and automation
- **envoy**: Envoy proxy with CRDs
- **external-dns-public**: External DNS for public domains
- **kyverno**: Policy management for Kubernetes
- **metallb**: Load balancer for bare metal Kubernetes
- **rook-ceph**: Distributed storage system (operator and cluster)
- **vpa**: Vertical Pod Autoscaler

### Infrastructure (Other)

- **haproxy**: HAProxy load balancer
- **ingress-nginx**: NGINX Ingress Controller (version 4.13.0)

### Media (Homelab)

- **arr-stack**: Media management automation stack
- **jellyfin**: Media streaming server

## Cluster Apps of Apps

The `clusters/` directory contains Apps of Apps that automatically deploy applications to specific clusters (directory name).

The `clusters/homelab/` directory contains the following applications:

- `infra.yaml`: Infrastructure applications
- `infra-private.yaml`: Private infrastructure applications
- `media.yaml`: Media applications

## CI/CD Workflows

This repository uses GitHub Actions to automate validation and deployment of App of Apps manifests.

### Validation Workflows (Pull Requests)

#### `yamllint`
- **Trigger**: On pull requests
- **Purpose**: Validates YAML syntax across all manifests
- **Tools**: yamllint with custom configuration

#### `kubeconform`
- **Trigger**: On pull requests
- **Purpose**: Validates Kubernetes manifests against schemas
- **Features**:
  - Validates against default Kubernetes schemas
  - Includes CRD validation from [datreeio/CRDs-catalog](https://github.com/datreeio/CRDs-catalog)
  - Validates `infrastructure/`, `clusters/`, and `media/` directories

### Deployment Workflows

#### `dispatch` (Workflow Dispatcher)
- **Trigger**: When pull requests are merged (closed)
- **Purpose**: Automatically detects changed clusters and dispatches deployments
- **Process**:
  1. Detects which clusters have changes in `clusters/*/` directory
  2. Identifies unique clusters affected by the PR
  3. Dispatches `cluster_deploy` workflow for each affected cluster
  4. Passes base SHA and merge commit SHA for diffing

#### `cluster_deploy` (Cluster Deployment)
- **Trigger**: Manual dispatch or automated via `dispatch` workflow
- **Purpose**: Deploys App of Apps to the specified cluster
- **Features**:
  - Connects to cluster via OpenVPN tunnel
  - Supports cluster-specific configurations (currently: `homelab`)
  - Can deploy all files or only changed files (diff-based)
  - Uses cluster-specific secrets for VPN and kubeconfig
- **Process**:
  1. Checkout repository
  2. Generate diff between base and merge commit (if provided)
  3. Setup kubectl and OpenVPN client
  4. Configure VPN credentials from GitHub secrets
  5. Setup kubeconfig from GitHub secrets
  6. Connect to VPN tunnel
  7. Apply changed manifests or all cluster manifests to ArgoCD

#### `test_vpn` (VPN Testing)
- **Trigger**: Manual dispatch
- **Purpose**: Validates VPN connectivity to clusters
- **Features**:
  - Tests OpenVPN tunnel creation
  - Verifies `tun0` interface is active
  - Can be uncommented in `dispatch` workflow for automated testing

### Required GitHub Secrets

For each cluster (e.g., `homelab`), the following secrets must be configured:

- `OPENVPN_CONF_<CLUSTER>`: OpenVPN configuration file
- `OPENVPN_CA_<CLUSTER>`: Certificate Authority certificate
- `OPENVPN_CERT_<CLUSTER>`: Client certificate
- `OPENVPN_KEY_<CLUSTER>`: Client private key
- `OPENVPN_TLS_<CLUSTER>`: TLS authentication key
- `OPENVPN_CREDENTIALS_<CLUSTER>`: VPN authentication credentials
- `KUBECONFIG_<CLUSTER>`: Kubernetes configuration file
- `TRIGGER_WORKFLOW_TOKEN`: GitHub token for triggering workflows

## Usage

### Automated Deployment (Recommended)

1. Make changes to App of Apps manifests in `clusters/<cluster-name>/`
2. Create a pull request
3. CI validates YAML syntax and Kubernetes schemas
4. Upon merge, the `dispatch` workflow automatically:
   - Detects affected clusters
   - Triggers deployment for each cluster
   - Applies only changed manifests

### Manual Deployment

1. Ensure ArgoCD is installed and configured in your cluster
2. Apply the Apps of Apps to deploy all applications:
   ```bash
   kubectl apply -f clusters/homelab/
   ```
3. ArgoCD will automatically sync and deploy the applications based on the ApplicationSet patterns

## Sync Policy

Most applications use automated sync with:
- **Prune**: Remove resources when removed from git
- **Self-Heal**: Automatically sync when cluster state drifts from git

## Requirements

- Kubernetes cluster
- ArgoCD installed
- Appropriate storage classes for persistent volumes
- LoadBalancer support (MetalLB for bare metal)
- OpenVPN server for secure cluster access (for CI/CD)
- GitHub repository secrets configured for each cluster
