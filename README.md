# Fleet Azure Cluster Provisioning Example

This repository demonstrates how to provision Kubernetes clusters on Azure using Rancher's Fleet GitOps engine with a simplified approach using template variables.

## Architecture Overview

This setup uses:
- **Fleet Controller**: GitOps engine for manifest synchronization
- **Template Variables**: Simple variable substitution for environment-specific values
- **Azure Cloud Provider**: Infrastructure provisioning on Microsoft Azure
- **Rancher Provisioning APIs**: `provisioning.cattle.io/v1` for cluster management

## Repository Structure

```
fleet-azure-example/
├── fleet.yaml              # Main Fleet configuration
├── base/                   # Base Kustomize configuration
│   ├── kustomization.yaml # Base kustomization
│   └── cluster.yaml       # Base RKE2 cluster definition
├── overlays/               # Environment-specific overlays
│   └── dev/               # Development environment
│       └── kustomization.yaml # Dev-specific patches
└── README.md              # This file
```

## Configuration

### Environment-Specific Values

The development environment (`overlays/dev/`) is configured with:

- **Cluster**: `fleet-test-dev` in `germanywestcentral`
- **Resource Group**: `foo-machine`
- **Network**: `fleet-test-dev-vnet` / `fleet-test-dev` subnet
- **VM Size**: `Standard_B2as_v2`
- **Image**: SUSE Linux Enterprise Server 15 SP5 BYOS
- **Nodes**: 1 control plane + 1 worker

### Customization

To modify values for the dev environment, edit the patches in `overlays/dev/kustomization.yaml`. Each patch targets specific resources and paths in the base configuration.

## Prerequisites

1. **Management Cluster**: Rancher management cluster ("local") with:
   - `fleet-controller` installed
   - `cluster-api-controller` installed
   - Required CRDs from `provisioning.cattle.io/v1`, `rke-machine-config.cattle.io/v1`, `rke-machine.cattle.io/v1`

2. **Azure Credentials**: Existing cloud credential secret in Rancher
   - Secret name should be referenced in `cloudCredentialSecretName`

## How It Works

This repository uses Kustomize with Fleet to manage environment-specific cluster configurations.

### Kustomize Structure

1. **Base Configuration** (`base/`): Contains the common cluster definition with placeholder values
2. **Environment Overlays** (`overlays/dev/`): Contains environment-specific patches that modify the base configuration

### Configuration Process

1. Fleet monitors the `overlays/dev` directory
2. Kustomize applies the base configuration from `base/`
3. Environment-specific patches from `overlays/dev/kustomization.yaml` are applied
4. The final manifests are deployed to the target cluster

### Adding New Environments

To add a new environment (e.g., `staging`):

1. Create `overlays/staging/kustomization.yaml`
2. Define environment-specific patches
3. Update the main `fleet.yaml` to include the new path
   - Must have permissions for VM, Network, and Storage operations

3. **Fleet Repository**: This repository should be added as a Fleet Git repository

## Environment Configuration

### Development Environment
- **Cluster Name**: `azure-dev-cluster`
- **Node Count**: 2 nodes
- **VM Size**: `Standard_D2s_v3`
- **Kubernetes Version**: Latest stable

### Staging Environment
- **Cluster Name**: `azure-staging-cluster`
- **Node Count**: 3 nodes
- **VM Size**: `Standard_D4s_v3`
- **Kubernetes Version**: Latest stable

## File Descriptions

### Base Configuration Files

- **`base/cluster.yaml`**: Defines the main Kubernetes cluster resource using Rancher's `provisioning.cattle.io/v1` API. Includes RKE2 configuration, machine pools, and cluster-wide settings.
- **`base/machine-config.yaml`**: Azure machine configuration defining VM specifications, networking, storage, and custom initialization scripts.
- **`base/nodepool.yaml`**: Additional node pool definitions for specialized workloads (high-memory nodes, additional workers).
- **`base/kustomization.yaml`**: Base kustomization configuration that defines common resources, labels, and annotations.

### Environment-Specific Files

#### Development Environment (`environments/dev/`)
- **`fleet.yaml`**: Fleet bundle configuration for dev environment with relaxed settings and cost optimization.
- **`cluster-values.yaml`**: Development-specific values including smaller VM sizes, reduced node counts, and development features.
- **`kustomization.yaml`**: Kustomization overlay that applies dev-specific patches and transformations.
- **`cluster-patch.yaml`**: Strategic merge patches for cluster configuration (single control plane, relaxed security).
- **`machine-config-patch.yaml`**: Azure machine configuration patches for development (smaller VMs, standard storage).

