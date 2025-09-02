# Phase 1: Core Concepts & Architecture

Welcome to Phase 1 of your Kubernetes mastery journey! In this phase, you'll learn the fundamental building blocks of Kubernetes.

## What You'll Learn

1. **Pods** - The smallest deployable units
2. **Deployments** - Managing and scaling applications
3. **Services** - Network access to your applications
4. **ConfigMaps** - Configuration management
5. **Secrets** - Secure sensitive data
6. **Namespaces** - Resource organization and isolation
7. **Labels & Selectors** - Resource identification and grouping

## Prerequisites

- Kubernetes cluster (minikube, kind, or cloud provider)
- kubectl CLI tool installed and configured
- Basic understanding of containers and Docker

## Learning Path

Work through each folder in numerical order:

1. Start with `01-pods/` to understand the basic unit
2. Move to `02-deployments/` to learn scaling and management
3. Continue through each folder sequentially

## Commands You'll Use

```bash
# Apply configurations
kubectl apply -f <filename>.yaml

# View resources
kubectl get <resource-type>
kubectl describe <resource-type> <name>

# Clean up
kubectl delete -f <filename>.yaml
