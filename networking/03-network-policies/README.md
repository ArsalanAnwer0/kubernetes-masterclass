# Network Policies - Traffic Control & Security

## Learning Objectives
By the end of this section, you will understand:
- How network policies control traffic flow
- Ingress and egress traffic rules
- Namespace isolation strategies
- Advanced security patterns

## What's in this folder

| File | Description | Policy Type | Security Level |
|------|-------------|-------------|----------------|
| `deny-all-policy.yaml` | Block all traffic | Ingress | Maximum |
| `allow-specific-pods.yaml` | Allow specific pod communication | Ingress | Selective |
| `namespace-isolation.yaml` | Isolate namespaces | Ingress | Namespace-level |
| `egress-policy.yaml` | Control outbound traffic | Egress | Outbound control |
| `complete-network-policy.yaml` | Full ingress/egress control | Both | Production-ready |

## Quick Start

```bash
# Check if network policies are supported
kubectl get networkpolicies

# Deploy test applications
kubectl create deployment frontend --image=nginx:1.21
kubectl create deployment backend --image=httpd:2.4
kubectl create deployment database --image=postgres:13

# Apply basic deny-all policy
kubectl apply -f deny-all-policy.yaml

# Test connectivity (should be blocked)
kubectl exec -it deployment/frontend -- curl backend
```

## Network Policy Concepts

### What are Network Policies?
- **Firewall for Pods**: Control network traffic to/from pods
- **Default Allow**: Kubernetes allows all traffic by default
- **Additive Rules**: Multiple policies are combined (OR logic)
- **CNI Dependent**: Requires CNI plugin support (Calico, Cilium, etc.)

### Policy Types
- **Ingress**: Control incoming traffic TO pods
- **Egress**: Control outgoing traffic FROM pods
- **Both**: Control traffic in both directions

### Selection Methods
- **Pod Selector**: Select pods by labels
- **Namespace Selector**: Select entire namespaces
- **IP Block**: Select by IP ranges (CIDR)

## Security Models

### Default Deny Model (Recommended)
1. Deny all traffic by default
2. Explicitly allow only required communication
3. Principle of least privilege

### Namespace Isolation Model
1. Isolate namespaces from each other
2. Allow communication within namespace
3. Explicit cross-namespace rules

## Hands on

### Test Default Behavior (No Policies)
```bash
# Create test namespaces and apps
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace database

# Deploy applications
kubectl create deployment web --image=nginx:1.21 -n frontend
kubectl create deployment api --image=httpd:2.4 -n backend
kubectl create deployment db --image=postgres:13 -n database --env POSTGRES_PASSWORD=secret

# Expose services
kubectl expose deployment web --port=80 -n frontend
kubectl expose deployment api --port=80 -n backend
kubectl expose deployment db --port=5432 -n database

# Test connectivity (should work)
kubectl exec -n frontend deployment/web -- curl api.backend.svc.cluster.local
kubectl exec -n backend deployment/api -- curl db.database.svc.cluster.local:5432
```

### Deny All Traffic
```bash
# Apply deny-all policy to backend namespace
kubectl apply -f deny-all-policy.yaml -n backend

# Test connectivity (should be blocked)
kubectl exec -n frontend deployment/web -- curl --connect-timeout 5 api.backend.svc.cluster.local
# Should timeout - traffic blocked

# Check from within backend namespace
kubectl exec -n backend deployment/api -- curl --connect-timeout 5 db.database.svc.cluster.local:5432
# Should also timeout - egress blocked
```

### Allow Specific Pod Communication
```bash
# Apply policy to allow frontend to backend
kubectl apply -f allow-specific-pods.yaml

# Test allowed communication
kubectl exec -n frontend deployment/web -- curl api.backend.svc.cluster.local
# Should work now

# Test still blocked communication
kubectl run test-pod --image=curlimages/curl -n default -i --tty --rm -- sh
curl api.backend.svc.cluster.local
# Should still be blocked (not from frontend namespace)
exit
```

