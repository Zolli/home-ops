# Demo App

Simple NGINX-based demo application to test ArgoCD deployment and Cilium BGP LoadBalancer.

## Features

- **Image**: nginx:1.27-alpine
- **Replicas**: 2 for HA
- **Service Type**: LoadBalancer with Cilium BGP advertisement
- **Health Checks**: Liveness and readiness probes enabled
- **Resource Limits**: Requests and limits configured

## BGP LoadBalancer

This service uses Cilium's BGP control plane to advertise the LoadBalancer IP to your UDM-Pro router.

**Required labels:**
```yaml
labels:
  bgp.cilium.io/ip-pool: default
  bgp.cilium.io/advertise-service: default
```

## Access

After deployment, get the LoadBalancer IP:

```bash
kubectl get svc -n demo-app demo-app
```

Access the application at `http://<LOADBALANCER-IP>`

## Verification

Check BGP advertisement:

```bash
# Check BGP peers
kubectl exec -n kube-system ds/cilium -- cilium bgp peers

# Check advertised routes
kubectl exec -n kube-system ds/cilium -- cilium bgp routes advertised ipv4 unicast
```

## Customization

To request a specific IP from the pool, uncomment and modify the annotation in `service.yaml`:

```yaml
annotations:
  io.cilium/lb-ipam-ips: "172.20.10.21"
```
