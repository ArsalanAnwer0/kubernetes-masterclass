# Service Types - Network Access Patterns

## Learning Objectives
By the end of this section, you will understand:
- All Kubernetes service types and their use cases
- How service discovery and DNS work
- Load balancing strategies and session affinity
- When to use each service type in production

## What's in this folder

| File | Service Type | Purpose | Access Level |
|------|--------------|---------|--------------|
| `clusterip-service.yaml` | ClusterIP | Internal communication | Cluster-only |
| `nodeport-service.yaml` | NodePort | External access via node ports | External |
| `loadbalancer-service.yaml` | LoadBalancer | Cloud load balancer | External |
| `externalname-service.yaml` | ExternalName | DNS alias to external service | External |
| `multi-port-service.yaml` | Multi-port | Multiple ports on same service | Any |

## Quick Start

```bash
# Deploy a sample application first
kubectl create deployment web-app --image=nginx:1.21 --replicas=3
kubectl create deployment api-app --image=httpd:2.4 --replicas=2

# Try different service types
kubectl apply -f clusterip-service.yaml
kubectl apply -f nodeport-service.yaml
kubectl apply -f loadbalancer-service.yaml

# Check services
kubectl get services
kubectl get svc -o wide
```

## Service Types Explained

### 1. ClusterIP (Default)
- **Access**: Internal cluster only
- **IP**: Virtual cluster IP
- **Use Case**: Internal microservice communication
- **DNS**: `<service-name>.<namespace>.svc.cluster.local`

### 2. NodePort
- **Access**: External via node IP:port
- **Port Range**: 30000-32767
- **Use Case**: Development, testing, legacy apps
- **Access**: `<node-ip>:<node-port>`

### 3. LoadBalancer
- **Access**: External via cloud load balancer
- **Provider**: Cloud provider (AWS, GCP, Azure)
- **Use Case**: Production external access
- **Access**: Cloud-provided external IP

### 4. ExternalName
- **Access**: DNS alias to external service
- **No Proxy**: Direct DNS resolution
- **Use Case**: External database, API integration
- **Access**: Standard DNS name

## Hands on

### ClusterIP Service (Internal)
```bash
# Create ClusterIP service
kubectl apply -f clusterip-service.yaml
kubectl get svc web-clusterip
kubectl describe svc web-clusterip

# Test internal access
kubectl run test-pod --image=curlimages/curl -i --tty --rm -- sh
# Inside the pod:
curl web-clusterip
curl web-clusterip.default.svc.cluster.local
nslookup web-clusterip
exit

# Check endpoints
kubectl get endpoints web-clusterip
kubectl describe endpoints web-clusterip
```

### NodePort Service (External)
```bash
# Create NodePort service
kubectl apply -f nodeport-service.yaml
kubectl get svc web-nodeport
kubectl describe svc web-nodeport

# Get node IP and port
kubectl get nodes -o wide
kubectl get svc web-nodeport -o jsonpath='{.spec.ports[0].nodePort}'

# Test external access
# Access via: http://<node-ip>:<node-port>
curl http://<node-ip>:<node-port>

# Test from inside cluster
kubectl run test-pod --image=curlimages/curl -i --tty --rm -- sh
curl web-nodeport:80
exit
```

### LoadBalancer Service (Cloud)
```bash
# Create LoadBalancer service (requires cloud provider)
kubectl apply -f loadbalancer-service.yaml
kubectl get svc web-loadbalancer

# Wait for external IP (may take a few minutes)
kubectl get svc web-loadbalancer -w

# Test external access
kubectl get svc web-loadbalancer -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
curl http://<external-ip>

# Check load balancer details
kubectl describe svc web-loadbalancer
```

### ExternalName Service
```bash
# Create ExternalName service
kubectl apply -f externalname-service.yaml
kubectl get svc external-api
kubectl describe svc external-api

# Test DNS resolution
kubectl run test-pod --image=curlimages/curl -i --tty --rm -- sh
# Inside the pod:
nslookup external-api
curl external-api  # This will resolve to httpbin.org
exit
```

### Multi-port Service
```bash
# Create multi-port service
kubectl apply -f multi-port-service.yaml
kubectl get svc multi-port-service
kubectl describe svc multi-port-service

# Test different ports
kubectl run test-pod --image=curlimages/curl -i --tty --rm -- sh
# Inside the pod:
curl multi-port-service:80      # HTTP port
curl multi-port-service:8080    # Admin port
exit
```

## Service Discovery and DNS

### DNS Names
```bash
# Service DNS patterns
<service-name>                              # Same namespace
<service-name>.<namespace>                  # Different namespace
<service-name>.<namespace>.svc.cluster.local # Full FQDN

# Examples
curl web-clusterip                          # Same namespace
curl web-clusterip.default                 # Specify namespace
curl web-clusterip.default.svc.cluster.local # Full DNS name
```

