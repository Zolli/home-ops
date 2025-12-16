# ArgoCD Bootstrap

This directory contains initial ArgoCD installation resources.

## Deployment

ArgoCD is deployed via Ansible:

```bash
cd ansible
ansible-playbook -i inventory/hosts.yml playbooks/deploy_argocd.yml
```

Or as part of the full cluster deployment:

```bash
cd ansible
ansible-playbook -i inventory/hosts.yml playbooks/deploy.yml
```

## Access

After deployment, ArgoCD is accessible at:

- **URL**: https://172.20.10.20
- **Username**: admin
- **Password**: Run the following command:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Architecture

ArgoCD is deployed with:
- **Helm chart**: argo/argo-cd v7.7.11
- **Exposure**: Cilium LoadBalancer with BGP advertisement (172.20.10.20)
- **Pattern**: App of Apps (root Application watches `argocd-apps/` directory)
- **HA**: 2 replicas for server and repo-server

## App of Apps Pattern

The root Application is created by Ansible and syncs from:
- **Repository**: https://github.com/Zolli/home-ops.git
- **Path**: argocd-apps/
- **Mode**: Automatic sync with self-heal and prune enabled

Any Application manifests placed in `argocd-apps/` will be automatically deployed by ArgoCD.
