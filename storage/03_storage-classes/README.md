# Storage Classes - Dynamic Storage Provisioning

## Learning Objectives
By the end of this section, you will understand:
- How dynamic storage provisioning works
- Different storage types and performance characteristics
- Cloud provider storage integration
- Storage class parameters and policies

## What's in this folder

| File | Description | Purpose |
|------|-------------|---------|
| `storage-class.yaml` | Custom storage class definitions | Learn storage class configuration |
| `dynamic-pvc.yaml` | PVCs using storage classes | Automatic PV creation |
| `deployment-dynamic-storage.yaml` | Application with dynamic storage | Production usage pattern |

## Quick Start

```bash
# 1. Check existing storage classes
kubectl get storageclass
kubectl get sc  # shorthand

# 2. Create custom storage classes
kubectl apply -f storage-class.yaml
kubectl get sc

# 3. Create PVC with automatic provisioning
kubectl apply -f dynamic-pvc.yaml
kubectl get pvc
kubectl get pv  # PV created automatically!

# 4. Use in application
kubectl apply -f deployment-dynamic-storage.yaml
kubectl get pods
```

## How Dynamic Provisioning Works

### Traditional (Static) Provisioning
1. Admin creates PV manually
2. User creates PVC
3. Kubernetes binds PVC to existing PV
4. Pod uses PVC

### Dynamic Provisioning
1. Admin creates StorageClass
2. User creates PVC referencing StorageClass
3. Kubernetes automatically creates PV
4. PVC binds to new PV
5. Pod uses PVC

### Benefits of Dynamic Provisioning

 No manual PV creation needed  
 On-demand storage allocation  
 Different storage tiers (SSD, HDD, etc.)  
 Cloud provider integration  
 Automatic cleanup when PVC deleted  

## Storage Class Components

### Basic Structure
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs  # Who creates the storage
parameters:
  type: gp3                         # Storage-specific config
  iops: "3000"
allowVolumeExpansion: true          # Can grow after creation
reclaimPolicy: Delete               # What happens when PVC deleted
volumeBindingMode: Immediate        # When to create PV
```

## Cloud Provider Examples

### AWS EBS (Elastic Block Store)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### Google Cloud Persistent Disk
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcp-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
allowVolumeExpansion: true
```

### Azure Disk
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  cachingmode: ReadOnly
allowVolumeExpansion: true
```

## Hands on 

### Explore Default Storage Classes
```bash
# Check what storage classes exist
kubectl get storageclass
kubectl get sc -o wide

# Describe default storage class
kubectl describe storageclass

# Check which is default
kubectl get sc -o yaml | grep "is-default-class"

# Test default behavior
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: default-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc
kubectl get pv
```

### Custom Storage Classes
```bash
# Create custom storage classes
kubectl apply -f storage-class.yaml

# List all storage classes
kubectl get sc
kubectl describe sc fast-ssd
kubectl describe sc slow-hdd

# Create PVCs using different classes
kubectl apply -f dynamic-pvc.yaml

# Check automatic PV creation
kubectl get pvc
kubectl get pv
kubectl describe pv | grep "StorageClass"
```

### Application with Multiple Storage Types
```bash
# Deploy application with different storage
kubectl apply -f deployment-dynamic-storage.yaml

# Check created resources
kubectl get deployments
kubectl get pods
kubectl get pvc
kubectl get pv

# Test storage access
kubectl exec -it deployment/web-app-dynamic -- /bin/bash
echo "Fast storage test" > /data/speed-test.txt
echo "Backup data" > /backup/backup-test.txt
df -h /data /backup
exit

# Check storage classes used
kubectl get pv -o custom-columns=NAME:.metadata.name,STORAGECLASS:.spec.storageClassName,SIZE:.spec.capacity.storage
```

### Storage Expansion
```bash
# Check current PVC size
kubectl get pvc dynamic-storage-claim

# Expand storage (if allowVolumeExpansion: true)
kubectl patch pvc dynamic-storage-claim -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'

# Check expansion status
kubectl describe pvc dynamic-storage-claim
kubectl get events --sort-by=.metadata.creationTimestamp

