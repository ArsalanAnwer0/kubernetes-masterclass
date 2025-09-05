# StatefulSets - Managing Stateful Applications

## Learning Objectives
By the end of this section, you will understand:
- When and why to use StatefulSets vs Deployments
- How StatefulSets provide stable identity and storage
- Database cluster deployment and management
- Scaling stateful applications safely

## What's in this folder

| File | Description | Complexity | Use Case |
|------|-------------|------------|----------|
| `headless-service.yaml` | Services for StatefulSets | Basic | Network identity |
| `simple-statefulset.yaml` | Basic web StatefulSet | Intermediate | Learning StatefulSet features |
| `mysql-statefulset.yaml` | MySQL database cluster | Advanced | Production database |
| `mongodb-statefulset.yaml` | MongoDB replica set | Expert | Distributed database |

## Quick Start

```bash
# 1. Create headless service
kubectl apply -f headless-service.yaml

# 2. Deploy simple StatefulSet
kubectl apply -f simple-statefulset.yaml
kubectl get statefulsets
kubectl get pods

# 3. Watch ordered creation
kubectl get pods -w
# Pods created in order: web-0, web-1, web-2

# 4. Test stable identity
kubectl run test --image=curlimages/curl -i --tty --rm -- sh
curl web-0.nginx-headless.default.svc.cluster.local
exit
```

## StatefulSet vs Deployment

### When to Use Each

| Feature | Deployment | StatefulSet |
|---------|------------|-------------|
| Pod Identity | Random names | Stable ordered names (web-0, web-1) |
| Network Identity | Dynamic IPs | Stable DNS names |
| Storage | Shared or none | Individual PVC per pod |
| Scaling | Parallel | Sequential (ordered) |
| Updates | Rolling (any order) | Rolling (reverse order) |
| Use Cases | Stateless web apps | Databases, queues, distributed systems |

### StatefulSet Guarantees
- Stable, unique network identifiers
- Stable, persistent storage
- Ordered, graceful deployment and scaling
- Ordered, automated rolling updates

## StatefulSet Components

### Headless Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None  # This makes it headless
  selector:
    app: nginx-stateful
  ports:
  - port: 80
```
**Purpose**: Provides stable DNS names for each pod

### StatefulSet
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx-headless"  # Links to headless service
  replicas: 3
  selector:
    matchLabels:
      app: nginx-stateful
  template:
    # Pod template
  volumeClaimTemplates:
    # Storage template - creates PVC for each pod
```

## Hands on 

### Basic StatefulSet Features
```bash
# 1. Deploy headless service and StatefulSet
kubectl apply -f headless-service.yaml
kubectl apply -f simple-statefulset.yaml

# 2. Watch ordered creation
kubectl get pods -w
# Notice: web-0 created first, then web-1, then web-2

# 3. Check stable DNS names
kubectl run dns-test --image=busybox:1.35 -i --tty --rm -- sh
# Inside the pod:
nslookup nginx-headless
nslookup web-0.nginx-headless.default.svc.cluster.local
nslookup web-1.nginx-headless.default.svc.cluster.local
exit

# 4. Check individual storage
kubectl get pvc
# Should see: www-web-0, www-web-1, www-web-2

# 5. Test individual storage
kubectl exec -it web-0 -- /bin/bash
echo "I am web-0" > /usr/share/nginx/html/identity.txt
exit

kubectl exec -it web-1 -- /bin/bash
echo "I am web-1" > /usr/share/nginx/html/identity.txt
exit

# 6. Verify separate storage
kubectl exec -it web-0 -- cat /usr/share/nginx/html/identity.txt
kubectl exec -it web-1 -- cat /usr/share/nginx/html/identity.txt
```

### Scaling Behavior
```bash
# 1. Scale up (ordered creation)
kubectl scale statefulset web --replicas=5
kubectl get pods -w
# Watch: web-3 created, then web-4

# 2. Scale down (reverse order deletion)
kubectl scale statefulset web --replicas=2
kubectl get pods -w
# Watch: web-4 deleted first, then web-3

# 3. Check storage persistence
kubectl get pvc
# PVCs for deleted pods still exist (web-3, web-4)

# 4. Scale back up
kubectl scale statefulset web --replicas=4
kubectl get pods
# web-3 comes back and reattaches to its original PVC

# 5. Verify data persistence
kubectl exec -it web-0 -- cat /usr/share/nginx/html/identity.txt
# Should still say "I am web-0"
```