#### Staging Environment (`environments/staging/`)
- **`fleet.yaml`**: Fleet bundle configuration for staging with production-like settings and enhanced monitoring.
- **`cluster-values.yaml`**: Staging-specific values mirroring production settings for proper testing.
- **`kustomization.yaml`**: Kustomization overlay with staging-specific patches and compliance requirements.
- **`cluster-patch.yaml`**: Strategic merge patches for production-like cluster configuration (HA control plane, enhanced security).
- **`machine-config-patch.yaml`**: Azure machine configuration patches for staging (larger VMs, premium storage, security hardening).

### Utility Files

- **`scripts/deploy.sh`**: Comprehensive deployment script with validation, deployment, status checking, and destruction capabilities.
- **`fleet.yaml`**: Root Fleet configuration for repository-level settings.
- **`.gitignore`**: Git ignore patterns for temporary files, secrets, and build artifacts.

## Key Features Implemented

### GitOps Workflow
- **Fleet Integration**: Complete Fleet GitOps configuration with environment-specific bundles
- **Kustomize Overlays**: Base configuration with environment-specific patches
- **Automated Sync**: Polling-based synchronization with configurable intervals

### Environment Isolation
- **Development**: Cost-optimized, single control plane, relaxed security for rapid development
- **Staging**: Production-like HA setup with enhanced monitoring and security for testing
- **Resource Separation**: Different Azure regions, resource groups, and network configurations

### Azure Integration
- **Cloud Credentials**: References existing Azure credential secrets
- **VM Configurations**: Environment-specific VM sizes and storage configurations
- **Network Security**: Security groups, VNets, and subnets per environment
- **Managed Identity**: Azure managed identity integration for secure access

### Security & Compliance
- **RBAC**: Role-based access control configurations
- **Network Policies**: Environment-specific network isolation
- **Audit Logging**: Comprehensive audit logging for compliance
- **Secrets Encryption**: Kubernetes secrets encryption at rest

### Monitoring & Operations
- **Health Checks**: Automated cluster health monitoring
- **Logging**: Centralized logging with log rotation
- **Metrics**: Node exporter and system metrics collection
- **Alerting**: Environment-specific alerting configurations

## Deployment Instructions

### 1. Add Repository to Fleet

```bash
# Add this repository to Fleet
kubectl apply -f - <<EOF
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: azure-cluster-provisioning
  namespace: fleet-local
spec:
  repo: https://github.com/your-org/fleet-azure-example
  branch: main
  paths:
  - environments/dev
  - environments/staging
EOF
```

### 2. Monitor Deployment

```bash
# Check Fleet bundles
kubectl get bundles -n fleet-local

# Check cluster provisioning status
kubectl get clusters.provisioning.cattle.io -A

# Check machine status
kubectl get machines.rke-machine.cattle.io -A
```

### 3. Access Deployed Clusters

Once provisioned, clusters will be available in the Rancher UI or via kubectl:

```bash
# Get cluster kubeconfig
kubectl get secret <cluster-name>-kubeconfig -n fleet-default -o jsonpath='{.data.value}' | base64 -d > cluster-kubeconfig.yaml
```

## Customization

### Adding New Environments

1. Create new directory under `environments/`
2. Copy and modify `fleet.yaml` and `cluster-values.yaml`
3. Adjust values for the new environment
4. Commit and push changes

### Modifying Cluster Configuration

- **Node Count**: Edit `nodeQuantity` in environment-specific `cluster-values.yaml`
- **VM Size**: Edit `size` in environment-specific `cluster-values.yaml`
- **Kubernetes Version**: Edit `kubernetesVersion` in base `cluster.yaml`
- **Azure Region**: Edit `region` in environment-specific `cluster-values.yaml`

## Troubleshooting

### Common Issues

1. **Credential Issues**: Verify Azure credential secret exists and has proper permissions
2. **Quota Limits**: Check Azure subscription quotas for VM cores
3. **Network Issues**: Ensure proper VNet and subnet configuration
4. **Fleet Sync Issues**: Check Fleet controller logs and bundle status

### Debugging Commands

```bash
# Check Fleet controller logs
kubectl logs -n cattle-fleet-system -l app=fleet-controller

# Check cluster provisioning logs
kubectl logs -n cattle-provisioning-system -l app=provisioning-controller

# Describe cluster for detailed status
kubectl describe cluster.provisioning.cattle.io <cluster-name> -n fleet-default
```

## Security Considerations

- Azure credentials are stored as Kubernetes secrets
- RBAC is configured for least-privilege access
- Network security groups restrict access to necessary ports only
- Regular rotation of cloud credentials is recommended

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make changes and test in development environment
4. Submit a pull request with detailed description

## License

This project is licensed under the MIT License - see the LICENSE file for details.