# ConfigMaps - Configuration Management

## What is a ConfigMap?

A ConfigMap stores configuration data as key-value pairs, enabling you to decouple configuration from your container images. It allows you to:

- **Separate Configuration**: Keep config separate from container images
- **Environment Variables**: Inject config as environment variables
- **Volume Mounts**: Mount configuration as files in containers
- **Reusability**: Share configuration across multiple pods
- **Dynamic Updates**: Change configuration without rebuilding images

## Why Use ConfigMaps?

Imagine you have an application that needs different settings for development, testing, and production environments. Without ConfigMaps, you would need to:

- **Build separate images** for each environment (time-consuming and error-prone)
- **Hardcode configuration** in your application (difficult to change)
- **Rebuild images** every time you need to change a simple setting

With ConfigMaps, you can:
- **Use the same container image** across all environments
- **Change configuration** without rebuilding or redeploying code
- **Store configuration separately** from your application code
- **Update settings** and restart pods to apply changes instantly

## ConfigMap Usage Patterns

| Pattern | Use Case | Example | Best For |
|---------|----------|---------|----------|
| **Environment Variables** | Simple key-value config | `DATABASE_URL`, `API_KEY` | Application settings |
| **Volume Mount** | Configuration files | `nginx.conf`, `application.yaml` | Complex configurations |
| **Command Arguments** | Runtime parameters | `--port=8080`, `--debug` | Application startup |
| **Init Containers** | Pre-startup config | Database migrations, setup scripts | Initialization tasks |

## Key Benefits

- **Immutable Infrastructure**: Configuration changes without rebuilding images
- **Environment Separation**: Different configs for dev/staging/production
- **Centralized Management**: Single source of truth for configuration
- **Easy Updates**: Modify configuration and restart pods
- **Version Control**: Track configuration changes like code

## Files in this folder

1. `configmap-literal.yaml` - Simple key-value pairs for environment variables
2. `configmap-file.yaml` - Configuration files (nginx.conf, app.properties)
3. `configmap-multiline.yaml` - Complex multi-line configuration files
4. `deployment-with-configmap.yaml` - Using ConfigMaps in deployments
5. `deployment-env-from.yaml` - Loading entire ConfigMap as environment variables
6. `deployment-volume-mount.yaml` - Mounting ConfigMap as files

## Hands-on Practice

### Prerequisites: Clean environment

```bash
# Ensure we have a clean namespace
kubectl get configmaps
kubectl get deployments
```

### Step 1: Create ConfigMap with literals

```bash
kubectl apply -f configmap-literal.yaml
kubectl get configmaps
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```

**Expected Output:**
```
NAME         DATA   AGE
app-config   4      10s
```

### Step 2: Create ConfigMap with file content

```bash
kubectl apply -f configmap-file.yaml
kubectl describe configmap nginx-config
```

**Inspect the file content:**
```bash
kubectl get configmap nginx-config -o jsonpath='{.data.nginx\.conf}' | head -10
```

### Step 3: Use ConfigMap in deployment

```bash
kubectl apply -f deployment-with-configmap.yaml
kubectl get pods
kubectl wait --for=condition=Ready pod -l app=config-demo --timeout=60s
```

### Step 4: Verify configuration is loaded

**Check environment variables:**
```bash
# Get pod name
POD_NAME=$(kubectl get pods -l app=config-demo -o jsonpath='{.items[0].metadata.name}')

# Check environment variables
kubectl exec $POD_NAME -- env | grep -E "(DATABASE_URL|DEBUG_MODE|API_VERSION)"

# Check all environment variables from ConfigMap
kubectl exec $POD_NAME -- env | sort
```

**Check mounted files:**
```bash
# List mounted configuration files
kubectl exec $POD_NAME -- ls -la /etc/config/

# View configuration file contents
kubectl exec $POD_NAME -- cat /etc/config/database_url
kubectl exec $POD_NAME -- cat /etc/config/api_version

# If nginx config is mounted
kubectl exec $POD_NAME -- cat /etc/nginx/nginx.conf | head -20
```

### Step 5: Update ConfigMap and see changes

