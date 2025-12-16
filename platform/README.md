# Home-Ops GitOps Repository

GitOps repository for managing K3s homelab cluster applications via ArgoCD.

## Repository Structure

```
home-ops/
├── README.md                           # This file
├── bootstrap/                          # ArgoCD bootstrap resources
│   ├── argocd-namespace.yaml
│   └── README.md
├── platform/                           # Platform infrastructure
│   └── argocd/                        # ArgoCD self-management
│       ├── kustomization.yaml
│       ├── argocd-server-service.yaml # LoadBalancer with BGP
│       └── README.md
├── argocd-apps/                       # ArgoCD Application CRDs
│   ├── platform/                      # Platform app definitions
│   │   └── argocd-self.yaml          # ArgoCD managing itself
│   └── apps/                          # User app definitions
│       └── demo-app.yaml             # Demo application
└── apps/                              # Application manifests
    └── demo-app/                      # Example NGINX application
        ├── kustomization.yaml
        ├── namespace.yaml
        ├── deployment.yaml
        ├── service.yaml              # With BGP labels
        └── README.md
```

## Architecture

### Infrastructure Layers

**Layer 0-1: Infrastructure (Ansible)**
- System preparation
- K3s cluster (v1.33.6+k3s1)
- Cilium CNI (v1.18.4) with BGP
- ArgoCD bootstrap

**Layer 2-3: Platform & Apps (ArgoCD/GitOps)**
- Platform components (ArgoCD self-management)
- User applications (demo-app, etc.)

### Network Architecture

```
UDM-Pro Router (10.16.200.1, AS 65100)
   ↓ BGP Peering
K3s Cluster (10.16.200.11-13, AS 65200)
   ↓ Cilium BGP Control Plane
LoadBalancer IP Pool: 172.20.10.0/24
   ├── ArgoCD: 172.20.10.20
   └── Demo App: Auto-assigned
```

## Deployment

### Prerequisites

1. K3s cluster deployed via Ansible
2. Cilium CNI with BGP configured
3. ArgoCD installed via Ansible

### Initial Setup

1. **Fork/Clone this repository**:
   ```bash
   git clone https://github.com/Zolli/home-ops.git
   cd home-ops
   ```

2. **Update repository URL** in Ansible configuration:
   ```yaml
   # ansible/inventory/group_vars/all/vars.yml
   argocd_git_repo: "https://github.com/YOUR-USERNAME/home-ops.git"
   ```

3. **Deploy ArgoCD via Ansible**:
   ```bash
   cd ../lab-cluster/ansible
   ansible-playbook -i inventory/hosts.yml playbooks/deploy_argocd.yml
   ```

4. **Access ArgoCD UI**:
   ```bash
   # Get admin password
   kubectl -n argocd get secret argocd-initial-admin-secret \
     -o jsonpath="{.data.password}" | base64 -d

   # Access UI at https://172.20.10.20
   ```

### App of Apps Pattern

ArgoCD uses the **App of Apps** pattern:
- Root Application (created by Ansible) watches `argocd-apps/` directory
- Any Application manifests in `argocd-apps/` are automatically deployed
- Supports recursive directory structure

## Adding New Applications

### Method 1: Via Application CRD

1. Create application manifests in `apps/your-app/`
2. Create Application CRD in `argocd-apps/apps/your-app.yaml`
3. Commit and push to Git
4. ArgoCD automatically syncs and deploys

### Method 2: Via ArgoCD UI

1. Access ArgoCD UI (https://172.20.10.20)
2. Click "New App"
3. Fill in application details
4. Click "Create"

## BGP LoadBalancer Usage

For services to be advertised via BGP, add these labels:

```yaml
metadata:
  labels:
    bgp.cilium.io/ip-pool: default
    bgp.cilium.io/advertise-service: default
```

**Optional**: Request specific IP:
```yaml
metadata:
  annotations:
    io.cilium/lb-ipam-ips: "172.20.10.X"
```

## Verification

### Check ArgoCD Applications

```bash
# List all applications
kubectl get applications -n argocd

# Check specific application
kubectl get application -n argocd demo-app -o yaml

# ArgoCD CLI
argocd app list
argocd app get demo-app
```

### Check BGP Advertisement

```bash
# Check BGP peers
kubectl exec -n kube-system ds/cilium -- cilium bgp peers

# Check advertised routes
kubectl exec -n kube-system ds/cilium -- cilium bgp routes advertised ipv4 unicast

# From router (UDM-Pro)
ssh router
vtysh -c "show ip bgp"
vtysh -c "show ip bgp neighbors 10.16.200.11 routes"
```

### Check LoadBalancer Services

```bash
# List all LoadBalancer services
kubectl get svc --all-namespaces | grep LoadBalancer

# Check specific service
kubectl get svc -n demo-app demo-app
```

## Common Operations

### Sync All Applications

```bash
argocd app sync --all
```

### Refresh Application

```bash
argocd app refresh demo-app
```

### View Sync Status

```bash
argocd app get demo-app --refresh
```

### Delete Application

```bash
argocd app delete demo-app
# Or remove the Application manifest and commit
```

## Troubleshooting

### Application Not Syncing

```bash
# Check application events
kubectl describe application -n argocd demo-app

# Check ArgoCD logs
kubectl logs -n argocd deployment/argocd-application-controller

# Force sync
argocd app sync demo-app --force
```

### LoadBalancer IP Not Assigned

```bash
# Check Cilium IP pool
kubectl get ciliumloadbalancerippool -o yaml

# Check service events
kubectl describe svc -n demo-app demo-app

# Check Cilium logs
kubectl logs -n kube-system ds/cilium | grep loadbalancer
```

### BGP Not Advertising

```bash
# Check BGP peering status
kubectl exec -n kube-system ds/cilium -- cilium bgp peers

# Check service has correct labels
kubectl get svc -n demo-app demo-app -o yaml | grep bgp.cilium.io

# Check BGP configuration
kubectl get ciliumbgpclusterconfig -o yaml
```

## Best Practices

1. **GitOps Workflow**: All changes go through Git (no kubectl apply)
2. **Branch Protection**: Require PR reviews for main branch
3. **Sync Policy**: Use automatic sync with selfHeal for platform, manual for critical apps
4. **Resource Limits**: Always define requests and limits
5. **Health Checks**: Configure liveness and readiness probes
6. **BGP Labels**: Remember to add BGP labels to LoadBalancer services
7. **Namespaces**: Use dedicated namespaces for each application
8. **Secrets**: Use Sealed Secrets or External Secrets Operator (future)

## Future Enhancements

- [ ] cert-manager for Let's Encrypt certificates
- [ ] Ingress controller (Traefik/NGINX)
- [ ] Sealed Secrets for GitOps-friendly secret management
- [ ] External Secrets Operator for Vault integration
- [ ] Prometheus + Grafana monitoring stack
- [ ] Loki for log aggregation
- [ ] ApplicationSets for dynamic app generation
- [ ] ArgoCD Projects for RBAC
- [ ] ArgoCD Notifications (Slack/Email)

## Support

For issues or questions:
1. Check ArgoCD documentation: https://argo-cd.readthedocs.io/
2. Review Cilium BGP docs: https://docs.cilium.io/en/stable/network/bgp-control-plane/
3. Check application logs and events

## License

MIT
