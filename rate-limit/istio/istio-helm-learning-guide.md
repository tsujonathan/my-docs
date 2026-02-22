# Istio and Helm Learning Guide
**Context:** External gRPC Access Implementation for RateLimit Service in TrueDev

---

## Table of Contents
1. [Istio Service Mesh Fundamentals](#istio-service-mesh-fundamentals)
2. [Helm Chart Architecture](#helm-chart-architecture)
3. [Traffic Flow Architecture](#traffic-flow-architecture)
4. [VirtualService Deep Dive](#virtualservice-deep-dive)
5. [Gateway Types](#gateway-types)
6. [Authorization Policies](#authorization-policies)
7. [Helm Values: publicApi vs internalApi](#helm-values-publicapi-vs-internalapi)
8. [Without Istio: Alternatives](#without-istio-alternatives)
9. [Key Takeaways](#key-takeaways)

---

## Istio Service Mesh Fundamentals

### What is Istio?
Istio is a **service mesh** that provides:
- **Traffic Management**: Advanced routing (canary, A/B testing, traffic splitting)
- **Security**: mTLS encryption between services, authorization policies
- **Observability**: Distributed tracing, metrics, logging via sidecar proxies
- **Resilience**: Circuit breaking, retries, timeouts

### Core Components in Your Deployment
1. **Istio Proxy (Envoy Sidecar)**: Injected into each pod as `istio-proxy` container
2. **Istio Ingress Gateway**: Entry point for external traffic (LoadBalancer service)
3. **Istio Control Plane**: Manages configuration and policy distribution

### Key Istio Custom Resources
- **VirtualService**: L7 routing rules (path/host matching, traffic splitting)
- **Gateway**: Configures load balancer for inbound/outbound traffic
- **DestinationRule**: Traffic policies after routing (load balancing, circuit breaking)
- **AuthorizationPolicy**: RBAC for service-to-service and ingress traffic

---

## Helm Chart Architecture

### What is Helm?
**Package manager for Kubernetes**. Combines templates with values to generate Kubernetes manifests.

### Key Concepts

#### 1. Templates
Template files with placeholders:
```yaml
host: "{{ .Release.Name }}.{{ .Release.Namespace }}.svc.cluster.local"
```

#### 2. Values
Configuration data (app-values.yaml):
```yaml
virtualService:
  enabled: true
```

#### 3. Template Variables
- `{{ .Release.Name }}`: Release name (e.g., "ratelimit")
- `{{ .Release.Namespace }}`: Kubernetes namespace (e.g., "ratelimit")
- `{{ .Values.service.port }}`: Value from app-values.yaml

#### 4. Release
An instance of a Helm chart deployed in the cluster:
```bash
helm list -n ratelimit
# NAME       NAMESPACE  REVISION  STATUS
# ratelimit  ratelimit  1         deployed
```

---

## Traffic Flow Architecture

### External gRPC Request Flow
```
External Client (grpcurl)
    ↓
DNS: services.true.dev.docusign.net → 4.227.44.227 (Azure Load Balancer)
    ↓
[Azure Load Balancer]
    ↓
Istio Ingress Gateway Pod (istio-ingressgateway)
    ↓ (reads Gateway resource)
[Gateway: istio-resources/services-gateway]
    ↓ (matches host: *.services.true.dev.docusign.net)
[VirtualService: ratelimit-grpc-external]
    ↓ (matches URI prefix: /docusign.ratelimit.v1.)
[Kubernetes Service: ratelimit:8081]
    ↓ (load balances to pods with label app=ratelimit)
[Pod: ratelimit-xxx]
    ↓ (traffic hits istio-proxy sidecar first)
[AuthorizationPolicy Check]
    ↓ (verifies principal: istio-ingressgateway-service-account)
[Application Container: ratelimit]
    ↓
gRPC Handler: ShouldRateLimit()
    ↓
Response: {"overallCode":"CODE_OK"}
```

### Internal Service-to-Service Flow
```
Service A Pod
    ↓
[Service A's istio-proxy sidecar]
    ↓ (uses mesh VirtualService for routing)
[VirtualService: ratelimit-mesh]
    ↓ (gateway: mesh - built-in Istio keyword)
Kubernetes DNS: ratelimit.ratelimit.svc.cluster.local
    ↓
[Service B's istio-proxy sidecar]
    ↓ (authorization check, mTLS decryption)
[Application Container: ratelimit]
```

---

## VirtualService Deep Dive

### What is a VirtualService?
An Istio CRD that defines **routing rules** for traffic. Like an advanced reverse proxy configuration.

### Three VirtualService Types in Your Deployment

#### 1. Internal Ingress VirtualService
**Purpose:** REST API routing for internal gateways
```yaml
virtualService:
  enabled: true
  spec:
    http:
      - match:
          - uri:
              prefix: "/ratelimit/v1/"
        name: ratelimit
        route:
          - destination:
              host: "{{ .Release.Name }}"
              port:
                number: 8081
            weight: 100
```
- **Gateway:** Internal gateways (auto-configured by Helm chart)
- **Hosts:** Auto-configured by chart
- **Use Case:** REST API calls within the cluster via ingress

#### 2. External Ingress VirtualService (Your Implementation)
**Purpose:** External gRPC access via public gateway
```yaml
additionalVirtualService:
  enabled: true
  virtualServices:
    grpc:
      enabled: true
      name: grpc-external
      gatewayName: istio-resources/services-gateway  # EXPLICIT external gateway
      hosts:
        - "*"
      spec:
        http:
          - match:
              - uri:
                  prefix: "/docusign.ratelimit.v1."  # gRPC package path
            name: ratelimit-grpc-external
            route:
              - destination:
                  host: "{{ .Release.Name }}"
                  port:
                    number: 8081
                weight: 100
```
- **Gateway:** `istio-resources/services-gateway` (external, LoadBalancer)
- **Hosts:** `*` (accepts all hostnames)
- **Use Case:** External gRPC calls from outside the cluster

#### 3. Mesh VirtualService (Auto-Generated)
**Purpose:** Service-to-service communication
```yaml
# Auto-generated when internalApi.enabled: true
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratelimit-mesh
spec:
  hosts:
    - ratelimit.ratelimit.svc.cluster.local
  gateways:
    - mesh  # Special Istio keyword for sidecar-to-sidecar routing
  http:
    - name: ratelimit-mesh
      route:
        - destination:
            host: ratelimit
            port:
              number: 8081
```
- **Gateway:** `mesh` (not a real Gateway resource, built-in Istio concept)
- **Hosts:** Service's Kubernetes DNS name
- **Use Case:** When Service A calls Service B within the cluster

### virtualService vs additionalVirtualService

| Feature | `virtualService` | `additionalVirtualService` |
|---------|------------------|---------------------------|
| **Purpose** | Primary VirtualService for the service | Additional VirtualServices for special cases |
| **Validation** | Helm chart applies validation rules | Bypasses some Helm chart validations |
| **Count** | One | Multiple (named: grpc, customDomain, etc.) |
| **Gateway Control** | Auto-configured by chart | Explicit `gatewayName` specification |
| **When to Use** | Standard REST API routing | gRPC routing, multi-gateway scenarios, special routing needs |

**Why you used `additionalVirtualService`:**
- Needed explicit external gateway (`istio-resources/services-gateway`)
- gRPC path prefix (`/docusign.ratelimit.v1.`) might not pass Helm validation
- Separation of concerns: REST routes in primary, gRPC routes in additional

---

## Gateway Types

### What is an Istio Gateway?
Configures a **load balancer** (typically Envoy proxy) for inbound/outbound traffic at the edge of the mesh.

### Gateway Types

#### 1. Ingress Gateway (External Traffic)
**Purpose:** Accept traffic from outside the cluster
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: services-gateway
  namespace: istio-resources
spec:
  selector:
    istio: ingressgateway  # Targets istio-ingressgateway pods
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - "*.services.true.dev.docusign.net"
      tls:
        mode: SIMPLE
        credentialName: tls-cert
```
- **Selector:** Points to Istio ingress gateway pods (LoadBalancer service)
- **Hosts:** Accepts traffic for these DNS patterns
- **TLS:** Handles SSL termination

**Your Gateway:** `istio-resources/services-gateway`
- Shared across multiple services
- Public IP: 4.227.44.227
- DNS: `*.services.true.dev.docusign.net`

#### 2. Mesh Gateway (Service-to-Service)
**Purpose:** Sidecar-to-sidecar communication

**Important:** `mesh` is a **special keyword**, NOT a real Gateway resource!
```bash
kubectl get gateway mesh -n ratelimit
# Error: gateways.networking.istio.io "mesh" not found
```

When you specify `gateways: [mesh]` in a VirtualService:
- Istio applies routing rules to **sidecar proxies** (not gateway pods)
- Enables service-to-service routing within the mesh
- Automatically available when Istio is enabled

**Enabled by:** `internalApi.enabled: true` in app-values.yaml
- Helm chart auto-generates VirtualServices with `gateways: [mesh]`
- No explicit Gateway resource needed

#### 3. Egress Gateway (Outbound Traffic)
**Purpose:** Control traffic leaving the mesh (not used in your deployment)

---

## Authorization Policies

### What is AuthorizationPolicy?
Istio CRD for **access control** (RBAC). Enforced at the **pod's istio-proxy sidecar**.

### Your Configuration
```yaml
authorization:
  allow:
    enabled: true
    name: ratelimit-authz
    rules:
    - from:
      - source:
          namespaces: 
          - "ratelimit"
          - "istio-system"
          principals:
            - "cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
```

### How it Works
1. **Request arrives** at ratelimit pod
2. **Istio-proxy sidecar** checks AuthorizationPolicy
3. **Extracts identity** from mTLS certificate or JWT
4. **Matches against rules:**
   - ✅ From namespace `ratelimit` → ALLOW
   - ✅ From namespace `istio-system` → ALLOW
   - ✅ Principal `istio-ingressgateway-service-account` → ALLOW
   - ❌ All others → DENY (403 Forbidden)

### Why You Needed the Ingress Gateway Principal
External requests flow through istio-ingressgateway:
```
External Client → Ingress Gateway (sa: istio-ingressgateway-service-account)
                               ↓
                    VirtualService → Service → Pod
                                              ↓
                                    AuthorizationPolicy Check
```

Without adding `istio-ingressgateway-service-account` principal:
- Authorization check sees request from `istio-ingressgateway-service-account`
- No matching rule → **Denied** → `PermissionDenied: RBAC: access denied`

### Troubleshooting AuthorizationPolicy
**Check istio-proxy logs:**
```bash
kubectl logs -n ratelimit <pod-name> -c istio-proxy --tail=50
```
Look for:
```json
{
  "response_code_details": "rbac_access_denied_matched_policy[none]",
  "x_forwarded_client_cert": "...;URI=spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
}
```

---

## Helm Values: publicApi vs internalApi

### Comparison Table

| Feature | `internalApi` | `publicApi` |
|---------|---------------|-------------|
| **Purpose** | Service-to-service (mesh) routing | External ingress routing |
| **Gateway Type** | `mesh` (sidecar-to-sidecar) | External ingress gateway |
| **VirtualService Generated** | Yes, with `gateways: [mesh]` | Yes, with external gateway reference |
| **DNS Scope** | `service.namespace.svc.cluster.local` | Public DNS (e.g., `*.services.true.dev.docusign.net`) |
| **Access From** | Other services in the mesh | External clients (internet, corporate network) |
| **Your Usage** | ✅ Enabled (`internalApi.enabled: true`) | ❌ Not used (using `additionalVirtualService` instead) |

### internalApi Configuration
```yaml
internalApi:
  enabled: true
```

**When enabled, Helm chart generates:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratelimit-mesh
spec:
  hosts:
    - ratelimit.ratelimit.svc.cluster.local
  gateways:
    - mesh
  http:
    - name: ratelimit-mesh
      route:
        - destination:
            host: ratelimit
            port:
              number: 8081
```

**Result:** Other services can call:
```
ratelimit.ratelimit.svc.cluster.local:8081
```

### publicApi Configuration
```yaml
publicApi:
  enabled: true
```

**When enabled, Helm chart generates:**
- VirtualService bound to external gateway
- Default routing rules for public access
- Auto-configured hosts and paths

**Your Approach:**
You used `additionalVirtualService` instead for **more control**:
- Explicit gateway selection (`istio-resources/services-gateway`)
- Custom gRPC path matching (`/docusign.ratelimit.v1.`)
- Fine-grained authorization configuration

---

## Without Istio: Alternatives

### If Your Cluster Doesn't Use Istio

#### What You Lose
- ❌ VirtualService (no L7 routing)
- ❌ Gateway resources
- ❌ AuthorizationPolicy (no mesh-level RBAC)
- ❌ Sidecar proxies (no mTLS, observability)
- ❌ Advanced traffic management (canary, traffic splitting)

#### What You Use Instead

**1. Kubernetes Service**
```yaml
service:
  type: LoadBalancer  # or ClusterIP with Ingress
  port: 8081
```

**2. Kubernetes Ingress (for HTTP/HTTPS)**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ratelimit-ingress
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"  # For gRPC
spec:
  ingressClassName: nginx
  rules:
    - host: ratelimit.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ratelimit
                port:
                  number: 8081
```

**3. Ingress Controllers**
Popular options:
- **nginx-ingress**: Most common, supports gRPC
- **Traefik**: Modern, dynamic configuration
- **HAProxy**: High performance
- **Contour**: Envoy-based (like Istio, but simpler)

**4. Service-to-Service Communication**
Direct Kubernetes DNS:
```go
// Service A calls Service B
conn, err := grpc.Dial("ratelimit.ratelimit.svc.cluster.local:8081")
```
- No sidecar, no VirtualService routing
- Plain TCP connection
- No automatic mTLS (must implement yourself)

---

## Key Takeaways

### 1. Istio Service Mesh
- **Adds sidecar proxies** to every pod for traffic management and security
- **VirtualServices** = L7 routing rules (like advanced nginx config)
- **Gateways** = Load balancer configuration for edge traffic

### 2. Gateway Types
- **External Gateway** (`istio-resources/services-gateway`): Real Gateway resource, LoadBalancer service
- **Mesh Gateway** (`mesh` keyword): Not a resource, represents sidecar proxies for service-to-service

### 3. VirtualService Routing
- **Match URI prefix** for routing (`/docusign.ratelimit.v1.` for gRPC, `/ratelimit/v1/` for REST)
- **Gateway binding** determines where rules apply (external vs mesh)
- **Weight-based routing** enables canary deployments (100/0 split in your case)

### 4. Traffic Flow
```
External: DNS → Azure LB → Ingress Gateway → VirtualService → Service → Pod → App
Internal: Service A → Sidecar → Mesh VirtualService → Service B Sidecar → App
```

### 5. Authorization
- **Enforced at the destination pod's sidecar** (not at gateway)
- **Must include ingress gateway principal** for external access
- **Check istio-proxy logs** for troubleshooting denials

### 6. Helm Chart Patterns
- **`{{ .Release.Name }}`**: Use variables for multi-tenant deployments
- **`virtualService`**: Primary routing configuration
- **`additionalVirtualService`**: Bypass validations, add custom routes
- **`internalApi.enabled`**: Auto-generates mesh VirtualServices
- **`publicApi.enabled`**: Auto-generates external VirtualServices (you didn't use this)

### 7. Your Implementation Strategy
1. ✅ Used `additionalVirtualService` for external gRPC access
2. ✅ Explicitly specified `gatewayName: istio-resources/services-gateway`
3. ✅ Fixed VirtualService path prefix to match proto package
4. ✅ Added ingress gateway principal to AuthorizationPolicy
5. ✅ Simplified to shared deployment (release name: `ratelimit`)
6. ✅ Result: External access working at `services.true.dev.docusign.net:443`

---

## References

### Istio Documentation
- [VirtualService](https://istio.io/latest/docs/reference/config/networking/virtual-service/)
- [Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/)
- [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/)

### Kubernetes Documentation
- [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

### Helm Documentation
- [Template Functions](https://helm.sh/docs/chart_template_guide/functions_and_pipelines/)
- [Built-in Objects](https://helm.sh/docs/chart_template_guide/builtin_objects/)

---

**Document Created:** Learning from TrueDev RateLimit External gRPC Access Implementation
**Date:** February 13, 2026
