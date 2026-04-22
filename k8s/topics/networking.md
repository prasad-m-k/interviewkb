# Kubernetes Networking

**Topic:** [[k8s/topics/networking]]
**Related:** [[k8s/concepts/service]], [[k8s/concepts/ingress]]

## The Kubernetes Network Model

Every Pod gets a unique IP address. Pods can communicate with any other Pod directly by IP — no NAT. This flat network is implemented by the CNI plugin (Calico, Flannel, Cilium, AWS VPC CNI, etc.).

## Service Types

A Service provides a stable virtual IP (ClusterIP) and DNS name for a set of Pods, selected by label.

| Type | Reachable From | How |
|---|---|---|
| **ClusterIP** (default) | Inside cluster only | Virtual IP; kube-proxy routes to pod IPs |
| **NodePort** | Outside cluster via `<NodeIP>:<NodePort>` | Allocates port 30000–32767 on every node |
| **LoadBalancer** | Outside cluster via cloud LB | Cloud controller provisions an external load balancer |
| **ExternalName** | Inside cluster | Returns a CNAME DNS alias to an external hostname |
| **Headless** (`clusterIP: None`) | Inside cluster | No virtual IP; DNS returns Pod IPs directly |

**Headless Services** are used by StatefulSets — each Pod gets a stable DNS name `<pod>.<svc>.<ns>.svc.cluster.local`.

**See:** [[k8s/concepts/service]]

## CoreDNS and Service Discovery

CoreDNS runs as a Deployment in `kube-system`. Every Pod's `/etc/resolv.conf` points to the CoreDNS ClusterIP.

DNS name format: `<service-name>.<namespace>.svc.cluster.local`

Short names work within the same namespace (`my-svc`) due to the `ndots` search path.

**Debug DNS:** `kubectl exec -it <pod> -- nslookup my-svc.my-namespace`

## Ingress

An Ingress is a Layer-7 (HTTP/HTTPS) routing rule. An **Ingress Controller** (nginx, Traefik, AWS ALB, GCP GCLB) watches Ingress objects and configures the actual load balancer/proxy.

**See:** [[k8s/concepts/ingress]]

## NetworkPolicy

A NetworkPolicy applies firewall rules to Pods using label selectors. By default, all traffic is allowed. A NetworkPolicy restricts to only what's explicitly allowed.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
```

Key points:
- **Requires a CNI plugin that supports NetworkPolicy** (Calico, Cilium — not Flannel by default)
- An empty `podSelector` (`{}`) selects ALL pods in the namespace
- Ingress/Egress policies are additive across multiple NetworkPolicy objects
- A Pod with any NetworkPolicy becomes restricted; a Pod with no NetworkPolicy is unrestricted

## kube-proxy and Packet Flow

When a Pod sends a request to a Service ClusterIP:
1. kube-proxy (iptables mode) has installed `DNAT` rules: ClusterIP:Port → one of the backend Pod IPs (randomly selected per connection)
2. The packet is rewritten and sent directly to the Pod
3. Return traffic goes directly back (no reverse DNAT needed — conntrack handles it)

**ipvs mode**: more scalable than iptables for large clusters (thousands of Services); uses kernel-level load balancing.

## Common Networking Debug Flow

```
Can Pod A reach Service B?
│
├── DNS resolves? → kubectl exec podA -- nslookup serviceB
├── ClusterIP reachable? → kubectl exec podA -- curl <clusterIP>:<port>
├── Endpoints populated? → kubectl get endpoints serviceB
│   └── Empty? → label selector doesn't match pod labels
├── kube-proxy rules present? → iptables -t nat -L | grep <clusterIP>
└── NetworkPolicy blocking? → kubectl describe netpol -n <namespace>
```

## Sources
- [[K8S overview]]