```bash
# Update ConfigMap data
kubectl patch configmap app-config --patch '{"data":{"debug_mode":"false","new_setting":"updated_value"}}'

# Verify the change
kubectl get configmap app-config -o yaml | grep -A 10 "data:"

# Note: Pod needs restart to pick up environment variable changes
kubectl rollout restart deployment/config-demo-app
kubectl rollout status deployment/config-demo-app
```

**Check updated configuration:**
```bash
# Wait for new pod
kubectl wait --for=condition=Ready pod -l app=config-demo --timeout=60s

# Get new pod name and check environment
NEW_POD=$(kubectl get pods -l app=config-demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec $NEW_POD -- env | grep -E "(DEBUG_MODE|NEW_SETTING)"
```

### Step 6: Hot reload with volume mounts

```bash
kubectl apply -f deployment-volume-mount.yaml

# Update ConfigMap file content
kubectl patch configmap nginx-config --patch '{"data":{"nginx.conf":"# Updated configuration\nserver { listen 80; return 200 \"Hello Updated World\"; }"}}'

# Files are updated automatically (no restart needed for file mounts)
kubectl exec -l app=nginx-config-demo -- cat /etc/nginx/nginx.conf
```

## Creating ConfigMaps via kubectl

### From Literal Values
```bash
# Single key-value pair
kubectl create configmap my-config --from-literal=key1=value1

# Multiple key-value pairs
kubectl create configmap app-settings \
  --from-literal=database_url="postgres://localhost:5432/mydb" \
  --from-literal=debug_mode="true" \
  --from-literal=max_connections="100"
```

### From Files
```bash
# From single file
kubectl create configmap nginx-config --from-file=nginx.conf

# From file with custom key name
kubectl create configmap app-config --from-file=config=application.yaml

# From multiple files
kubectl create configmap multi-config \
  --from-file=app.properties \
  --from-file=database.conf \
  --from-file=logging.yaml
```

### From Directories
```bash
# Create ConfigMap from all files in directory
kubectl create configmap app-config --from-file=config/

# This creates keys based on filenames
# config/app.yaml -> key: app.yaml
# config/db.conf -> key: db.conf
```

### From Environment Files
```bash
# Create from .env file format
kubectl create configmap env-config --from-env-file=.env

# .env file format:
# KEY1=value1
# KEY2=value2
```

## Essential Commands Sheet

### ConfigMap Management
```bash
# Create ConfigMap
kubectl create configmap <name> --from-literal=<key>=<value>
kubectl apply -f configmap.yaml

# List ConfigMaps
kubectl get configmaps
kubectl get cm  # Short form

# Describe ConfigMap
kubectl describe configmap <name>

# Get ConfigMap as YAML
kubectl get configmap <name> -o yaml

# Edit ConfigMap
kubectl edit configmap <name>

# Delete ConfigMap
kubectl delete configmap <name>
```

### Using ConfigMaps in Deployments
```bash
# Update deployment to use ConfigMap
kubectl patch deployment <name> -p '{"spec":{"template":{"spec":{"containers":[{"name":"<container>","envFrom":[{"configMapRef":{"name":"<configmap>"}}]}]}}}}'

# Restart deployment after ConfigMap changes
kubectl rollout restart deployment/<name>

# Check rollout status
kubectl rollout status deployment/<name>
```

### Debugging and Inspection
```bash
# Check what environment variables are set
kubectl exec <pod> -- env

# Check mounted files
kubectl exec <pod> -- ls -la /path/to/mounted/config

# View file contents
kubectl exec <pod> -- cat /path/to/config/file

# Get ConfigMap data directly
kubectl get configmap <name> -o jsonpath='{.data}'
```

## ConfigMap Configuration Examples

### Basic Key-Value ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgres://db:5432/myapp"
  debug_mode: "true"
  max_connections: "100"
  api_version: "v2"
```

### File-Based ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        
        location /api {
            proxy_pass http://api-service:8080;
        }
    }
```

### Using ConfigMap as Environment Variables
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: nginx:1.20
        envFrom:
        - configMapRef:
            name: app-config
        # Or specific keys:
        env:
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database_url
```

### Using ConfigMap as Volume Mount
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: app-config
          mountPath: /etc/config
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
      - name: app-config
        configMap:
          name: app-config
```

