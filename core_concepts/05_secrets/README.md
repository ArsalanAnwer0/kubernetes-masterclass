# Secrets - Secure Sensitive Data

## What is a Secret?

A Secret stores sensitive information like passwords, tokens, and keys. Similar to ConfigMaps but:

- **Base64 Encoded**: Data is encoded (not encrypted)
- **Restricted Access**: Can be restricted via RBAC
- **Memory Storage**: Stored in tmpfs (memory) not disk
- **Automatic**: Some secrets created automatically (service accounts)

## Secret Types

- **Opaque**: Arbitrary user-defined data (default)
- **kubernetes.io/tls**: TLS certificates
- **kubernetes.io/dockerconfigjson**: Docker registry credentials
- **kubernetes.io/service-account-token**: Service account tokens

## Files in this folder

1. `opaque-secret.yaml` - Basic secret with credentials
2. `tls-secret.yaml` - TLS certificate secret
3. `deployment-with-secrets.yaml` - Using secrets in deployments

## Hands-on Practice

### Step 1: Create basic secret
```bash
kubectl apply -f opaque-secret.yaml
kubectl get secrets
kubectl describe secret app-secret
```

### Step 2: View secret content (be careful!)
```bash
kubectl get secret app-secret -o yaml
# Decode base64 values
echo "YWRtaW4=" | base64 --decode  # admin
echo "cGFzc3dvcmQxMjM=" | base64 --decode  # password123
```

### Step 3: Use secret in deployment
```bash
kubectl apply -f deployment-with-secrets.yaml
kubectl get pods
```

### Step 4: Verify secrets are loaded
```bash
# Check environment variables (be careful in production!)
kubectl exec -it deployment/secure-app -- printenv | grep -E "(DB_USERNAME|DB_PASSWORD)"

# Check mounted secret files
kubectl exec -it deployment/secure-app -- ls -la /etc/secrets/
kubectl exec -it deployment/secure-app -- cat /etc/secrets/username
```

### Step 5: Create TLS secret
```bash
# First generate a self-signed certificate (for demo)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=example.com/O=example.com"

# Create TLS secret from files
kubectl create secret tls my-tls-secret --cert=tls.crt --key=tls.key

# Or apply the YAML version
kubectl apply -f tls-secret.yaml
```

## Creating Secrets via kubectl

```bash
# From literals
kubectl create secret generic my-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123

# From files
kubectl create secret generic ssh-secret --from-file=ssh-key=/path/to/ssh/key

# Docker registry secret
kubectl create secret docker-registry my-registry-secret \
  --docker-server=<server> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email>

# TLS secret
kubectl create secret tls my-tls-secret --cert=tls.crt --key=tls.key
```

## Security Best Practices

 **Important Security Notes:**
- Secrets are base64 encoded, NOT encrypted
- Use external secret management for production (Vault, AWS Secrets Manager)
- Limit access with RBAC
- Rotate secrets regularly
- Never commit secrets to Git

## Cleanup

```bash
# Delete all resources
kubectl delete -f .

# Delete specific secret
kubectl delete secret app-secret

# Delete TLS secret
kubectl delete secret tls-secret
```

## Additional Commands

```bash
# List all secrets
kubectl get secrets

# Get secret details
kubectl describe secret <secret-name>

# Edit secret (careful!)
kubectl edit secret <secret-name>

# Create secret from command line
kubectl create secret generic <name> --from-literal=key=value

# View secret in different formats
kubectl get secret <name> -o yaml
kubectl get secret <name> -o jsonpath='{.data}'
```

## Common Use Cases

1. **Database Credentials**: Store database usernames and passwords
2. **API Keys**: Store third-party service API keys
3. **TLS Certificates**: Store SSL/TLS certificates for HTTPS
4. **Docker Registry**: Store credentials for private container registries
5. **SSH Keys**: Store SSH keys for secure connections

## Production Considerations

- **External Secret Management**: Use HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault
- **Secret Rotation**: Implement automatic secret rotation
- **Least Privilege**: Grant minimal necessary permissions
- **Audit Logging**: Monitor secret access and usage
- **Backup Strategy**: Ensure secrets are included in backup procedures