### MySQL Database Cluster
```bash
# 1. Create MySQL secret
kubectl create secret generic mysql-secret \
  --from-literal=mysql-root-password=rootpass123 \
  --from-literal=mysql-password=userpass123

# 2. Deploy MySQL StatefulSet
kubectl apply -f mysql-statefulset.yaml

# 3. Wait for MySQL to be ready
kubectl get pods -w
kubectl wait --for=condition=ready pod mysql-0 --timeout=300s

# 4. Connect to MySQL
kubectl exec -it mysql-0 -- mysql -u root -p
# Password: rootpass123
# SQL Commands:
SHOW DATABASES;
CREATE DATABASE testapp;
USE testapp;
CREATE TABLE users (id INT PRIMARY KEY, name VARCHAR(50), email VARCHAR(100));
INSERT INTO users VALUES (1, 'Alice', 'alice@example.com');
INSERT INTO users VALUES (2, 'Bob', 'bob@example.com');
SELECT * FROM users;
EXIT;

# 5. Test data persistence through pod restart
kubectl delete pod mysql-0
kubectl wait --for=condition=ready pod mysql-0 --timeout=300s

kubectl exec -it mysql-0 -- mysql -u root -p testapp
# Password: rootpass123
SELECT * FROM users;
# Data should still be there!
EXIT;

# 6. Scale MySQL (read replicas)
kubectl scale statefulset mysql --replicas=2
kubectl wait --for=condition=ready pod mysql-1 --timeout=300s

# 7. Connect to replica
kubectl exec -it mysql-1 -- mysql -u root -p testapp
# Password: rootpass123
SELECT * FROM users;
# Data replicated to second instance
EXIT;
```

### MongoDB Replica Set
```bash
# 1. Create MongoDB secret
kubectl create secret generic mongodb-secret \
  --from-literal=mongodb-root-password=mongopass123

# 2. Deploy MongoDB StatefulSet
kubectl apply -f mongodb-statefulset.yaml

# 3. Wait for all MongoDB pods
kubectl wait --for=condition=ready pod mongodb-0 --timeout=300s
kubectl wait --for=condition=ready pod mongodb-1 --timeout=300s
kubectl wait --for=condition=ready pod mongodb-2 --timeout=300s

# 4. Initialize replica set (connect to primary)
kubectl exec -it mongodb-0 -- mongosh admin -u admin -p mongopass123

# In MongoDB shell:
rs.initiate({
  _id: "rs0",
  members: [
    {_id: 0, host: "mongodb-0.mongodb-headless.default.svc.cluster.local:27017"},
    {_id: 1, host: "mongodb-1.mongodb-headless.default.svc.cluster.local:27017"},
    {_id: 2, host: "mongodb-2.mongodb-headless.default.svc.cluster.local:27017"}
  ]
});

# Wait for replica set to initialize
rs.status();

# Create test data
use testdb;
db.users.insertOne({name: "Alice", age: 30, city: "New York"});
db.users.insertOne({name: "Bob", age: 25, city: "San Francisco"});
db.users.find();
exit

# 5. Test replica set functionality
kubectl exec -it mongodb-1 -- mongosh admin -u admin -p mongopass123
use testdb;
db.users.find();  # Data should be replicated
rs.status();      # Check replica set status
exit

# 6. Test failover simulation
kubectl delete pod mongodb-0  # Delete primary
kubectl get pods -w          # Watch new primary election

# Connect to any running instance
kubectl exec -it mongodb-1 -- mongosh admin -u admin -p mongopass123
rs.status();  # Check new primary
use testdb;
db.users.find();  # Data still accessible
exit
```

## StatefulSet Management

### Rolling Updates
```bash
# Update StatefulSet image
kubectl set image statefulset/web nginx=nginx:1.22

# Watch rolling update (reverse order)
kubectl rollout status statefulset/web
kubectl get pods -w
# Updates: web-2 first, then web-1, then web-0

# Check rollout history
kubectl rollout history statefulset/web

# Rollback if needed
kubectl rollout undo statefulset/web
```

### Manual Pod Management
```bash
# Delete specific pod
kubectl delete pod web-1
kubectl get pods
# StatefulSet recreates web-1 with same identity

# Force delete stuck pod
kubectl delete pod web-1 --force --grace-period=0

# Get specific pod logs
kubectl logs web-0
kubectl logs mysql-1 -c mysql
```