# Verify in pod
kubectl exec -it deployment/web-app-dynamic -- df -h /data
```

## Volume Binding Modes

### Immediate
- **When**: PV created immediately when PVC is created
- **Use case**: Single zone clusters
- **Risk**: Pod might be scheduled on wrong node

### WaitForFirstConsumer
- **When**: PV created when first pod using PVC is scheduled
- **Use case**: Multi-zone clusters, topology-aware storage
- **Benefit**: PV created in correct zone for the pod

### Example
```yaml
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
    - us-west-2a
    - us-west-2b
```

## Storage Performance Tiers

### Performance Comparison

| Storage Type | IOPS | Throughput | Cost | Use Case |
|--------------|------|------------|------|----------|
| NVMe SSD | 50,000+ | 2GB/s+ | $$$ | Databases, high-performance apps |
| SSD | 3,000-16,000 | 250MB/s+ | $$ | General applications |
| HDD | 100-500 | 100MB/s | $ | Backups, archives |
| Network | Varies | Varies | $ | Shared storage, multi-attach |

### Choosing Storage Types
```yaml
# High performance database
storageClassName: "nvme-ssd"

# Web application
storageClassName: "standard-ssd"

# Log archival
storageClassName: "slow-hdd"

# Shared file system
storageClassName: "nfs-shared"
```

## Troubleshooting

### PVC Stuck in Pending
```bash
# Check storage class exists
kubectl get sc <storageclass-name>

# Check provisioner is available
kubectl describe sc <storageclass-name>

# Check for provisioner pods
kubectl get pods -n kube-system | grep -i provision

# Check events
kubectl describe pvc <pvc-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Volume Expansion Issues
```bash
# Check if expansion is allowed
kubectl describe sc <storageclass-name> | grep allowVolumeExpansion

# Check expansion status
kubectl describe pvc <pvc-name>

# Check if pod restart is needed
kubectl get events | grep VolumeExpansion
```

### Storage Class Not Working
```bash
# Check provisioner logs
kubectl logs -n kube-system <provisioner-pod-name>

# Verify cloud provider permissions
# (AWS: EBS permissions, GCP: Compute permissions, etc.)

# Check node capacity
kubectl describe nodes | grep -A 5 "Capacity"
```

## Best Practices

### For Administrators
- Create appropriate storage classes for different use cases
- Set reasonable defaults for common workloads
- Monitor storage costs and usage
- Enable volume expansion when possible
- Use topology-aware storage in multi-zone clusters

### For Developers
- Choose appropriate storage class for performance needs
- Request reasonable storage sizes (not too big to waste money)
- Consider data access patterns (read-heavy vs write-heavy)
- Plan for growth with expandable storage
- Clean up unused PVCs to avoid costs

## Cost Optimization

### Storage Cost Tips
```bash
# Monitor PVC usage
kubectl get pvc -o custom-columns=NAME:.metadata.name,SIZE:.spec.resources.requests.storage,STORAGECLASS:.spec.storageClassName

# Find unused PVCs
kubectl get pvc --all-namespaces | grep -v Bound

# Check PV reclaim policy
kubectl get pv -o custom-columns=NAME:.metadata.name,RECLAIM:.spec.persistentVolumeReclaimPolicy
```

### Cost-Effective Storage Strategy
- Use HDD for cold data (backups, archives)
- Use SSD for active data (databases, applications)
- Enable auto-deletion with Delete reclaim policy
- Monitor and right-size storage requests
- Use storage lifecycle policies in cloud providers

## Key Takeaways

 **Storage Classes enable dynamic provisioning** - no manual PV creation  
 **Different classes for different needs** - performance vs cost  
 **Cloud provider integration** - leverages cloud storage services  
 **Volume expansion** - grow storage after creation  
 **Topology awareness** - storage in the right location  

## Next Steps

Ready for stateful applications? Move to ../04-statefulsets/ to learn about:
- Managing databases and stateful workloads
- Ordered deployment and scaling
- Stable network identities
- Individual storage per replica
