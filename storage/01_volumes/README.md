# Volumes - Pod Level Storage

## Learning Objectives
By the end of this section, you will understand:
- Different volume types and their use cases
- How containers share data within a pod
- Volume lifecycle and persistence characteristics
- When to use each volume type

## What's in this folder

| File                  | Description                      | Use Case                       |
|-----------------------|----------------------------------|--------------------------------|
| `emptydir-volume.yaml` | Temporary shared storage between containers | Shared cache, temporary processing |
| `hostpath-volume.yaml` | Mount host node filesystem       | Access host files, development |
| `configmap-volume.yaml` | Mount ConfigMap as files        | Configuration files, scripts   |
| `secret-volume.yaml`   | Mount Secret as files            | Certificates, private keys     |

## Quick Start

### Prerequisites
```bash
# Create required ConfigMaps and Secrets
kubectl create configmap app-settings \
  --from-literal=environment=production \
  --from-literal=debug=false

kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=secret123
```

### Try it out
```bash
# 1. Start with emptyDir (easiest)
kubectl apply -f emptydir-volume.yaml
kubectl logs shared-storage-pod -c writer
kubectl logs shared-storage-pod -c reader

# 2. Test shared storage
kubectl exec -it shared-storage-pod -c writer -- /bin/bash
echo "Hello from writer!" > /shared-data/message.txt
exit

kubectl exec -it shared-storage-pod -c reader -- cat /shared-data/message.txt
```

## Volume Types Explained

### 1. emptyDir
- **Purpose**: Temporary shared storage within a pod
- **Lifecycle**: Created when pod starts, deleted when pod is removed
- **Use Cases**:
  - Cache data between containers
  - Temporary processing space
  - Shared logs or files
- **Size**: Limited by node disk space
- **Persistence**: No (ephemeral)

### 2. hostPath
- **Purpose**: Access to host node filesystem
- **Lifecycle**: Exists on host, survives pod deletion
- **Use Cases**:
  - Development and testing
  - Access host system files
  - Node monitoring tools
- **Security**: High risk - use carefully
- **Persistence**: Yes (on that specific node)

### 3. configMap
- **Purpose**: Mount configuration as files
- **Lifecycle**: Tied to ConfigMap lifecycle
- **Use Cases**:
  - Application configuration files
  - Scripts and initialization files
  - Environment-specific settings
- **Updates**: Files update when ConfigMap changes
- **Persistence**: Yes (as long as ConfigMap exists)

### 4. secret
- **Purpose**: Mount sensitive data as files
- **Lifecycle**: Tied to Secret lifecycle
- **Use Cases**:
  - TLS certificates and keys
  - SSH keys
  - Database passwords in files
- **Security**: Files stored in tmpfs (memory)
- **Persistence**: Yes (as long as Secret exists)

## Troubleshooting

### Common Issues

#### Pod stuck in Pending
```bash
kubectl describe pod <pod-name>
# Look for: "MountVolume.SetUp failed"
```

#### Volume mount failed
```bash
# Check if ConfigMap/Secret exists
kubectl get configmap app-settings
kubectl get secret db-credentials

# Check hostPath permissions
kubectl exec -it hostpath-pod -- ls -la /host-data/
```

#### emptyDir size issues
```bash
# Check node disk space
kubectl describe nodes
# Look for: "DiskPressure" condition
```

## Key Takeaways

- **emptyDir**: Best for temporary shared storage between containers
- **hostPath**: Use only for development or special node access needs
- **configMap**: Perfect for application configuration files
- **secret**: Essential for certificates and sensitive files
- **Volume lifecycle**: Understand what happens when pods are deleted

## Volume Comparison Matrix

| Volume Type | Persistence | Scope | Security | Best Use Case |
|-------------|-------------|-------|----------|---------------|
| emptyDir    | Pod lifetime | Pod-local | Medium | Shared temporary storage |
| hostPath    | Node lifetime | Node-specific | Low | Development/testing |
| configMap   | Object lifetime | Cluster-wide | Medium | Configuration files |
| secret      | Object lifetime | Cluster-wide | High | Sensitive data |

## Best Practices

### Security Considerations
- Avoid `hostPath` volumes in production
- Use `secret` volumes for sensitive data
- Set appropriate file permissions
- Consider using `readOnly: true` when possible

### Performance Tips
- Use `emptyDir` with `medium: Memory` for high-speed access
- Monitor volume usage to prevent disk pressure
- Clean up unused ConfigMaps and Secrets

### Operational Guidelines
- Always document volume dependencies
- Test volume behavior during pod restarts
- Implement proper backup strategies for persistent data
- Use meaningful names for ConfigMaps and Secrets

## Example YAML Configurations

### emptyDir Volume Example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-storage-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date) >> /shared-data/log.txt; sleep 30; done"]
    volumeMounts:
    - name: shared-volume
      mountPath: /shared-data
  - name: reader
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do cat /shared-data/log.txt; sleep 60; done"]
    volumeMounts:
    - name: shared-volume
      mountPath: /shared-data
  volumes:
  - name: shared-volume
    emptyDir: {}
```

### hostPath Volume Example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "ls -la /host-data && sleep 3600"]
    volumeMounts:
    - name: host-volume
      mountPath: /host-data
  volumes:
  - name: host-volume
    hostPath:
      path: /tmp
      type: Directory
```

### ConfigMap Volume Example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "cat /etc/config/environment && sleep 3600"]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-settings
```

### Secret Volume Example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "cat /etc/secrets/username && sleep 3600"]
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
```

## Advanced Topics

### Volume Subpaths
Use subpaths to mount specific files from volumes:

```yaml
volumeMounts:
- name: config-volume
  mountPath: /app/config.yml
  subPath: config.yml
```

### Multiple Volume Mounts
A single container can mount multiple volumes:

```yaml
volumeMounts:
- name: config-volume
  mountPath: /etc/config
- name: secret-volume
  mountPath: /etc/secrets
- name: temp-volume
  mountPath: /tmp/shared
```

### Resource Limits for emptyDir
Control emptyDir size:

```yaml
volumes:
- name: temp-storage
  emptyDir:
    sizeLimit: 1Gi
```

## Conclusion

Understanding Kubernetes volumes is crucial for managing data in containerized applications. Each volume type serves specific purposes:

- Use **emptyDir** for temporary shared storage between containers in a pod
- Use **hostPath** sparingly and only for development or specialized node access
- Use **configMap** volumes to provide configuration files to your applications
- Use **secret** volumes for sensitive data like certificates and credentials

