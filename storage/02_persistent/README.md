# Persistent Volumes - Cluster-Level Storage

## Learning Objectives
By the end of this section, you will understand:
- The difference between Volumes and Persistent Volumes
- How PVs and PVCs work together
- Storage lifecycle independent of pods
- Different access modes and reclaim policies

## What's in this folder

| File | Description | Purpose |
|------|-------------|---------|
| `persistent-volume.yaml` | Manual PV creation | Learn PV structure and configuration |
| `persistent-volume-claim.yaml` | Storage request | How applications request storage |
| `pod-with-pvc.yaml` | Pod using PVC | Basic PVC usage pattern |
| `deployment-with-pvc.yaml` | Deployment with shared storage | Production usage pattern |

## Quick Start

```bash
# 1. Create Persistent Volume
kubectl apply -f persistent-volume.yaml
kubectl get pv

# 2. Claim the storage
kubectl apply -f persistent-volume-claim.yaml
kubectl get pvc
kubectl get pv  # Status should be "Bound"

# 3. Use storage in a pod
kubectl apply -f pod-with-pvc.yaml
kubectl exec -it pvc-pod -- echo "Hello PV!" > /data/test.txt

# 4. Test persistence
kubectl delete pod pvc-pod
kubectl apply -f pod-with-pvc.yaml
kubectl exec -it pvc-pod -- cat /data/test.txt  # Data persists!
```

## Core Concepts

### Persistent Volume (PV)
- **What**: Cluster-level storage resource
- **Who creates**: Usually cluster administrator
- **Lifecycle**: Independent of any pod
- **Analogy**: Like a "storage disk" in the cluster

### Persistent Volume Claim (PVC)
- **What**: Request for storage by a user/pod
- **Who creates**: Application developer
- **Purpose**: "I need 5GB of storage"
- **Analogy**: Like a "storage reservation"

### The PV/PVC Dance
1. **Admin creates PV** (available storage)
   ↓
2. **User creates PVC** (storage request)
   ↓
3. **Kubernetes binds PVC to suitable PV**
   ↓
4. **Pod uses PVC to access storage**

## Access Modes Explained

| Mode | Code | Description | Use Case |
|------|------|-------------|----------|
| ReadWriteOnce | RWO | Single node read-write | Databases, file systems |
| ReadOnlyMany | ROX | Multiple nodes read-only | Static content, configuration |
| ReadWriteMany | RWX | Multiple nodes read-write | Shared file systems |
| ReadWriteOncePod | RWOP | Single pod read-write | High security applications |

### Access Mode Examples

```yaml
# RWO - Database storage
accessModes: ["ReadWriteOnce"]

# ROX - Static website content
accessModes: ["ReadOnlyMany"]

# RWX - Shared logs directory
accessModes: ["ReadWriteMany"]
```

## Reclaim Policies

| Policy | What happens when PVC is deleted |
|--------|----------------------------------|
| Retain | PV remains, data preserved, manual cleanup |
| Delete | PV and underlying storage deleted automatically |
| Recycle | Deprecated - data wiped, PV reused |

## Hands on 

### PV/PVC Lifecycle

```bash
# 1. Check initial state
kubectl get pv
kubectl get pvc

# 2. Create PV
kubectl apply -f persistent-volume.yaml
kubectl get pv my-pv
kubectl describe pv my-pv
# Status: Available

# 3. Create PVC
kubectl apply -f persistent-volume-claim.yaml
kubectl get pvc my-pvc
kubectl get pv my-pv
# Status: Bound

# 4. Check binding details
kubectl describe pvc my-pvc
kubectl describe pv my-pv
```

### Data Persistence Test

```bash
# 1. Deploy pod with PVC
kubectl apply -f pod-with-pvc.yaml

# 2. Create test data
kubectl exec -it pvc-pod -- /bin/bash
echo "Persistent data test" > /data/important.txt
echo "$(date): First write" >> /data/timeline.txt
ls -la /data/
exit

# 3. Simulate pod failure
kubectl delete pod pvc-pod

# 4. Recreate pod
kubectl apply -f pod-with-pvc.yaml

# 5. Verify data persistence
kubectl exec -it pvc-pod -- cat /data/important.txt
kubectl exec -it pvc-pod -- cat /data/timeline.txt

# 6. Add more data
kubectl exec -it pvc-pod -- /bin/bash
echo "$(date): After pod recreation" >> /data/timeline.txt
cat /data/timeline.txt
exit
```

### Deployment with Shared Storage

```bash
# 1. Clean up pod
kubectl delete -f pod-with-pvc.yaml

# 2. Deploy with Deployment
kubectl apply -f deployment-with-pvc.yaml
kubectl get deployments
kubectl get pods

# 3. Test shared access (Note: RWO limitation)
kubectl scale deployment pvc-deployment --replicas=2
kubectl get pods
# One pod will be Pending due to RWO access mode

# 4. Check events
kubectl describe pod <pending-pod-name>
# Should see: "Multi-Attach error"

# 5. Scale back
kubectl scale deployment pvc-deployment --replicas=1
```

### Storage Inspection

```bash
# Inspect storage usage
kubectl exec -it deployment/pvc-deployment -- df -h /data

# Check PVC details
kubectl describe pvc my-pvc

# Check PV details
kubectl describe pv my-pv

# View all storage resources
kubectl get pv,pvc
```

## Troubleshooting

### Common Issues

