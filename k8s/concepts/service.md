# Service

**Topic:** [[k8s/topics/networking]]
**Related:** [[k8s/concepts/ingress]], [[k8s/topics/networking]]

## What it is
A Service provides a stable virtual IP (ClusterIP) and DNS name for a dynamic set of Pods selected by label. Pods come and go; the Service IP stays constant.

## How it works
1. The Service controller watches for Pods matching `spec.selector`
2. Creates an **EndpointSlice** (or Endpoints) object listing their IPs and ports
3. kube-proxy on every node installs iptables/ipvs rules: `ClusterIP:port → one of the Pod IPs`
4. CoreDNS registers `<svc>.<ns>.svc.cluster.local → ClusterIP`

When a Pod is removed (failing readiness, deleted), it's removed from the EndpointSlice and kube-proxy updates the rules. There is a brief window where traffic can still be routed to a terminating Pod — this is why `preStop` sleep + `terminationGracePeriodSeconds` matter.

## Service Types

```yaml
apiVersion: v1
kind: Service
spec:
  selector:
    app: model-server
  ports:
  - port: 80          # service port
    targetPort: 8080  # container port
  type: ClusterIP     # ClusterIP | NodePort | LoadBalancer | ExternalName
```

**ClusterIP** (default): virtual IP reachable only within the cluster.

**NodePort**: allocates a port (30000–32767) on every node. Accessible at `<NodeIP>:<NodePort>`. Not recommended for production (exposes all nodes, no TLS).

**LoadBalancer**: triggers the cloud controller to provision a cloud load balancer. Inherits NodePort behavior plus external LB.

**ExternalName**: returns a CNAME. No proxying. Use to alias an external service: `database.company.com`.

## Headless Service

```yaml
spec:
  clusterIP: None
```

No virtual IP. DNS returns the IPs of all Pods directly. StatefulSets use this so each Pod has a stable DNS name: `<pod-name>.<svc-name>.<ns>.svc.cluster.local`.

## Debugging a Service

```bash
# Step 1: Does the Service have endpoints?
kubectl get endpoints <svc-name>
# Empty → selector doesn't match any Pod labels

# Step 2: Are the backend Pods ready?
kubectl get pods -l <selector> -o wide
# NotReady → readiness probe failing

# Step 3: Can you reach the ClusterIP directly?
kubectl exec -it debug-pod -- curl <clusterIP>:<port>

# Step 4: Does DNS resolve?
kubectl exec -it debug-pod -- nslookup <svc-name>.<namespace>

# Step 5: Are kube-proxy rules present?
iptables -t nat -L KUBE-SERVICES | grep <clusterIP>
```

## Common Interview Angles
- "Service has no endpoints" → selector mismatch (label typo is the most common cause)
- "Traffic goes to pods but they 404" → `targetPort` is wrong
- "Pods are Ready but Service is unreachable" → NetworkPolicy blocking traffic; kube-proxy not running on that node
- "How does a Pod discover other services?" → DNS via CoreDNS; or env vars (`<SVC>_SERVICE_HOST`) — but env vars only work for Services created before the Pod

## Sources
- [[K8S overview]]