### Namespace Isolation
```bash
# Apply namespace isolation policies
kubectl apply -f namespace-isolation.yaml

# Test within namespace (should work)
kubectl exec -n frontend deployment/web -- curl web.frontend.svc.cluster.local

# Test cross-namespace (should be blocked)
kubectl exec -n frontend deployment/web -- curl --connect-timeout 5 api.backend.svc.cluster.local
# Should be blocked by namespace isolation
```

### Egress Policies
```bash
# Apply egress policy (control outbound traffic)
kubectl apply -f egress-policy.yaml -n backend

# Test allowed egress (to database)
kubectl exec -n backend deployment/api -- curl db.database.svc.cluster.local:5432

# Test blocked egress (to internet)
kubectl exec -n backend deployment/api -- curl --connect-timeout 5 google.com
# Should be blocked
```

### Complete Network Policy
```bash
# Apply comprehensive policy
kubectl apply -f complete-network-policy.yaml

# Test all scenarios
# 1. Frontend to backend (allowed)
kubectl exec -n frontend deployment/web -- curl api.backend.svc.cluster.local

# 2. Backend to database (allowed)
kubectl exec -n backend deployment/api -- curl db.database.svc.cluster.local:5432

# 3. Direct frontend to database (blocked)
kubectl exec -n frontend deployment/web -- curl --connect-timeout 5 db.database.svc.cluster.local:5432

# 4. External access to backend (configure as needed)
```

## Example Network Policy Configurations

### Deny All Traffic
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Allow Specific Pod Communication
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 80
```

### Namespace Isolation
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: {{ .Release.Namespace }}
```

### Egress Policy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: database
    ports:
    - protocol: TCP
      port: 5432
  - to: []
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
```

## Advanced Network Policy Patterns

### Multi-tier Application Security
```yaml
# Three-tier architecture policies
# 1. Frontend: Accept from LoadBalancer, connect to backend
# 2. Backend: Accept from frontend, connect to database
# 3. Database: Accept from backend only
```

### Microservices Communication
```yaml
# Service mesh-like policies
# Each service can only communicate with specific services
# Based on service identity (labels)
```

### Development vs Production
```yaml
# Development: More permissive
# - Allow debugging access
# - Allow external connections for testing

# Production: Strict isolation
# - Minimal required communication only
# - No external access except through approved channels
```

## Troubleshooting

### Network Policy Not Working
```bash
# Check CNI plugin supports network policies
kubectl get nodes -o wide
kubectl describe node <node-name> | grep -i cni

# Common CNI plugins with network policy support:
# - Calico 
# - Cilium 
# - Weave Net 
# - Flannel  (basic version)
```

### Traffic Still Allowed/Blocked
```bash
# Check all network policies affecting a pod
kubectl get networkpolicy --all-namespaces
kubectl describe networkpolicy <policy-name> -n <namespace>

# Check pod labels match selectors
kubectl get pods --show-labels -n <namespace>

# Verify policy syntax
kubectl get networkpolicy <policy-name> -o yaml
```

### Debugging Connectivity
```bash
# Test connectivity with verbose output
kubectl exec -n <namespace> deployment/<app> -- curl -v --connect-timeout 10 <target>

# Check service endpoints
kubectl get endpoints <service-name> -n <namespace>

# Use network debugging tools
kubectl run netshoot --rm -i --tty --image nicolaka/netshoot -- /bin/bash
# Inside netshoot:
nmap -p 80 <service-ip>
tcpdump -i any host <pod-ip>
exit
```

### Policy Conflicts
```bash
# List all policies affecting a namespace
kubectl get networkpolicy -n <namespace>

# Check policy precedence (all policies are additive)
kubectl describe networkpolicy --all-namespaces | grep -A 20 "Spec:"