#### PVC stuck in Pending
```bash
kubectl describe pvc <pvc-name>
# Look for: "waiting for a volume to be created"

# Check if suitable PV exists
kubectl get pv
# Look for Available PVs with matching size/access mode
```

#### Pod stuck in Pending due to storage
```bash
kubectl describe pod <pod-name>
# Look for: "pod has unbound immediate PersistentVolumeClaims"

# Check PVC status
kubectl get pvc
# Should be "Bound"
```

#### Multi-Attach errors
```bash
# Check access modes
kubectl describe pv <pv-name>
# RWO can only be attached to one node at a time

# Check which pods are using the PVC
kubectl get pods -o wide | grep <node-name>
```

## PV Status States

| State | Description | Next Action |
|-------|-------------|-------------|
| Available | Ready to be claimed | Create PVC |
| Bound | Claimed by a PVC | Use in pod |
| Released | PVC deleted, but not reclaimed | Admin cleanup |
| Failed | Automatic reclamation failed | Manual intervention |

## Best Practices

### For Administrators
- Create PVs in advance for predictable workloads
- Use appropriate reclaim policies (Retain for production)
- Monitor storage usage and capacity
- Label PVs for better organization

### For Developers
- Request appropriate storage sizes (not too big/small)
- Choose correct access modes for your use case
- Handle storage failures gracefully in applications
- Clean up PVCs when no longer needed

## Storage Sizing Guidelines

| Application Type | Typical Size | Access Mode | Notes |
|------------------|--------------|-------------|-------|
| Database | 10GB - 1TB+ | RWO | Size based on data growth |
| Log aggregation | 50GB - 500GB | RWX | Multiple pods write logs |
| Static content | 1GB - 50GB | ROX | Shared across many pods |
| Cache | 1GB - 10GB | RWO | Fast storage preferred |

## Example YAML Configurations

### Persistent Volume Example
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: local
    environment: demo
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /tmp/pv-data
    type: DirectoryOrCreate
```

### Persistent Volume Claim Example
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  labels:
    app: storage-demo
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      type: local
```

### Pod with PVC Example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
  labels:
    app: storage-demo
spec:
  containers:
  - name: storage-container
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date) >> /data/log.txt; sleep 30; done"]
    volumeMounts:
    - name: storage-volume
      mountPath: /data
  volumes:
  - name: storage-volume
    persistentVolumeClaim:
      claimName: my-pvc
  restartPolicy: Never
```

### Deployment with PVC Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pvc-deployment
  labels:
    app: storage-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: storage-demo
  template:
    metadata:
      labels:
        app: storage-demo
    spec:
      containers:
      - name: app-container
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: storage-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: storage-volume
        persistentVolumeClaim:
          claimName: my-pvc
```

## Advanced Topics

### PV Selectors
Use selectors to bind PVCs to specific PVs:

```yaml
# In PVC
selector:
  matchLabels:
    environment: production
    tier: ssd

# In PV
metadata:
  labels:
    environment: production
    tier: ssd
```

### Volume Expansion
Enable volume expansion for supported storage classes:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-storage
provisioner: kubernetes.io/aws-ebs
allowVolumeExpansion: true
```

### PV Node Affinity
Restrict PV to specific nodes:

```yaml
nodeAffinity:
  required:
    nodeSelectorTerms:
    - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - node01
```

## Monitoring and Observability

### Key Metrics to Monitor
- **PV utilization**: How much storage is actually used
- **PVC binding time**: How long it takes to bind claims
- **Storage I/O performance**: Read/write speeds and IOPS
- **Available storage capacity**: Remaining space in PVs

### Useful Commands for Monitoring
```bash
# Monitor PV/PVC status
kubectl get pv,pvc -o wide

# Check storage events
kubectl get events --sort-by=.metadata.creationTimestamp

# Monitor storage usage
kubectl top pods --containers

# Check node storage capacity
kubectl describe nodes | grep -A 5 "Allocated resources"
```

## Security Considerations

### Access Control
- Use RBAC to control who can create PVs and PVCs
- Implement pod security policies for volume types
- Consider using service accounts with limited permissions

### Data Protection
- Encrypt sensitive data at rest
- Use appropriate reclaim policies for data retention
- Implement backup strategies for persistent data
- Consider using sealed secrets for sensitive configurations

## Performance Optimization

### Storage Performance Tips
- Choose appropriate storage classes for performance requirements
- Use SSD-backed storage for high I/O workloads
- Consider local storage for ultra-low latency requirements
- Monitor and tune storage I/O patterns

### Resource Management
- Set appropriate resource requests and limits
- Use storage quotas to prevent resource exhaustion
- Implement proper cleanup procedures for unused PVCs
- Monitor storage growth patterns

## Key Takeaways

 **PVs are cluster resources** - exist independently of pods  
 **PVCs are storage requests** - like "reservations" for storage  
 **Access modes matter** - choose based on sharing needs  
 **Reclaim policies control** what happens to data when PVC is deleted  
 **Data persists** across pod deletion and rescheduling  

## Comparison: Volumes vs Persistent Volumes

| Aspect | Pod Volumes | Persistent Volumes |
|--------|-------------|-------------------|
| **Lifecycle** | Tied to pod | Independent of pods |
| **Scope** | Pod-local | Cluster-wide |
| **Data Persistence** | Pod lifetime only | Survives pod deletion |
| **Management** | Defined in pod spec | Separate resource |
| **Sharing** | Within pod only | Across pods (with RWX) |
| **Use Cases** | Temporary data | Persistent application data |

