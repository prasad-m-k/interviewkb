# Ingress

**Topic:** [[k8s/topics/networking]]
**Related:** [[k8s/concepts/service]], [[k8s/topics/security]]

## What it is
An Ingress is an API object that defines Layer-7 (HTTP/HTTPS) routing rules. An **Ingress Controller** (a Pod running a reverse proxy) watches Ingress objects and configures routing accordingly.

Key distinction: the Ingress *object* is just config. Without an Ingress Controller, nothing happens.

## Ingress Controller Options

| Controller | Common In | Notes |
|---|---|---|
| **nginx-ingress** | Self-managed K8s | Most common; highly configurable |
| **AWS ALB Ingress Controller** | EKS | Provisions ALB per Ingress |
| **GKE Ingress** | GKE | Provisions GCLB; integrates with Cloud Armor |
| **Traefik** | Edge/homelab | Dynamic config; good for service mesh |
| **Istio Gateway** | Service mesh | Combines with VirtualService for fine-grained routing |

## YAML Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: model-serving-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"   # long for inference
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.mycompany.com
    secretName: tls-cert-secret
  rules:
  - host: api.mycompany.com
    http:
      paths:
      - path: /v1/predict
        pathType: Prefix
        backend:
          service:
            name: model-server-svc
            port:
              number: 80
      - path: /v2/models
        pathType: Prefix
        backend:
          service:
            name: model-server-v2-svc
            port:
              number: 80
```

## TLS Termination

TLS is terminated at the Ingress Controller. The backend Services use plain HTTP within the cluster. The TLS certificate is stored as a `kubernetes.io/tls` Secret.

For automatic cert management: **cert-manager** watches Ingress objects and automatically provisions Let's Encrypt certificates.

## Common Interview Angles
- "Ingress returns 404 for valid paths" → path regex/rewrite misconfiguration; check `annotations` for rewrite rules
- "Ingress returns 502/503" → backend Service has no ready endpoints; check Pod readiness
- "How do you do canary deployments with Ingress?" → nginx-ingress annotations `canary: "true"` + `canary-weight: "10"` to route 10% to a canary Service
- "Ingress vs Service LoadBalancer" → LoadBalancer provisions one LB per Service (expensive); Ingress shares one LB across many Services with L7 routing (cost-efficient)

## Sources
- [[K8S overview]]