# Test policy effects incrementally
kubectl delete networkpolicy <policy-name> -n <namespace>
# Test connectivity
kubectl apply -f <policy-file>
# Test again
```

## Network Policy Best Practices

### Security Principles
 **Default Deny**: Start with deny-all, add specific allows  
 **Least Privilege**: Allow only necessary communication  
 **Defense in Depth**: Combine with other security measures  
 **Regular Audit**: Review and update policies regularly  

### Design Patterns
 **Namespace Isolation**: Separate environments and teams  
 **Service-to-Service**: Control microservice communication  
 **Ingress Control**: Manage incoming traffic carefully  
 **Egress Control**: Monitor and control outbound traffic  

### Operational Practices
 **Test Thoroughly**: Verify policies don't break functionality  
 **Monitor Traffic**: Use observability tools  
 **Document Policies**: Maintain clear documentation  
 **Version Control**: Track policy changes  

## Common Security Scenarios

### Database Protection
```yaml
# Only allow backend services to access database
# Block direct access from frontend or external sources
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-access
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: backend
    ports:
    - protocol: TCP
      port: 5432
```

### Multi-tenant Isolation
```yaml
# Isolate different tenants/customers
# Each tenant namespace can't access others
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-isolation
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant: "tenant-a"
```

### External API Access Control
```yaml
# Control which services can access external APIs
# Prevent data exfiltration
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: external-api-access
spec:
  podSelector:
    matchLabels:
      allowed-external: "true"
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 80
```

## Production Deployment Strategy

### Phase 1: Observation
```bash
# 1. Deploy without policies
# 2. Monitor traffic patterns
# 3. Identify communication requirements
# 4. Document service dependencies
```

### Phase 2: Gradual Implementation
```bash
# 1. Start with non-critical namespaces
# 2. Apply namespace isolation first
# 3. Add service-specific policies
# 4. Test thoroughly at each step
```

### Phase 3: Full Security
```bash
# 1. Apply deny-all policies
# 2. Add explicit allow rules
# 3. Monitor for blocked legitimate traffic
# 4. Refine policies based on feedback
```

## Monitoring and Observability

### Policy Effectiveness
```bash
# Check policy hits/effectiveness
kubectl get networkpolicy -o wide
kubectl describe networkpolicy <policy-name>

# Monitor dropped connections (CNI-specific)
# Calico example:
kubectl logs -n calico-system daemonset/calico-node | grep "DROP"
```

### Traffic Analysis
```bash
# Use service mesh for detailed traffic analysis
# Istio, Linkerd provide traffic visibility

# CNI plugin monitoring
# Cilium Hubble, Calico observability features

# Traditional monitoring
kubectl top pods --containers
kubectl logs <pod-name> | grep "connection refused"
```

## CNI Plugin Support

| CNI Plugin | Network Policy Support | Features |
|------------|------------------------|----------|
| Calico |  Full support | Advanced policies, observability |
| Cilium |  Full support | eBPF-based, rich features |
| Weave Net |  Basic support | Simple implementation |
| Flannel |  Limited | Basic networking only |
| Antrea |  Full support | VMware-backed solution |

## Policy Testing Strategy

### Testing Framework
```bash
# 1. Create test pods with known labels
kubectl run test-client --image=curlimages/curl --labels="app=test-client"
kubectl run test-server --image=nginx --labels="app=test-server"

# 2. Apply policy
kubectl apply -f test-policy.yaml

# 3. Test connectivity
kubectl exec test-client -- curl test-server

# 4. Verify expected behavior
# - Should succeed if policy allows
# - Should timeout if policy denies
```

### Automated Testing
```bash
# Use tools like:
# - Falco for runtime security monitoring
# - OPA Gatekeeper for policy validation
# - Custom test scripts for connectivity validation
```

## Key Takeaways

 **Network Policies provide pod-level firewall rules**  
 **Default behavior allows all traffic** - must explicitly deny  
 **CNI plugin support required** - not all CNI plugins support policies  
 **Additive rules** - multiple policies combine with OR logic  
 **Ingress and egress control** - bidirectional traffic management  
 **Label-based selection** - flexible pod and namespace targeting  

---
