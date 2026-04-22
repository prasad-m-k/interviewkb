# Scenario: Service Not Reachable

**Category:** Debugging / Networking
**Difficulty:** Hard
**Seen at:** Google, Meta, Amazon

## The Scenario
Pod A cannot reach Service B. Requests time out or get connection refused. Both Pods appear `Running`.

## Systematic Debug Ladder

### Level 1: Check the Service has endpoints
```bash
kubectl get endpoints <service-name> -n <namespace>
# If ENDPOINTS column is <none> or empty → no Pods matched the selector
```

**Empty endpoints is the #1 cause of "Service unreachable."**

```bash
# Check the selector on the Service
kubectl get svc <service-name> -o yaml | grep -A 5 selector

# Check labels on the target Pods
kubectl get pods -l <selector-labels> -n <namespace>
# If no output → labels don't match; fix the selector or Pod labels
```

### Level 2: Check Pod readiness
```bash
kubectl get pods -n <namespace>
# Are the backend Pods in Running state AND Ready (1/1)?
# Not Ready → readiness probe failing → removed from endpoints
kubectl describe pod <backend-pod> | grep -A 10 Readiness
```

### Level 3: Test connectivity directly (bypass Service)
```bash
# Get the backend Pod IP
kubectl get pods -o wide

# From Pod A, hit the backend Pod IP directly
kubectl exec -it <pod-a> -- curl <backend-pod-ip>:<container-port>
# Works → Service/kube-proxy issue
# Fails → network/CNI issue or app not listening
```

### Level 4: Test the Service ClusterIP
```bash
# From Pod A, hit the ClusterIP
kubectl exec -it <pod-a> -- curl <clusterIP>:<service-port>

# Check the Service port vs targetPort
kubectl get svc <service-name> -o yaml
# port: 80 (service port) → targetPort: 8080 (container port)
# Sending to wrong port? → fix targetPort
```

### Level 5: DNS resolution
```bash
kubectl exec -it <pod-a> -- nslookup <service-name>
kubectl exec -it <pod-a> -- nslookup <service-name>.<namespace>.svc.cluster.local

# DNS failing → CoreDNS issue
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### Level 6: NetworkPolicy
```bash
# Any NetworkPolicy in the namespace?
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy -n <namespace>
# Is Pod A's label in an allowed ingress source for Service B's namespace?
```

### Level 7: kube-proxy
```bash
# Is kube-proxy healthy on the node?
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system <kube-proxy-pod>

# Are iptables rules present?
# (on the node)
iptables -t nat -L KUBE-SERVICES | grep <clusterIP>
```

## Decision Tree Summary

```
Service unreachable
├── No endpoints → Selector mismatch (fix labels)
├── Pods not Ready → Readiness probe failing (fix probe / app)
├── Pod IP works, ClusterIP doesn't → kube-proxy rules missing
├── ClusterIP works, DNS doesn't → CoreDNS issue
├── DNS + ClusterIP works → NetworkPolicy blocking
└── Nothing works → CNI plugin issue
```

## Sources
- [[K8S overview]]
- [[k8s/topics/networking]]