## Common Troubleshooting Scenarios

### ConfigMap Not Found
```bash
# Check if ConfigMap exists
kubectl get configmaps

# Check in correct namespace
kubectl get configmaps -n <namespace>

# Describe deployment to see error
kubectl describe deployment <name>
```

### Environment Variables Not Loading
```bash
# Check pod environment
kubectl exec <pod> -- env | grep <EXPECTED_VAR>

# Check deployment configuration
kubectl get deployment <name> -o yaml | grep -A 20 envFrom

# Verify ConfigMap data
kubectl describe configmap <name>
```

### File Mount Issues
```bash
# Check if volume is mounted
kubectl exec <pod> -- df -h

# Check mount path contents
kubectl exec <pod> -- ls -la /path/to/mount

# Verify volume configuration
kubectl describe pod <pod> | grep -A 10 Volumes
```

### ConfigMap Updates Not Reflecting
```bash
# For environment variables - restart required
kubectl rollout restart deployment/<name>

# For file mounts - check if files updated
kubectl exec <pod> -- cat /path/to/config/file

# Check ConfigMap was actually updated
kubectl get configmap <name> -o yaml
```

## Best Practices

### Naming Conventions
```yaml
# Use descriptive, hierarchical names
metadata:
  name: frontend-app-config      # Good
  name: backend-database-config  # Good
  name: config                   # Too generic
```

### Data Organization
```yaml
# Group related settings
data:
  # Database settings
  db.host: "postgres.example.com"
  db.port: "5432"
  db.name: "myapp"
  
  # API settings
  api.timeout: "30"
  api.retries: "3"
  
  # Feature flags
  feature.new_ui: "true"
  feature.beta_api: "false"
```

### Environment-Specific ConfigMaps
```bash
# Different ConfigMaps per environment
kubectl create configmap app-config-dev --from-file=config/dev/
kubectl create configmap app-config-staging --from-file=config/staging/
kubectl create configmap app-config-prod --from-file=config/prod/
```

### Immutable ConfigMaps
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
immutable: true  # Cannot be updated
data:
  version: "1.0.0"
  config: "immutable configuration"
```

### Size Limitations
- ConfigMaps have a 1MB size limit
- For larger configurations, consider multiple ConfigMaps
- Use external configuration management for very large configs

## Security Considerations

### Sensitive Data
- **Never store secrets in ConfigMaps** - use Secrets instead
- ConfigMaps are stored in plain text in etcd
- Anyone with read access can see ConfigMap data

### RBAC (Role-Based Access Control)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
```

### What Should NOT Go in ConfigMaps
- Database passwords
- API keys
- TLS certificates
- OAuth tokens
- Any sensitive credentials

## ConfigMaps vs Secrets vs Environment Variables

| Method | Best For | Security | Complexity | Hot Reload |
|--------|----------|----------|------------|------------|
| **ConfigMaps** | Non-sensitive config | Plain text | Low | Yes (files only) |
| **Secrets** | Sensitive data | Base64 encoded | Medium | Yes (files only) |
| **Environment Variables** | Simple settings | Varies | Low | No |
| **External Config** | Complex/large configs | High | High | Yes |


## Quick Reference 

```bash
# Essential ConfigMap commands
kubectl create cm <name> --from-literal=key=value  # Create from literals
kubectl create cm <name> --from-file=config.yaml   # Create from file
kubectl get cm                                     # List ConfigMaps
kubectl describe cm <name>                         # View details
kubectl get cm <name> -o yaml                      # Get as YAML

# Using in deployments
kubectl set env deployment/<name> --from=configmap/<cm-name>
kubectl rollout restart deployment/<name>         # Apply env changes

# Debugging
kubectl exec <pod> -- env                         # Check environment
kubectl exec <pod> -- cat /path/to/config        # Check mounted files
```
---

> **Tip:** Use `kubectl create configmap --dry-run=client -o yaml` to generate ConfigMap YAML templates, then customize them for your needs!

> **Security Warning:** Never put passwords, API keys, or other sensitive data in ConfigMaps. Use Secrets instead!
