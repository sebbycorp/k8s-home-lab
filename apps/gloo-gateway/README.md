# Gloo Mesh Enterprise Installation via ArgoCD GitOps

This directory contains ArgoCD Application manifests for installing Gloo Mesh Enterprise with proper control plane and data plane separation.

## Architecture

```
Management Cluster (maniak-talos-mgmt)
├── Control Plane Components:
│   ├── Gloo Management Server
│   ├── Gloo Mesh UI
│   ├── Gloo Insights Engine
│   └── Gloo Analyzer
│
Data Plane Clusters (maniak-east, maniak-west)
├── Data Plane Components:
│   ├── Gloo Agent (connects to mgmt cluster)
│   └── Gloo Gateway
```

## GitOps Flow

```
Git Repository (apps/gloo-gateway/)
    ↓
Kustomization (kustomization.yaml includes all apps)
    ↓
Bootstrap Application (bootstrap/apps/gloo-gateway-apps.yaml)
    ↓
ArgoCD syncs and creates Applications
    ↓
Gloo Mesh Enterprise installed:
  - Control plane in mgmt cluster
  - Data plane in east/west clusters
```

## Structure

- **gloo-mesh-mgmt.yaml** - Control plane in management cluster (`maniak-talos-mgmt`)
- **gloo-mesh-east.yaml** - Data plane in east cluster (`maniak-east`)
- **gloo-mesh-west.yaml** - Data plane in west cluster (`maniak-west`)
- **kustomization.yaml** - Kustomize manifest that includes all Applications

## Installation Steps

### 1. Commit Application Manifests to Git

```bash
git add apps/gloo-gateway/
git commit -m "Add Gloo Gateway ArgoCD applications"
git push origin main
```

### 2. Bootstrap the Parent Application

Apply the bootstrap Application that manages these apps:

```bash
kubectl apply -f bootstrap/apps/gloo-gateway-apps.yaml
```

This creates a parent Application that watches the `apps/gloo-gateway/` directory in your Git repo.

### 3. ArgoCD Syncs Automatically

ArgoCD will:
- Discover the Application manifests in Git
- Create Applications for each YAML file
- Sync Gloo Gateway to your clusters automatically

## Verification

Check the parent application:
```bash
argocd app get gloo-gateway-apps --insecure
```

Check individual applications:
```bash
argocd app get gloo-gateway-east --insecure
argocd app get gloo-gateway-west --insecure
```

View in ArgoCD UI:
```bash
kubectl port-forward -n argocd svc/argocd-server 8080:80
# Open http://localhost:8080
```

## Configuration

### Management Cluster (Control Plane)
- **Gloo Management Server**: Enabled (control plane)
- **Gloo Mesh UI**: Enabled (Enterprise feature)
- **Gloo Insights Engine**: Enabled (Enterprise analytics)
- **Gloo Analyzer**: Enabled (Enterprise feature)
- **Gloo Gateway**: Disabled (data plane only)
- **Gloo Agent**: Disabled (data plane only)

### Data Plane Clusters (East & West)
- **Gloo Agent**: Enabled (connects to mgmt cluster)
- **Gloo Gateway**: Enabled (data plane gateway)
- **Gloo Management Server**: Disabled (control plane only)
- **Gloo Mesh UI**: Disabled (control plane only)
- **Gloo Insights Engine**: Disabled (control plane only)
- **Gloo Analyzer**: Disabled (control plane only)

## License Configuration

Gloo Mesh Enterprise requires a license key. You have two options:

### Option 1: Direct License Key (Simple)

Edit the `glooMeshLicenseKey` field in the Application manifests:

```yaml
licensing:
  glooMeshLicenseKey: "your-license-key-here"
```

### Option 2: License Secret (Recommended for Production)

1. Create a secret with your license key:
   ```bash
   kubectl create secret generic license-keys \
     --from-literal=glooMeshLicenseKey='your-license-key-here' \
     -n gloo-system
   ```

2. Update the Application manifests to use the secret:
   ```yaml
   licensing:
     licenseSecretName: license-keys
   ```

**Note**: The secret must exist in the `gloo-system` namespace before the Gloo Platform installation.

## Helm Chart Details

- **Repository**: `https://storage.googleapis.com/gloo-platform/helm-charts`
- **Chart**: `gloo-platform`
- **Version**: `2.7.0` (adjust as needed)
- **Namespace**: `gloo-system` (created automatically)

## Customization

To customize the installation:

1. Edit the `values` section in `apps/gloo-gateway/*.yaml`
2. Commit and push to Git
3. ArgoCD will automatically sync the changes

Example: Add a license key
```yaml
values: |
  licensing:
    glooMeshLicenseKey: "your-license-key"
  glooGateway:
    enabled: true
  # ... rest of config
```

### Enterprise Features

The configuration includes these Gloo Mesh Enterprise features:
- **Gloo Mesh UI**: Web-based management interface
- **Gloo Insights Engine**: Analytics and observability
- **Gloo Analyzer**: Policy and configuration analysis

To disable any feature, set `enabled: false` for that component.

## GitOps Benefits

- **Version Control**: All changes tracked in Git
- **Automated Sync**: Changes automatically applied
- **Self-Healing**: ArgoCD ensures desired state
- **Audit Trail**: Full history in Git commits
- **Multi-Environment**: Easy to replicate across clusters

## Troubleshooting

If applications don't sync:

1. Check Git repo access:
   ```bash
   argocd repo get https://github.com/sebbycorp/k8s-home-lab.git --insecure
   ```

2. Check application status:
   ```bash
   argocd app list --insecure | grep gloo-gateway
   ```

3. Check sync status:
   ```bash
   argocd app get gloo-gateway-east --insecure
   ```

4. Force sync if needed:
   ```bash
   argocd app sync gloo-gateway-east --insecure
   ```

## Notes

- The parent Application (`gloo-gateway-apps`) watches the `apps/gloo-gateway/` directory
- Any YAML files in that directory will be automatically discovered as Applications
- Applications use cluster names (`maniak-east`, `maniak-west`) to target specific clusters
- Automated sync is enabled with prune and self-heal for true GitOps
