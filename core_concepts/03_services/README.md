# Services - Networking Your Applications

## What is a Service?

A Service provides a stable network endpoint to access your pods. Since pods are ephemeral and get new IP addresses when they restart, Services provide:

- **Stable IP Address**: Doesn't change when pods restart
- **DNS Name**: Access via name instead of IP
- **Load Balancing**: Distributes traffic across multiple pods
- **Service Discovery**: Find and connect to other services

## Service Types

| Type | Description | Use Case | Access Method |
|------|-------------|----------|---------------|
| **ClusterIP** | Internal access only | Microservices communication | Pod-to-pod within cluster |
| **NodePort** | Exposes service on each node | Development/testing | `<NodeIP>:<NodePort>` |
| **LoadBalancer** | Cloud provider load balancer | Production external access | External load balancer |
| **Headless** | Direct pod access | StatefulSets, databases | Direct pod IPs via DNS |

## Files in this folder

1. `clusterip-service.yaml` - Internal service for pod-to-pod communication
2. `nodeport-service.yaml` - External access via node ports (30000-32767)
3. `loadbalancer-service.yaml` - Cloud provider load balancer
4. `headless-service.yaml` - Direct pod access without load balancing
5. `multi-port-service.yaml` - Service exposing multiple ports

## Hands-on Practice

### Prerequisites: Create a deployment first

```bash
# Navigate to deployments folder and create pods
cd ../02-deployments
kubectl apply -f simple-deployment.yaml

# Verify pods are running
kubectl get pods -l app=nginx

# Return to services folder
cd ../03-services
```

### Step 1: ClusterIP Service (Internal Communication)

```bash
kubectl apply -f clusterip-service.yaml
kubectl get services
kubectl describe service nginx-service-clusterip
```

**Expected Output:**
```
NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx-service-clusterip  ClusterIP   10.100.1.100   <none>        80/TCP    5s
```

### Step 2: Test internal access

```bash
# Create a temporary test pod
kubectl run test-pod --image=curlimages/curl -i --tty --rm -- sh
```

**Inside the test pod, try these commands:**
```bash
# Test service by name (same namespace)
curl nginx-service-clusterip

# Test with full DNS name
curl nginx-service-clusterip.default.svc.cluster.local

# Check DNS resolution
nslookup nginx-service-clusterip

# Exit the test pod
exit
```

### Step 3: NodePort Service (External Access)

```bash
kubectl apply -f nodeport-service.yaml
kubectl get services
```

**Expected Output:**
```
NAME                     TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-service-nodeport   NodePort   10.100.1.101   <none>        80:30080/TCP   5s
```

**Access the service:**
- **Minikube**: `minikube service nginx-service-nodeport --url`
- **Docker Desktop**: `http://localhost:30080`
- **Cloud/VM**: `http://<node-ip>:30080`

### Step 4: Port forwarding (Easy local testing)

```bash
# Forward service port to your local machine
kubectl port-forward service/nginx-service-clusterip 8080:80

# In another terminal or browser, visit:
# http://localhost:8080
```

### Step 5: Explore service endpoints

```bash
kubectl get endpoints nginx-service-clusterip
kubectl describe endpoints nginx-service-clusterip
```

**Understanding endpoints:**
```bash
# Endpoints show the actual pod IPs backing the service
kubectl get pods -o wide -l app=nginx
kubectl get endpoints nginx-service-clusterip -o yaml
```

### Step 6: LoadBalancer Service (Cloud environments)

```bash
# Only works in cloud environments (GKE, EKS, AKS)
kubectl apply -f loadbalancer-service.yaml
kubectl get services

# Wait for external IP (may take a few minutes)
kubectl get services -w
```

### Step 7: Headless Service

```bash
kubectl apply -f headless-service.yaml
kubectl get services

# Test DNS resolution - returns individual pod IPs
kubectl run dns-test --image=busybox -i --tty --rm -- sh
nslookup nginx-headless-service
exit
```

## Essential Commands Cheat Sheet

### Service Management
```bash
# Create/Apply service
kubectl apply -f service.yaml

# Get services
kubectl get services
kubectl get svc  # Short form

# Detailed service info
kubectl describe service <service-name>

# Get service in different formats
kubectl get service <service-name> -o yaml
kubectl get service <service-name> -o wide

# Delete service
kubectl delete service <service-name>
```

### Testing and Debugging
```bash
# Port forwarding for testing
kubectl port-forward service/<service-name> <local-port>:<service-port>

# Check endpoints
kubectl get endpoints <service-name>
kubectl describe endpoints <service-name>

# Test from within cluster
kubectl run test-pod --image=curlimages/curl -i --tty --rm -- sh

# Check service logs (via pods)
kubectl logs -l <label-selector>
```

### Service Discovery
```bash
# DNS names for services
<service-name>                                # Same namespace
<service-name>.<namespace>                   # Cross namespace
<service-name>.<namespace>.svc.cluster.local # Full FQDN

# Environment variables (legacy)
kubectl exec <pod-name> -- env | grep SERVICE
```

## Service Configuration Examples

### Basic ClusterIP Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80        # Service port
      targetPort: 8080 # Container port
      protocol: TCP
  type: ClusterIP
```

### NodePort with Custom Port
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-nodeport
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080  # Must be 30000-32767
  type: NodePort
```

### LoadBalancer with Annotations
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  type: LoadBalancer
```

### Multi-Port Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
spec:
  selector:
    app: my-app
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
```

## Common Troubleshooting Scenarios

### Service Not Accessible

1. **Check if service exists:**
   ```bash
   kubectl get services
   ```

2. **Verify endpoints:**
   ```bash
   kubectl get endpoints <service-name>
   ```

3. **Check pod labels match service selector:**
   ```bash
   kubectl get pods --show-labels
   kubectl describe service <service-name>
   ```

### No Endpoints Available

1. **Check if pods are running:**
   ```bash
   kubectl get pods -l <label-selector>
   ```

2. **Verify label selectors:**
   ```bash
   kubectl describe service <service-name>
   kubectl get pods --show-labels
   ```

3. **Check pod readiness:**
   ```bash
   kubectl describe pod <pod-name>
   ```

### External Access Issues

1. **For NodePort:**
   ```bash
   kubectl get nodes -o wide  # Get node IPs
   kubectl get services       # Get NodePort
   ```

2. **For LoadBalancer:**
   ```bash
   kubectl describe service <service-name>
   # Check events for provisioning issues
   ```

## Best Practices

### Naming Conventions
- Use descriptive names: `user-api-service`, `database-service`
- Include environment: `user-api-prod`, `user-api-dev`
- Be consistent across your organization

### Port Naming
```yaml
ports:
  - name: http    # Named ports are recommended
    port: 80
    targetPort: http  # Reference container port by name
```

### Health Checks Integration
```yaml
# In your deployment
ports:
  - name: http
    containerPort: 8080
  - name: health
    containerPort: 8081

# In your service
ports:
  - name: http
    port: 80
    targetPort: http
```

### Security Considerations
- Use ClusterIP for internal services
- Limit NodePort range in production
- Use Network Policies to restrict traffic
- Consider service mesh for advanced networking

## Service vs Ingress vs Gateway

| Component | Purpose | Level | Best For |
|-----------|---------|-------|----------|
| **Service** | Pod networking & load balancing | L4 (Transport) | Internal communication, simple external access |
| **Ingress** | HTTP/HTTPS routing | L7 (Application) | Web applications, path-based routing |
| **Gateway API** | Advanced traffic management | L4-L7 | Complex routing, multiple protocols |