### Service Discovery Test
```bash
# Create services in different namespaces
kubectl create namespace frontend
kubectl create namespace backend

# Deploy apps in different namespaces
kubectl create deployment web --image=nginx:1.21 -n frontend
kubectl create deployment api --image=httpd:2.4 -n backend

# Create services
kubectl expose deployment web --port=80 -n frontend
kubectl expose deployment api --port=80 -n backend

# Test cross-namespace communication
kubectl run test -n frontend --image=curlimages/curl -i --tty --rm -- sh
# Inside the pod:
curl web              # Same namespace
curl api.backend      # Different namespace
curl api.backend.svc.cluster.local  # Full FQDN
exit
```

## Load Balancing and Session Affinity

### Session Affinity Example
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sticky-service
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
  sessionAffinity: ClientIP  # Stick to same pod
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600   # 1 hour timeout
```

### Load Balancing Test
```bash
# Scale deployment to see load balancing
kubectl scale deployment web-app --replicas=5
kubectl get pods -l app=web-app -o wide

# Test load balancing
kubectl run test-pod --image=curlimages/curl -i --tty --rm -- sh
# Inside the pod:
for i in {1..10}; do curl web-clusterip; echo ""; done
# You should see responses from different pods
exit
```

## Service Type Comparison

| Aspect | ClusterIP | NodePort | LoadBalancer | ExternalName |
|--------|-----------|----------|--------------|--------------|
| **Accessibility** | Internal only | External via node | External via LB | External DNS alias |
| **IP Assignment** | Cluster IP | Cluster IP + Node Port | Cluster IP + External IP | None |
| **Use Case** | Microservices | Development/Testing | Production | External integrations |
| **Cloud Dependency** | No | No | Yes | No |
| **Port Range** | Any | 30000-32767 | Any | N/A |

## Troubleshooting

### Service Not Accessible
```bash
# Check service exists
kubectl get svc <service-name>

# Check endpoints
kubectl get endpoints <service-name>
kubectl describe endpoints <service-name>

# Check pod labels match service selector
kubectl get pods --show-labels
kubectl describe svc <service-name>

# Check pod readiness
kubectl get pods
kubectl describe pod <pod-name>
```

### DNS Issues
```bash
# Test DNS resolution
kubectl run test-pod --image=busybox:1.35 -i --tty --rm -- sh
nslookup kubernetes.default.svc.cluster.local
nslookup <service-name>
exit

# Check CoreDNS
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system deployment/coredns
```

### LoadBalancer Pending
```bash
# Check cloud provider support
kubectl describe svc <loadbalancer-service>

# Check cloud controller manager
kubectl get pods -n kube-system | grep cloud-controller

# Check provider-specific logs
kubectl logs -n kube-system <cloud-controller-pod>
```

## Example Service Configurations

### ClusterIP Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-clusterip
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

### NodePort Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    protocol: TCP
```

### LoadBalancer Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

### ExternalName Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-api
spec:
  type: ExternalName
  externalName: httpbin.org
  ports:
  - port: 80
```

## Best Practices

### Service Design
 Use ClusterIP for internal services  
 Use LoadBalancer for production external access  
 Avoid NodePort in production (use for development only)  
 Use meaningful service names for easy discovery  
 Label services for organization  

### Performance
 Enable session affinity when needed  
 Use appropriate port numbers  
 Monitor service endpoints  
 Scale backend pods based on traffic  

### Security
 Limit service exposure (don't expose what's not needed)  
 Use Network Policies for additional security  
 Monitor service access  
 Use TLS for sensitive services  

## Advanced Configuration

### Multi-Port Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
spec:
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: admin
    port: 8080
    targetPort: 8080
```

### Service with Health Check
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
  healthCheckNodePort: 32000
```

## Monitoring and Observability

### Key Metrics to Monitor
- **Service availability**: Are endpoints healthy?
- **Response times**: How fast are services responding?
- **Connection counts**: How many active connections?
- **Error rates**: What percentage of requests fail?

### Useful Commands for Monitoring
```bash
# Check service status
kubectl get svc -o wide

# Monitor endpoints
kubectl get endpoints

# Check service events
kubectl get events --field-selector involvedObject.kind=Service

# Test service connectivity
kubectl run test --image=curlimages/curl -i --tty --rm -- curl <service-name>
```

## Key Takeaways

 **ClusterIP**: Default, internal cluster communication  
 **NodePort**: External access via node ports (development)  
 **LoadBalancer**: Production external access via cloud LB  
 **ExternalName**: DNS alias to external services  
 **Service discovery**: DNS-based, automatic  
 **Load balancing**: Built-in, configurable  

---