### Storage Management
```bash
# View all PVCs for StatefulSet
kubectl get pvc -l app=nginx-stateful

# Check storage usage
kubectl exec -it web-0 -- df -h /usr/share/nginx/html

# Backup pod data
kubectl exec -it web-0 -- tar czf /tmp/backup.tar.gz /usr/share/nginx/html/
kubectl cp web-0:/tmp/backup.tar.gz ./web-0-backup.tar.gz

# Restore to another pod
kubectl cp ./web-0-backup.tar.gz web-1:/tmp/
kubectl exec -it web-1 -- tar xzf /tmp/backup.tar.gz -C /
```

## Troubleshooting

### Pod Stuck in Pending
```bash
# Check pod events
kubectl describe pod <pod-name>

# Common issues:
# 1. PVC creation failed
kubectl describe pvc <pvc-name>

# 2. Storage class issues
kubectl get storageclass

# 3. Resource constraints
kubectl describe nodes | grep -A 5 "Allocated resources"
```

### StatefulSet Not Scaling
```bash
# Check StatefulSet status
kubectl describe statefulset <statefulset-name>

# Check for failed pods
kubectl get pods | grep -v Running

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Database Connection Issues
```bash
# Check service endpoints
kubectl get endpoints <service-name>

# Test DNS resolution
kubectl run test --image=busybox:1.35 -i --tty --rm -- nslookup <service-name>

# Check database logs
kubectl logs <pod-name> -c <container-name>

# Test connection from another pod
kubectl run mysql-client --image=mysql:8.0 -i --tty --rm -- mysql -h mysql-0.mysql-headless -u root -p
```

## StatefulSet Patterns

### Database Patterns
```yaml
# Primary-Secondary (MySQL)
- Primary: mysql-0 (read-write)
- Secondaries: mysql-1, mysql-2 (read-only)

# Replica Set (MongoDB)
- All replicas can handle reads
- One primary for writes
- Automatic failover
```

### Scaling Strategies
```bash
# Safe database scaling
# 1. Scale up gradually
kubectl scale statefulset mysql --replicas=3

# 2. Verify health before adding more
kubectl exec -it mysql-2 -- mysql -u root -p -e "SHOW SLAVE STATUS\G"

# 3. Scale down from highest ordinal
kubectl scale statefulset mysql --replicas=2
# mysql-2 deleted first (highest ordinal)
```

## Best Practices

### Design Principles
 Use headless services for stable DNS names  
 Implement health checks (liveness/readiness probes)  
 Plan storage carefully (size, performance, backup)  
 Handle initialization (database setup, clustering)  
 Monitor resource usage (CPU, memory, disk)  

### Operational Practices
 Scale gradually and verify health  
 Backup before major changes  
 Test disaster recovery procedures  
 Monitor cluster health continuously  
 Use appropriate resource limits  

### Security Considerations
 Use secrets for credentials  
 Enable authentication in databases  
 Restrict network access with Network Policies  
 Regular security updates  
 Audit database access  

## Production Readiness Checklist

### Before Production
- [ ] Backup strategy implemented
- [ ] Monitoring and alerting setup
- [ ] Resource limits properly configured
- [ ] Health checks thoroughly tested
- [ ] Disaster recovery procedures documented
- [ ] Security hardening completed
- [ ] Performance testing completed
- [ ] Scaling procedures tested

### Monitoring Metrics
```bash
# Pod status and restarts
kubectl get pods -o wide

# Resource usage
kubectl top pods

# Storage usage
kubectl exec -it <pod> -- df -h

# Database-specific metrics
kubectl exec -it mysql-0 -- mysql -u root -p -e "SHOW STATUS LIKE 'Threads_connected'"
kubectl exec -it mongodb-0 -- mongosh --eval "db.serverStatus().connections"
```

## Key Takeaways

 **StatefulSets provide stable identity** - predictable names and DNS  
 **Individual storage per pod** - each replica has its own data  
 **Ordered operations** - creation, scaling, updates happen in sequence  
 **Perfect for databases** - MySQL, MongoDB, PostgreSQL, etc.  
 **Requires careful planning** - storage, networking, initialization  

## Storage Sizing Guidelines

| Application Type | Typical Size | Access Mode | Notes |
|------------------|--------------|-------------|-------|
| Database | 10GB - 1TB+ | RWO | Size based on data growth |
| Log aggregation | 50GB - 500GB | RWX | Multiple pods write logs |
| Static content | 1GB - 50GB | ROX | Shared across many pods |
| Cache | 1GB - 10GB | RWO | Fast storage preferred |

---

