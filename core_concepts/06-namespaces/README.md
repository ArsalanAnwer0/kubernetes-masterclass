# Namespaces - Resource Organization & Isolation

## What is a Namespace?

A Namespace provides a way to organize and isolate resources within a cluster. Think of it as a virtual cluster within your physical cluster.

## Benefits

- **Organization**: Group related resources together
- **Isolation**: Separate environments (dev, staging, prod)
- **Resource Quotas**: Limit resource usage per namespace
- **RBAC**: Control access per namespace
- **DNS**: Services get namespace-specific DNS names

## Default Namespaces

- **default**: Where resources go if no namespace specified
- **kube-system**: Kubernetes system components
- **kube-public**: Publicly readable by all users
- **kube-node-lease**: Node lease objects for node heartbeats

## Files in this folder

1. `namespaces.yaml` - Create development and production namespaces
2. `resource-quota.yaml` - Set resource limits for namespaces
3. `deployment-in-namespace.yaml` - Deploy applications to specific namespaces

## Hands-on Practice

### Step 1: Create namespaces
```bash
kubectl apply -f namespaces.yaml
kubectl get namespaces
kubectl get ns  # shorthand
```

### Step 2: Deploy to specific namespace
```bash
kubectl apply -f deployment-in-namespace.yaml
kubectl get pods -n development
kubectl get services -n development
```

### Step 3: Apply resource quotas
```bash
kubectl apply -f resource-quota.yaml
kubectl get resourcequota -n development
kubectl describe resourcequota dev-quota -n development
```

### Step 4: Set default namespace context
```bash
# Set default namespace to development
kubectl config set-context --current --namespace=development

# Now commands work in development namespace by default
kubectl get pods  # shows pods in development namespace

# Reset to default
kubectl config set-context --current --namespace=default
```

### Step 5: Cross-namespace communication
```bash
# Deploy to both namespaces
kubectl apply -f deployment-in-namespace.yaml

# Test cross-namespace service access
kubectl run test-pod --image=curlimages/curl -i --tty --rm -- sh
# Inside the pod:
curl web-app-service.development.svc.cluster.local
exit
```

## Important Commands

```bash
# Namespace management
kubectl create namespace <name>
kubectl get namespaces
kubectl delete namespace <name>  # Deletes ALL resources in namespace!

# Work with specific namespaces
kubectl get pods -n <namespace>
kubectl apply -f file.yaml -n <namespace>
kubectl delete -f file.yaml -n <namespace>

# Get resources from all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A  # shorthand

# Set default namespace
kubectl config set-context --current --namespace=<namespace>
kubectl config view --minify | grep namespace  # check current namespace
```

## Service Discovery with Namespaces

```bash
# Same namespace
curl <service-name>

# Different namespace
curl <service-name>.<namespace>

# Full DNS name
curl <service-name>.<namespace>.svc.cluster.local
```

## Resource Quotas

You can limit resources per namespace:

- **CPU and memory limits**
- **Number of pods, services, etc.**
- **Storage requests**
- **Number of secrets, configmaps**

### Example Resource Quota Configuration
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "10"
    persistentvolumeclaims: "5"
    services: "5"
    secrets: "10"
    configmaps: "10"
```

## Monitoring Resource Usage

```bash
# Check quota usage
kubectl describe quota -n <namespace>

# Monitor resource consumption
kubectl top pods -n <namespace>
kubectl top nodes

# View all quotas across namespaces
kubectl get resourcequota --all-namespaces
```

## Cleanup

```bash
# Delete specific resources
kubectl delete -f deployment-in-namespace.yaml
kubectl delete -f resource-quota.yaml

# Delete entire namespace (CAUTION: This deletes ALL resources!)
kubectl delete namespace development
kubectl delete namespace production
kubectl delete namespace testing

# Delete all namespaces created
kubectl delete -f namespaces.yaml
```

## Best Practices

### Naming Conventions
- Use descriptive names: `frontend-prod`, `backend-dev`
- Include environment: `app-staging`, `app-production`
- Team-based: `team-alpha`, `team-beta`

### Environment Separation
```bash
# Typical environment setup
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production
```

### Resource Management
- Always set resource quotas for non-production environments
- Monitor resource usage regularly
- Use different quotas for different environments (prod gets more resources)

## Security Considerations

- **RBAC**: Use Role-Based Access Control per namespace
- **Network Policies**: Isolate network traffic between namespaces
- **Pod Security**: Apply different security policies per namespace
- **Service Accounts**: Use namespace-specific service accounts

## Advanced Usage

### Labels for Namespace Management
```bash
# Add labels to namespaces
kubectl label namespace development environment=dev team=backend

# Select namespaces by labels
kubectl get namespaces -l environment=dev
```

### Cross-Namespace Resource References
```yaml
# ConfigMap from different namespace
envFrom:
- configMapRef:
    name: shared-config
    namespace: shared-resources
```

## Common Use Cases

1. **Multi-tenancy**: Separate teams or applications
2. **Environment Separation**: dev/staging/prod isolation
3. **Resource Governance**: Control resource consumption
4. **Security Boundaries**: Implement access controls
5. **Cost Management**: Track resource usage by team/project


## Next Steps

After mastering namespaces, explore:
- **Network Policies**: Control traffic between namespaces
- **RBAC**: Implement role-based access control
- **Pod Security Standards**: Apply security policies per namespace
- **Resource Management**: Advanced quota and limit ranges
