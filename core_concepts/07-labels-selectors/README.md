# Labels & Selectors - Resource Identification & Grouping

## What are Labels?

Labels are key-value pairs attached to Kubernetes objects for identification and organization. They enable:

- **Organization**: Group related resources
- **Selection**: Query resources by label
- **Service Routing**: Services use labels to find pods
- **Deployment Management**: Deployments use labels to manage pods

## Label Best Practices

Use consistent labeling strategies:
- `app`: Application name
- `version`: Application version
- `environment`: dev, staging, prod
- `component`: database, frontend, backend
- `tier`: frontend, backend, database
- `team`: team responsible for the resource

## What are Selectors?

Selectors are queries that use labels to find resources:
- **Equality-based**: `app=nginx`, `env!=production`
- **Set-based**: `app in (nginx, apache)`, `env notin (dev, test)`

## Files in this folder

1. `labeled-deployment.yaml` - Deployment with comprehensive labeling
2. `service-with-selectors.yaml` - Service using selectors to find pods

## Hands-on Practice

### Step 1: Deploy labeled application
```bash
kubectl apply -f labeled-deployment.yaml
kubectl get deployments --show-labels
kubectl get pods --show-labels
```

### Step 2: Create service with selectors
```bash
kubectl apply -f service-with-selectors.yaml
kubectl get services --show-labels
kubectl describe service frontend-service
```

### Step 3: Query resources by labels
```bash
# Get pods with specific label
kubectl get pods -l app=frontend
kubectl get pods -l component=web-server
kubectl get pods -l environment=production

# Multiple labels (AND)
kubectl get pods -l app=frontend,component=web-server

# Set-based selectors
kubectl get pods -l 'environment in (production,staging)'
kubectl get pods -l 'version notin (v1.0.0)'
```

### Step 4: Add labels to existing resources
```bash
kubectl get pods
kubectl label pod <pod-n> region=us-west-2
kubectl label pod <pod-n> maintainer=john.doe@company.com
kubectl get pods --show-labels
```

### Step 5: Update existing labels
```bash
kubectl label pod <pod-n> environment=staging --overwrite
kubectl get pods --show-labels
```

### Step 6: Remove labels
```bash
kubectl label pod <pod-n> region-
kubectl get pods --show-labels
```

### Step 7: Use labels for bulk operations
```bash
# Delete all pods with specific label
kubectl delete pods -l app=frontend

# Scale deployment selected by label
kubectl scale deployment -l app=frontend --replicas=2
```

## Advanced Selector Examples

```yaml
# In deployment or service specs
selector:
  matchLabels:           # Simple equality
    app: frontend
    tier: web
  matchExpressions:      # Advanced expressions
  - key: version
    operator: In
    values: ["v1.0", "v1.1"]
  - key: environment
    operator: NotIn
    values: ["development"]
```

### Selector Operators

- **In**: Value is in the list
- **NotIn**: Value is not in the list
- **Exists**: Key exists (regardless of value)
- **DoesNotExist**: Key does not exist

## Label Commands Cheat Sheet

```bash
# View labels
kubectl get <resource> --show-labels
kubectl describe <resource> <n>

# Add labels
kubectl label <resource> <n> <key>=<value>
kubectl label <resource> <n> <key>=<value> --overwrite

# Remove labels  
kubectl label <resource> <n> <key>-

# Select by labels
kubectl get <resource> -l <key>=<value>
kubectl get <resource> -l '<key> in (value1,value2)'
kubectl get <resource> -l <key>!=<value>

# Bulk operations
kubectl delete <resource> -l <key>=<value>
```

## Common Label Patterns

### Recommended Kubernetes Labels
```yaml
# Standard recommended labels
metadata:
  labels:
    app.kubernetes.io/n: nginx           # Application n
    app.kubernetes.io/instance: nginx-prod  # Instance n  
    app.kubernetes.io/version: "1.21"       # Version
    app.kubernetes.io/component: server     # Component
    app.kubernetes.io/part-of: web-app     # Larger application
    app.kubernetes.io/managed-by: helm     # Management tool
```

### Custom Application Labels
```yaml
# Application-specific labels
metadata:
  labels:
    app: web-app
    version: v1.2.3
    environment: production
    component: frontend
    tier: web
    team: platform
    release: stable
    region: us-west-2
```

## Label Lifecycle Management

### Development Workflow
```bash
# Label for canary deployment
kubectl label pods -l app=frontend canary=true

# Label for A/B testing
kubectl label deployment frontend-app track=stable
kubectl label deployment frontend-app-v2 track=canary

# Environment promotion
kubectl label deployment app environment=staging --overwrite
kubectl label deployment app environment=production --overwrite
```

## Service Discovery & Routing

```bash
# Service selects pods by labels
# Service: selector.app=frontend
# Pods: labels.app=frontend

# Multi-label selection (more specific)
# Service: selector.app=frontend,version=v1.2.3
# Only pods with both labels are selected
```

### DNS and Service Discovery
```bash
# Services create DNS entries based on selectors
curl frontend-service.default.svc.cluster.local

# Cross-namespace service access
curl api-service.backend.svc.cluster.local
```

## Troubleshooting with Labels

```bash
# Find orphaned pods (not managed by any deployment)
kubectl get pods -l '!deployment'

# Find all resources for an application
kubectl get all -l app=frontend

# Check which service endpoints match
kubectl get endpoints
kubectl describe endpoints <service-n>

# Debug service selection
kubectl get pods -l <service-selector-labels>
```

## Cleanup

```bash
# Delete all labeled resources
kubectl delete all -l app=frontend

# Delete specific resources by label
kubectl delete deployments -l component=web-server
kubectl delete services -l app=frontend

# Clean up all resources created
kubectl delete -f .
```

## Pro Tips

### Label Selection Strategies
```bash
# Combine multiple label queries
kubectl get pods -l app=frontend,environment!=development

# Use set-based selection for flexibility
kubectl get pods -l 'app in (frontend,backend)'

# Check resource relationships
kubectl get pods -l app=frontend -o wide
```

### Automation and Scripts
```bash
# Label all resources in a directory
find . -n "*.yaml" -exec kubectl label -f {} managed-by=gitops \;

# Bulk environment labeling
kubectl label deployment --all environment=development
```

## Real-World Scenarios

1. **Blue-Green Deployments**: Use version labels to switch traffic
2. **Feature Flags**: Label pods with feature toggles
3. **Cost Tracking**: Label resources by project/team for billing
4. **Monitoring**: Use labels for Prometheus metrics collection
5. **Backup Policies**: Label resources for backup schedules

## Next Steps

- **Annotations**: Learn about metadata that doesn't identify resources
- **Field Selectors**: Query resources by field values
- **Network Policies**: Use labels for network traffic control
- **Monitoring**: Integrate labels with monitoring solutions

