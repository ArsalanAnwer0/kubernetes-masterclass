# Ingress - HTTP/HTTPS Routing

## Learning Objectives
By the end of this section, you will understand:
- How Ingress controllers work
- Path-based and host-based routing
- SSL/TLS termination
- Advanced ingress features and annotations

## What's in this folder

| File | Description | Routing Type | Features |
|------|-------------|--------------|----------|
| `ingress-controller.yaml` | NGINX Ingress Controller | N/A | Controller setup |
| `basic-ingress.yaml` | Simple ingress rule | Single service | Basic routing |
| `path-based-ingress.yaml` | Path-based routing | Multiple paths | `/api`, `/web` |
| `host-based-ingress.yaml` | Host-based routing | Multiple hosts | `api.example.com` |
| `tls-ingress.yaml` | HTTPS with TLS | SSL termination | Certificate management |
| `ingress-with-annotations.yaml` | Advanced features | Custom behavior | Rewrites, redirects |

## Quick Start

```bash
# 1. Install NGINX Ingress Controller
kubectl apply -f ingress-controller.yaml

# 2. Deploy sample applications
kubectl create deployment web-app --image=nginx:1.21
kubectl create deployment api-app --image=httpd:2.4
kubectl expose deployment web-app --port=80
kubectl expose deployment api-app --port=80

# 3. Create basic ingress
kubectl apply -f basic-ingress.yaml

# 4. Test ingress
kubectl get ingress
kubectl describe ingress basic-ingress
```

## Ingress Concepts

### What is Ingress?
- **Layer 7 Load Balancer**: HTTP/HTTPS routing
- **Single Entry Point**: One external IP for multiple services
- **Advanced Routing**: Path, host, header-based routing
- **SSL Termination**: Handle HTTPS certificates
- **Cost Effective**: One load balancer for many services

### Ingress vs Service

| Feature | Service (LoadBalancer) | Ingress |
|---------|------------------------|---------|
| Layer | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
| Routing | Basic port-based | Advanced path/host-based |
| SSL | Pass-through only | SSL termination |
| Cost | One LB per service | One LB for all services |
| Features | Limited | Rich (redirects, rewrites) |

### Architecture
```
Internet → LoadBalancer → Ingress Controller → Services → Pods
              ↓
        [Ingress Rules]
         - Path routing
         - Host routing  
         - SSL termination
```

## Hands-on Exercises

### Install Ingress Controller
```bash
# Install NGINX Ingress Controller
kubectl apply -f ingress-controller.yaml

# Wait for controller to be ready
kubectl get pods -n ingress-nginx
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=300s

# Check ingress class
kubectl get ingressclass
kubectl describe ingressclass nginx
```

### Basic Ingress
```bash
# Create sample applications
kubectl create deployment web-app --image=nginx:1.21
kubectl create deployment api-app --image=httpd:2.4

# Expose as services
kubectl expose deployment web-app --port=80 --name=web-service
kubectl expose deployment api-app --port=80 --name=api-service

# Create basic ingress
kubectl apply -f basic-ingress.yaml
kubectl get ingress basic-ingress
kubectl describe ingress basic-ingress

# Test ingress (get external IP)
kubectl get svc -n ingress-nginx ingress-nginx-controller
# Test: curl http://<external-ip>/
```

### Path based Routing
```bash
# Create path-based ingress
kubectl apply -f path-based-ingress.yaml
kubectl get ingress path-based-ingress
kubectl describe ingress path-based-ingress

# Test different paths
curl http://<ingress-ip>/web
curl http://<ingress-ip>/api
curl http://<ingress-ip>/api/info

# Check routing in ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller | tail -20
```

### Host based Routing
```bash
# Create host-based ingress
kubectl apply -f host-based-ingress.yaml
kubectl get ingress host-based-ingress

# Test with host headers (if no real DNS)
curl -H "Host: web.example.com" http://<ingress-ip>/
curl -H "Host: api.example.com" http://<ingress-ip>/

# Add to /etc/hosts for real testing (optional)
# echo "<ingress-ip> web.example.com api.example.com" >> /etc/hosts
# curl http://web.example.com/
# curl http://api.example.com/
```

### TLS/SSL Termination
```bash
# Create TLS certificate (self-signed for demo)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=secure.example.com/O=secure.example.com"

# Create TLS secret
kubectl create secret tls secure-tls \
  --key tls.key --cert tls.crt

# Create TLS ingress
kubectl apply -f tls-ingress.yaml
kubectl get ingress tls-ingress
kubectl describe ingress tls-ingress

# Test HTTPS
curl -k -H "Host: secure.example.com" https://<ingress-ip>/
```

### Advanced Annotations
```bash
# Create ingress with annotations
kubectl apply -f ingress-with-annotations.yaml
kubectl describe ingress advanced-ingress

# Test rewrite functionality
curl -H "Host: app.example.com" http://<ingress-ip>/old-path
# Should redirect or rewrite to new path

# Test rate limiting
for i in {1..20}; do curl -H "Host: app.example.com" http://<ingress-ip>/; done
```

