# Kubernetes Core Concepts

This folder contains examples and implementations of the fundamental Kubernetes concepts that form the foundation of container orchestration. These core concepts are essential for understanding how Kubernetes manages, scales, and orchestrates containerized applications.

## Overview

The core concepts covered in this section include:

1. **Pods** - The smallest deployable units in Kubernetes
2. **Deployments** - Declarative updates for Pods and ReplicaSets  
3. **Services** - Network abstraction for accessing Pods
4. **ConfigMaps** - Configuration data management
5. **Secrets** - Sensitive data management
6. **Namespaces** - Virtual clusters for resource isolation
7. **Labels & Selectors** - Resource identification and grouping

## Folder Structure

```
core-concepts/
├── 01-pods/
│   ├── basic-pod.yaml
│   ├── multi-container-pod.yaml
│   └── pod-with-resources.yaml
├── 02-deployments/
│   ├── simple-deployment.yaml
│   ├── deployment-with-strategy.yaml
│   └── deployment-with-probes.yaml
├── 03-services/
│   ├── clusterip-service.yaml
│   ├── nodeport-service.yaml
│   ├── loadbalancer-service.yaml
│   └── headless-service.yaml
├── 04-configmaps/
│   ├── configmap-literal.yaml
│   ├── configmap-file.yaml
│   └── deployment-with-configmap.yaml
├── 05-secrets/
│   ├── opaque-secret.yaml
│   ├── tls-secret.yaml
│   └── deployment-with-secrets.yaml
├── 06-namespaces/
│   ├── namespaces.yaml
│   ├── resource-quota.yaml
│   └── deployment-in-namespace.yaml
└── 07-labels-selectors/
    ├── labeled-deployment.yaml
    └── service-with-selectors.yaml
```

## Core Concepts Explained

### 1. Pods
The smallest and simplest unit in the Kubernetes object model. A Pod represents a single instance of a running process in your cluster.

**Examples included:**
- **Basic Pod**: Simple single-container pod running Nginx
- **Multi-Container Pod**: Pod with multiple containers sharing storage and network
- **Pod with Resources**: Pod with resource limits, requests, and health probes

### 2. Deployments
Provides declarative updates for Pods and ReplicaSets. Deployments manage the desired state of your application.

**Examples included:**
- **Simple Deployment**: Basic deployment with 3 replicas
- **Deployment with Strategy**: Rolling update configuration with surge and unavailability settings
- **Deployment with Probes**: Includes liveness, readiness, and startup probes

### 3. Services
An abstract way to expose an application running on a set of Pods as a network service.

**Examples included:**
- **ClusterIP**: Internal cluster communication (default)
- **NodePort**: Exposes service on each Node's IP at a static port
- **LoadBalancer**: Exposes service externally using cloud provider's load balancer
- **Headless Service**: Direct pod-to-pod communication without load balancing

### 4. ConfigMaps
Allows you to decouple configuration artifacts from image content to keep containerized applications portable.

**Examples included:**
- **Literal ConfigMap**: Key-value pairs for application configuration
- **File ConfigMap**: Complete configuration files (nginx.conf, HTML files)
- **Deployment with ConfigMap**: How to consume ConfigMaps as environment variables and volume mounts

### 5. Secrets
Objects that store sensitive data such as passwords, OAuth tokens, and SSH keys.

**Examples included:**
- **Opaque Secret**: Generic secrets with base64 encoded data
- **TLS Secret**: Certificate and private key for HTTPS
- **Deployment with Secrets**: Consuming secrets as environment variables and volume mounts

### 6. Namespaces
Provides a mechanism for isolating groups of resources within a single cluster.

**Examples included:**
- **Multiple Namespaces**: Development, production, and testing environments
- **Resource Quotas**: Limiting resource usage per namespace
- **Namespace Deployments**: Same application deployed across different namespaces

### 7. Labels & Selectors
Labels are key/value pairs attached to objects. Selectors use labels to identify and group resources.

**Examples included:**
- **Labeled Deployment**: Comprehensive labeling strategy for applications
- **Service Selectors**: How services use selectors to target specific pods

## Usage Instructions

### Applying Resources

To apply any of these examples to your Kubernetes cluster:

```bash
# Apply a specific file
kubectl apply -f 01-pods/basic-pod.yaml

# Apply all files in a directory
kubectl apply -f 01-pods/

# Apply all core concepts
kubectl apply -f .
```

### Verifying Deployments

```bash
# Check pods
kubectl get pods

# Check deployments
kubectl get deployments

# Check services
kubectl get services

# Check all resources
kubectl get all
```

### Cleaning Up

```bash
# Delete specific resources
kubectl delete -f 01-pods/basic-pod.yaml

# Delete all resources in a directory
kubectl delete -f 01-pods/

# Delete all core concept resources
kubectl delete -f .
```

## Key Features Demonstrated

- **Resource Management**: CPU and memory requests/limits
- **Health Checks**: Liveness, readiness, and startup probes  
- **Rolling Updates**: Zero-downtime deployment strategies
- **Configuration Management**: Separating config from code
- **Security**: Handling sensitive data with secrets
- **Network Policies**: Different service exposure methods
- **Multi-tenancy**: Namespace isolation and resource quotas
- **Organization**: Effective labeling and selection strategies

## Learning Path

For beginners, it's recommended to explore these concepts in order:

1. Start with **Pods** to understand the basic unit of deployment
2. Move to **Deployments** to learn about managing multiple pod instances
3. Explore **Services** to understand networking and service discovery
4. Learn **ConfigMaps** and **Secrets** for configuration management
5. Understand **Namespaces** for resource isolation
6. Master **Labels & Selectors** for resource organization

## Next Steps

After mastering these core concepts, you can explore advanced Kubernetes topics such as:
- Persistent Volumes and Storage
- Security and RBAC
- Networking and Ingress
- Monitoring and Logging
- Custom Resources and Operators

---

**Note**: These examples are designed for learning purposes. For production use, consider additional configurations like security contexts, network policies, and resource governance.
