# Deployments - Managing Applications at Scale

## What is a Deployment?

A Deployment provides declarative updates for Pods and ReplicaSets. It's the standard way to run applications in Kubernetes because it provides:

- **Scaling**: Run multiple replicas of your application
- **Rolling Updates**: Update applications without downtime
- **Rollbacks**: Revert to previous versions
- **Self-healing**: Replace failed pods automatically

## Key Concepts

- **Desired State**: You declare what you want, Kubernetes makes it happen
- **ReplicaSet**: Manages the desired number of pod replicas
- **Rolling Updates**: Gradually replace old pods with new ones
- **Rollback**: Return to a previous version if something goes wrong

## Files in this folder

1. `simple-deployment.yaml` - Basic deployment with 3 replicas
2. `deployment-with-strategy.yaml` - Custom update strategy
3. `deployment-with-probes.yaml` - Health checks included


## Hands-on Practice

### Step 1: Create your first deployment

```bash
kubectl apply -f simple-deployment.yaml
kubectl get deployments
kubectl get pods
kubectl get replicasets
```

**Expected Output:**
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           30s
```

### Step 2: Scale the deployment

```bash
kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods
```

**Watch the magic happen:**
```bash
kubectl get pods -w
# Press Ctrl+C to stop watching
```

### Step 3: Update the deployment

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.22
kubectl rollout status deployment/nginx-deployment
```

**Monitor the rolling update:**
```bash
kubectl get pods -w
```

### Step 4: Check rollout history

```bash
kubectl rollout history deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment --revision=2
```

### Step 5: Rollback if needed

```bash
kubectl rollout undo deployment/nginx-deployment
kubectl rollout status deployment/nginx-deployment
```

**Rollback to specific revision:**
```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=1
```

### Step 6: Test self-healing

```bash
kubectl get pods
kubectl delete pod <pod-name>
kubectl get pods
# Watch how Kubernetes creates a new pod automatically!
```

**Pro tip:** Use labels to delete multiple pods:
```bash
kubectl delete pods -l app=nginx
```

## Essential Commands Cheat Sheet

### Deployment Management
```bash
# Create/Update deployment
kubectl apply -f deployment.yaml

# Get deployments
kubectl get deployments
kubectl get deploy  # Short form

# Detailed information
kubectl describe deployment <name>

# Scale deployment
kubectl scale deployment <name> --replicas=<number>

# Delete deployment
kubectl delete deployment <name>
```

### Rolling Updates
```bash
# Update image
kubectl set image deployment/<name> <container>=<new-image>

# Check rollout status
kubectl rollout status deployment/<name>

# View rollout history
kubectl rollout history deployment/<name>

# Rollback to previous version
kubectl rollout undo deployment/<name>

# Rollback to specific revision
kubectl rollout undo deployment/<name> --to-revision=<number>

# Pause/Resume rollout
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>
```

### Troubleshooting
```bash
# View events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check pod logs
kubectl logs deployment/<name>

# Describe problematic pods
kubectl describe pod <pod-name>

# Get deployment YAML
kubectl get deployment <name> -o yaml
```

## Best Practices

### Resource Management
```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

### Health Checks
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Update Strategy
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

## Common Troubleshooting Scenarios

### Deployment Stuck in Progressing State
```bash
# Check events
kubectl describe deployment <name>

# Check pod status
kubectl get pods -l app=<your-app>

# Check for resource constraints
kubectl describe nodes
```

### Image Pull Errors
```bash
# Check pod events
kubectl describe pod <pod-name>

# Verify image exists
kubectl run test-pod --image=<your-image> --dry-run=client
```

### Rolling Update Issues
```bash
# Check rollout status
kubectl rollout status deployment/<name>

# If stuck, rollback immediately
kubectl rollout undo deployment/<name>
```