## Ingress Controller Types

### Popular Ingress Controllers

| Controller | Provider | Features | Use Case |
|------------|----------|----------|----------|
| NGINX | NGINX Inc/Community | High performance, feature-rich | General purpose |
| Traefik | Traefik Labs | Auto-discovery, modern | Microservices |
| HAProxy | HAProxy Technologies | Enterprise features | High availability |
| Istio Gateway | Istio | Service mesh integration | Advanced traffic management |
| AWS ALB | Amazon | Native AWS integration | AWS environments |

### NGINX Ingress Annotations
```yaml
metadata:
  annotations:
    # Basic annotations
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/rate-limit: "10"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    
    # Authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    
    # Custom configuration
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Custom-Header: value";
```

## Example Ingress Configurations

### Basic Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: basic-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Path based Routing
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

### Host based Routing
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

### TLS Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.example.com
    secretName: secure-tls
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

## Troubleshooting

### Ingress Not Working
```bash
# Check ingress controller is running
kubectl get pods -n ingress-nginx
kubectl describe pod -n ingress-nginx <controller-pod>

# Check ingress resource
kubectl get ingress
kubectl describe ingress <ingress-name>

# Check service endpoints
kubectl get endpoints <service-name>
kubectl describe endpoints <service-name>

# Check ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

### SSL/TLS Issues
```bash
# Check TLS secret
kubectl get secret <tls-secret-name>
kubectl describe secret <tls-secret-name>

# Verify certificate
kubectl get secret <tls-secret-name> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout

# Check ingress TLS configuration
kubectl describe ingress <ingress-name>
```

### DNS and Host Issues
```bash
# Check ingress external IP
kubectl get svc -n ingress-nginx

# Test with curl and Host header
curl -v -H "Host: example.com" http://<ingress-ip>/

# Check DNS resolution (if using real DNS)
nslookup example.com
dig example.com
```

## Advanced Ingress Patterns

### Blue Green Deployments
```yaml
# Route traffic between versions
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-blue    # 90% traffic
            port:
              number: 80
      - path: /green
        pathType: Prefix
        backend:
          service:
            name: app-green   # 10% traffic
            port:
              number: 80
```

### Canary Deployments
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% to canary
```

### A/B Testing
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Version"
    nginx.ingress.kubernetes.io/canary-by-header-value: "beta"
```

## Best Practices

### Design Principles
 Use meaningful hostnames and paths  
 Implement SSL/TLS for all external traffic  
 Use annotations for advanced features  
 Monitor ingress performance  
 Plan for high availability  

### Security
 Enable SSL redirect for HTTPS  
 Use rate limiting to prevent abuse  
 Implement authentication where needed  
 Regular certificate renewal  
 Monitor access logs  

### Performance
 Configure appropriate timeouts  
 Use connection pooling  
 Enable compression  
 Monitor response times  
 Scale ingress controller as needed  

## Common Annotations

### NGINX Ingress Annotations Reference

| Annotation | Purpose | Example Value |
|------------|---------|---------------|
| `nginx.ingress.kubernetes.io/rewrite-target` | URL rewriting | `/` |
| `nginx.ingress.kubernetes.io/ssl-redirect` | Force HTTPS | `"true"` |
| `nginx.ingress.kubernetes.io/rate-limit` | Rate limiting | `"10"` |
| `nginx.ingress.kubernetes.io/auth-type` | Authentication | `basic` |
| `nginx.ingress.kubernetes.io/cors-allow-origin` | CORS policy | `"*"` |
| `nginx.ingress.kubernetes.io/proxy-body-size` | Upload size limit | `"50m"` |

### Advanced Configuration
```yaml
metadata:
  annotations:
    # Timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    
    # Load balancing
    nginx.ingress.kubernetes.io/upstream-hash-by: "$request_uri"
    nginx.ingress.kubernetes.io/load-balance: "round_robin"
    
    # Custom headers
    nginx.ingress.kubernetes.io/server-snippet: |
      add_header X-Frame-Options SAMEORIGIN;
      add_header X-Content-Type-Options nosniff;
```

## Monitoring and Observability

### Key Metrics to Monitor
- **Request rate**: Number of requests per second
- **Response time**: Latency of requests
- **Error rate**: Percentage of failed requests
- **Backend health**: Status of upstream services

### Monitoring Commands
```bash
# Check ingress controller metrics
kubectl get --raw /metrics | grep nginx

# Monitor ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller -f

# Check ingress status
kubectl get ingress -o wide

# Verify backend services
kubectl get endpoints
```

## Key Takeaways

 **Ingress provides Layer 7 routing** - HTTP/HTTPS features  
 **Single entry point** - Cost-effective external access  
 **Flexible routing** - Path, host, header-based  
 **SSL termination** - Centralized certificate management  
 **Rich annotations** - Advanced traffic control features  

---
