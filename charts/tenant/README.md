# Tenant Chart

![Version: 0.1.12](https://img.shields.io/badge/Version-0.1.12-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 0.29.0](https://img.shields.io/badge/AppVersion-0.29.0-informational?style=flat-square)

A Helm chart for deploying isolated virtual Kubernetes clusters (vclusters) in a multi-tenant environment using the Kubespaces platform.

## Overview

This chart deploys a tenant-specific virtual cluster using vcluster technology, providing complete Kubernetes API isolation while sharing the underlying infrastructure. Each tenant gets their own dedicated namespace with resource quotas, network policies, and security boundaries.

## Features

- **Complete Kubernetes API Isolation**: Each tenant operates with their own Kubernetes API server
- **Resource Quotas & Limits**: Configurable CPU, memory, storage, and object count limits
- **Network Isolation**: Built-in network policies for secure multi-tenancy
- **Gateway Integration**: Automatic TLS routing and external DNS configuration
- **Security Policies**: Pod Security Standards (baseline) enforcement
- **Istio Ambient Mesh Support**: Optional service mesh integration
- **Spot Instance Optimization**: Configured for Azure spot instances with appropriate tolerations

## Prerequisites

- Kubernetes cluster with Gateway API support
- Istio service mesh (optional, for ambient mesh features)
- External DNS controller (for automatic DNS record creation)
- Cert-manager (for TLS certificate management)

## Installation

### Quick Start

```bash
# Set your tenant configuration
export TENANT_NAME=dev
export ORG_NAME=contoso
export LOCATION_SHORT=ne
export CLOUD=azure
export DOMAIN=kubespaces.cloud

# Install the tenant chart
helm upgrade -i -n $TENANT_NAME-$ORG_NAME --create-namespace \
  $TENANT_NAME-$ORG_NAME oci://ghcr.io/kubespaces-io/kubespaces-public/tenant \
  --set tenant.name=$TENANT_NAME \
  --set tenant.org=$ORG_NAME \
  --set tenant.location_short=$LOCATION_SHORT \
  --set tenant.cloud=$CLOUD \
  --set tenant.domain=$DOMAIN \
  --set "vcluster.exportKubeConfig.server=https://api.$TENANT_NAME.$ORG_NAME.$LOCATION_SHORT.$CLOUD.$DOMAIN" \
  --set "vcluster.controlPlane.proxy.extraSANs[0]=api.$TENANT_NAME.$ORG_NAME.$LOCATION_SHORT.$CLOUD.$DOMAIN"
```

### Custom Values File

Create a `tenant-values.yaml` file:

```yaml
tenant:
  name: dev
  org: contoso
  location_short: ne
  cloud: azure
  domain: kubespaces.cloud

vcluster:
  policies:
    resourceQuota:
      quota:
        requests.cpu: 20
        requests.memory: 40Gi
        limits.cpu: 40
        limits.memory: 80Gi
        
  controlPlane:
    proxy:
      extraSANs:
        - api.dev.contoso.ne.azure.kubespaces.cloud
        
  exportKubeConfig:
    server: https://api.dev.contoso.ne.azure.kubespaces.cloud
```

Then install:

```bash
helm upgrade -i -n dev-contoso --create-namespace \
  dev-contoso oci://ghcr.io/kubespaces-io/kubespaces-public/tenant \
  -f tenant-values.yaml
```

## Accessing Your Tenant

After installation, retrieve the tenant kubeconfig:

```bash
# Get the kubeconfig
kubectl get secret vc-$TENANT_NAME-$ORG_NAME -n $TENANT_NAME-$ORG_NAME \
  -o jsonpath='{.data.config}' | base64 -d > /tmp/$TENANT_NAME-$ORG_NAME-kubeconfig

# Use the tenant cluster
export KUBECONFIG=/tmp/$TENANT_NAME-$ORG_NAME-kubeconfig
kubectl get nodes
kubectl get pods -A
```

## Configuration

### Tenant Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `tenant.name` | Tenant name (e.g., dev, prod, test) | `dev` |
| `tenant.org` | Organization name | `contoso` |
| `tenant.location_short` | Short location code (e.g., ne, we) | `rks` |
| `tenant.cloud` | Cloud provider (azure, aws, gcp) | `azure` |
| `tenant.domain` | Base domain for the tenant | `kubespaces.cloud` |
| `tenant.api_port` | API server port | `443` |
| `tenant.ambient.enabled` | Enable Istio ambient mesh | `true` |

### Resource Quotas

The chart includes sensible defaults for resource quotas that can be customized:

```yaml
vcluster:
  policies:
    resourceQuota:
      quota:
        requests.cpu: 10          # Total CPU requests
        requests.memory: 20Gi     # Total memory requests
        requests.storage: 100Gi   # Total storage requests
        limits.cpu: 20           # Total CPU limits  
        limits.memory: 40Gi      # Total memory limits
        count/pods: 200          # Maximum number of pods
        count/services: 200      # Maximum number of services
```

### Security Policies

Pod Security Standards are enforced at the baseline level:

```yaml
vcluster:
  policies:
    podSecurityStandard: baseline
    networkPolicy:
      enabled: true
```

### Node Scheduling

The chart is optimized for Azure spot instances:

```yaml
vcluster:
  controlPlane:
    statefulSet:
      scheduling:
        nodeSelector:
          kubespaces.io/purpose: tenants
        tolerations:
          - key: "kubernetes.azure.com/scalesetpriority"
            operator: "Equal"
            value: "spot"
            effect: "NoSchedule"
```

## Networking

The chart automatically configures:

- **TLS Routes**: For secure API server access
- **Reference Grants**: For cross-namespace service access
- **External DNS**: Automatic DNS record creation for the tenant API endpoint

The resulting API endpoint will be: `https://api.{tenant}.{org}.{location}.{cloud}.{domain}`

## Monitoring and Observability

Telemetry is disabled by default for privacy:

```yaml
vcluster:
  telemetry:
    enabled: false
```

## Gateway API Integration

The chart creates Gateway API resources for routing:

- **TLSRoute**: Routes traffic to the tenant's API server
- **ReferenceGrants**: Allows cross-namespace access from the gateway

## Troubleshooting

### Common Issues

1. **Pod stuck in Pending**: Check node selectors and tolerations
2. **API server not accessible**: Verify DNS and TLS route configuration
3. **Resource quota exceeded**: Adjust quota values in the configuration

### Logs

Check the vcluster logs:

```bash
kubectl logs -n $TENANT_NAME-$ORG_NAME -l app=vcluster
```

## Dependencies

| Repository | Name | Version |
|------------|------|---------|
| <https://charts.loft.sh/> | vcluster | 0.29.0 |

## Values Reference

### Core Configuration

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `tenant.name` | string | `"dev"` | Tenant name (cannot be 'default') |
| `tenant.org` | string | `"contoso"` | Organization name |
| `tenant.location_short` | string | `"rks"` | Short location identifier |
| `tenant.cloud` | string | `"azure"` | Cloud provider (azure, aws, gcp) |
| `tenant.domain` | string | `"kubespaces.cloud"` | Base domain name |
| `tenant.api_port` | int | `443` | Kubernetes API server port |
| `tenant.ambient.enabled` | bool | `true` | Enable Istio ambient mesh support |

### System Configuration

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `system.namespace` | string | `"istio-system"` | Gateway deployment namespace |
| `system.gateway` | string | `"api"` | Gateway name |
| `system.target` | string | `null` | External IP for on-premise installations |

### vCluster Resource Policies

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `vcluster.policies.podSecurityStandard` | string | `"baseline"` | Pod Security Standard level |
| `vcluster.policies.resourceQuota.enabled` | bool | `true` | Enable resource quotas |
| `vcluster.policies.limitRange.enabled` | bool | `true` | Enable limit ranges |
| `vcluster.policies.networkPolicy.enabled` | bool | `true` | Enable network policies |

### Default Resource Quotas

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `vcluster.policies.resourceQuota.quota.requests.cpu` | int | `10` | Total CPU requests |
| `vcluster.policies.resourceQuota.quota.requests.memory` | string | `"20Gi"` | Total memory requests |
| `vcluster.policies.resourceQuota.quota.requests.storage` | string | `"100Gi"` | Total storage requests |
| `vcluster.policies.resourceQuota.quota.limits.cpu` | int | `20` | Total CPU limits |
| `vcluster.policies.resourceQuota.quota.limits.memory` | string | `"40Gi"` | Total memory limits |
| `vcluster.policies.resourceQuota.quota.count/pods` | int | `200` | Maximum pod count |
| `vcluster.policies.resourceQuota.quota.count/services` | int | `200` | Maximum service count |

### Default Resource Limits

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `vcluster.policies.limitRange.default.cpu` | string | `"4"` | Default CPU limit |
| `vcluster.policies.limitRange.default.memory` | string | `"2Gi"` | Default memory limit |
| `vcluster.policies.limitRange.default.ephemeral-storage` | string | `"8Gi"` | Default ephemeral storage limit |
| `vcluster.policies.limitRange.defaultRequest.cpu` | string | `"100m"` | Default CPU request |
| `vcluster.policies.limitRange.defaultRequest.memory` | string | `"128Mi"` | Default memory request |
| `vcluster.policies.limitRange.defaultRequest.ephemeral-storage` | string | `"3Gi"` | Default ephemeral storage request |

### Control Plane Configuration

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `vcluster.controlPlane.distro.k8s.version` | string | `"v1.31.1"` | Kubernetes version |
| `vcluster.controlPlane.proxy.extraSANs` | list | `[]` | Additional SANs for API server certificate |
| `vcluster.telemetry.enabled` | bool | `false` | Enable telemetry collection |

## License

This chart is part of the Kubespaces project and is licensed under the Apache License 2.0.
