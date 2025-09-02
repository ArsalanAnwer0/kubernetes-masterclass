# Pods - The Foundation of Kubernetes

## What is a Pod?

A Pod is the smallest deployable unit in Kubernetes. It represents a group of one or more containers that:
- Share the same network (IP address and ports)
- Share storage volumes
- Are scheduled together on the same node
- Live and die together

## Key Concepts

- **Single Container Pod**: Most common - one application per pod
- **Multi-Container Pod**: Containers that need to work closely together
- **Ephemeral**: Pods are disposable and replaceable
- **IP Address**: Each pod gets its own IP address

## Files in this folder

1. `basic-pod.yaml` - Simple single container pod
2. `multi-container-pod.yaml` - Pod with multiple containers
3. `pod-with-resources.yaml` - Pod with resource limits

## Hands-on Practice

### Step 1: Create a basic pod
```bash
kubectl apply -f basic-pod.yaml
kubectl get pods
kubectl describe pod nginx-basic

### Step 2: Access the pod

```bash
kubectl port-forward pod/nginx-basic 8080:80
# Visit http://localhost:8080
```
### Step 3: Check logs

```bash
kubectl logs nginx-basic
```

### Step 4: Execute into the pod

```bash
kubectl exec -it nginx-basic -- /bin/bash
# Try: ls, ps aux, exit
```

### Step 5: Clean up

```bash
kubectl delete -f basic-pod.yaml
```
