# Understanding Kubernetes Secrets

## What Is a Secret?

A Secret in Kubernetes is an object that stores sensitive information, such as passwords, OAuth tokens, SSH keys, and other confidential data. Secrets are similar to ConfigMaps but are specifically designed for storing sensitive information that should not be exposed in plain text.

Think of a Secret as a secure vault for your application's sensitive data. Unlike ConfigMaps, Secrets are encoded (base64) by default and can be encrypted at rest when using appropriate storage backends.

## Key Characteristics of Secrets

- **Sensitive Data Storage**: Store confidential information like passwords, tokens, and keys
- **Base64 Encoding**: Data is encoded (but not encrypted) by default
- **Volume or Environment Variable Mounting**: Can be exposed to Pods as files or environment variables
- **Namespace Scoped**: Secrets are bound to a specific namespace
- **Size Limitation**: Limited to 1MB in size
- **Encryption Options**: Can be encrypted at rest when configured with appropriate storage encryption
- **Access Control**: Can be restricted using RBAC (Role-Based Access Control)

## How Secrets Work

1. You create a Secret containing your sensitive data
2. You reference the Secret in your Pod specification
3. The kubelet (node agent) makes the Secret data available to the container
4. Your application reads the sensitive data from environment variables or mounted files
5. Kubernetes ensures the Secret data is only distributed to nodes that run Pods that need the Secret

## Types of Secrets

Kubernetes supports several built-in types of Secrets:

- **Opaque** (default): Arbitrary user-defined data
- **kubernetes.io/service-account-token**: Service account tokens
- **kubernetes.io/dockercfg**: Serialized ~/.dockercfg file
- **kubernetes.io/dockerconfigjson**: Serialized ~/.docker/config.json file
- **kubernetes.io/basic-auth**: Credentials for basic authentication
- **kubernetes.io/ssh-auth**: Data for SSH authentication
- **kubernetes.io/tls**: Data for TLS client or server
- **bootstrap.kubernetes.io/token**: Bootstrap token data

## Creating Secrets

There are several ways to create Secrets:

### 1. From Literal Values

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=s3cr3t
```

### 2. From Files

```bash
kubectl create secret generic tls-certs \
  --from-file=cert.pem \
  --from-file=key.pem
```

### 3. Using YAML Definition

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  # Base64 encoded values
  username: YWRtaW4=  # admin
  password: czNjcjN0  # s3cr3t
```

Or using the `stringData` field for unencoded values:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  # Plain text values (will be encoded automatically)
  username: admin
  password: s3cr3t
```

### 4. Creating Specific Secret Types

#### TLS Secret

```bash
kubectl create secret tls tls-secret \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem
```

#### Docker Registry Secret

```bash
kubectl create secret docker-registry regcred \
  --docker-server=<your-registry-server> \
  --docker-username=<your-username> \
  --docker-password=<your-password> \
  --docker-email=<your-email>
```

## Using Secrets in Pods

There are two main ways to use Secrets in Pods:

### 1. As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-example
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    # Set a single environment variable from a Secret key
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password

    # Set all keys from a Secret as environment variables
    envFrom:
    - secretRef:
        name: app-secrets
```

### 2. As Mounted Volumes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-example
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      # Optional: specify permissions for the mounted files
      defaultMode: 0400  # Read-only for owner
```

You can also mount specific keys to specific paths:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-example
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      items:
      - key: username
        path: db/username.txt
      - key: password
        path: db/password.txt
      defaultMode: 0400
```

## Secret Updates and Pod Behavior

How Secret updates affect running Pods depends on how the Secret is consumed:

### Environment Variables

- Environment variables from Secrets are set only when the container starts
- Changes to the Secret are not reflected in running containers
- Pods must be restarted to pick up new values

### Volume Mounts

- Files mounted from Secrets are updated automatically when the Secret changes
- The update is not immediate; it can take up to the sync period (typically a few minutes)
- The application must be designed to reload its configuration when files change

## Working with Secrets

### Viewing Secrets

```bash
# List all Secrets in the current namespace
kubectl get secrets

# Get detailed information about a Secret
kubectl describe secret db-credentials

# View the data in a Secret (will show base64 encoded values)
kubectl get secret db-credentials -o yaml

# Decode a specific Secret value
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 --decode
```

### Editing Secrets

```bash
# Edit a Secret directly
kubectl edit secret db-credentials

# Update a Secret from a file
kubectl create secret generic db-credentials \
  --from-file=username.txt \
  --from-file=password.txt \
  --dry-run=client -o yaml | kubectl apply -f -

# Patch a Secret
kubectl patch secret db-credentials -p '{"data":{"password":"bmV3cGFzc3dvcmQ="}}'
```

### Deleting Secrets

```bash
# Delete a Secret
kubectl delete secret db-credentials
```

## Secret Security Considerations

### Default Security Limitations

By default, Kubernetes Secrets have some security limitations:

1. **Not Encrypted by Default**: Secrets are stored as base64-encoded strings in etcd (unless encryption at rest is configured)
2. **Accessible to Cluster Admins**: Anyone with access to the API server or etcd can view Secrets
3. **Visible in Pod Specifications**: Secret names are visible in Pod specifications
4. **Cached on Nodes**: Secrets are cached on nodes where Pods use them

### Enhancing Secret Security

To enhance the security of your Secrets:

1. **Enable Encryption at Rest**: Configure the API server to encrypt Secret data in etcd
2. **Use RBAC**: Restrict access to Secrets using Role-Based Access Control
3. **Minimize Secret Distribution**: Only use Secrets in Pods that need them
4. **Consider External Secret Management**: Use tools like HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault
5. **Use Immutable Secrets**: Make Secrets immutable to prevent accidental changes
6. **Regularly Rotate Secrets**: Change sensitive data periodically
7. **Audit Secret Access**: Monitor and audit access to Secrets

## Enabling Encryption at Rest

To encrypt Secrets at rest in etcd, you need to configure the API server with an encryption configuration:

1. Create an encryption configuration file:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-key>
      - identity: {}
```

2. Configure the API server to use the encryption configuration:

```
--encryption-provider-config=/path/to/encryption-config.yaml
```

3. Restart the API server

4. Re-create all existing Secrets to ensure they are encrypted

## Secret Best Practices

1. **Don't Commit Secrets to Source Control**: Never store Secret YAML files in source control
2. **Use Namespaces**: Organize Secrets by namespace to limit access
3. **Minimize Secret Content**: Store only what's necessary in Secrets
4. **Use Specific Secret Types**: Use the appropriate Secret type for your data
5. **Set Resource Quotas**: Limit the number of Secrets per namespace
6. **Use Immutable Secrets**: Make Secrets immutable when possible
7. **Implement Least Privilege**: Use RBAC to restrict access to Secrets
8. **Consider External Secret Management**: For highly sensitive data, use external secret management solutions
9. **Regularly Rotate Secrets**: Change sensitive data periodically
10. **Monitor Secret Usage**: Audit and monitor access to Secrets

## Common Use Cases for Secrets

### 1. Database Credentials

Store database connection credentials:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: s3cr3t
  connection-string: "postgresql://admin:s3cr3t@postgres-service:5432/mydb"
```

### 2. TLS Certificates

Store TLS certificates for secure communication:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

### 3. API Tokens

Store API tokens for external services:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-tokens
type: Opaque
stringData:
  github-token: "ghp_1234567890abcdef"
  aws-access-key: "AKIAIOSFODNN7EXAMPLE"
  aws-secret-key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

### 4. Docker Registry Credentials

Store credentials for pulling images from private registries:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: docker-registry-credentials
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

### 5. SSH Keys

Store SSH keys for Git operations or server access:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ssh-keys
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: <base64-encoded-private-key>
```

## Real-World Examples

### Web Application with Database Credentials

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: webapp-db-credentials
  labels:
    app: webapp
type: Opaque
stringData:
  username: webapp_user
  password: complex-password-123
  database: webapp_production
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: webapp:1.2.3
        env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: webapp-db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: webapp-db-credentials
              key: password
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: webapp-db-credentials
              key: database
```

### Microservice with Multiple Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: payment-service-secrets
type: Opaque
stringData:
  db-password: "db-password-123"
  stripe-api-key: "sk_test_1234567890abcdefghijklmn"
  encryption-key: "a1b2c3d4e5f6g7h8i9j0"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
      - name: payment-service
        image: payment-service:2.1.0
        volumeMounts:
        - name: secrets-volume
          mountPath: /app/secrets
          readOnly: true
        env:
        - name: STRIPE_API_KEY
          valueFrom:
            secretKeyRef:
              name: payment-service-secrets
              key: stripe-api-key
      volumes:
      - name: secrets-volume
        secret:
          secretName: payment-service-secrets
          items:
          - key: db-password
            path: database/password.txt
          - key: encryption-key
            path: encryption/key.txt
          defaultMode: 0400
```

### Ingress with TLS

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  tls:
  - hosts:
    - example.com
    secretName: tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: example-service
            port:
              number: 80
```

## Secret vs. Other Kubernetes Resources

### Secret vs. ConfigMap

- **Secrets** are designed for sensitive data and are base64 encoded
- **ConfigMaps** are for non-sensitive configuration data and are stored in plain text

### Secret vs. Environment Variables in Pod Spec

- **Secrets** provide a centralized way to manage sensitive data that can be shared across multiple Pods
- **Environment variables in Pod spec** are defined directly in the Pod and can't be shared or managed separately

### Secret vs. External Secret Management

- **Kubernetes Secrets** are native to Kubernetes but have limited security features
- **External Secret Management** (like HashiCorp Vault, AWS Secrets Manager) provide more advanced security features but require additional setup

## Advanced Secret Features

### Immutable Secrets

Make Secrets immutable to prevent accidental changes (Kubernetes v1.19+):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: immutable-secret
type: Opaque
data:
  username: YWRtaW4=
  password: czNjcjN0
immutable: true  # Makes this Secret immutable
```

### Secret Projections

Project specific keys to specific paths:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projection-example
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
  volumes:
  - name: secret-volume
    projected:
      sources:
      - secret:
          name: db-credentials
          items:
          - key: username
            path: database/username.txt
      - secret:
          name: api-credentials
          items:
          - key: api-key
            path: api/key.txt
```

### Optional Secrets

Make Secret references optional:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: optional-secret-example
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: optional-secret
          key: api-key
          optional: true  # Pod will start even if Secret doesn't exist
```

### Automatic Secret Rotation

While Kubernetes doesn't provide built-in secret rotation, you can implement it using:

1. **Sidecar Pattern**: Use a sidecar container to periodically update Secret files
2. **Init Container**: Use an init container to fetch the latest secrets before the main container starts
3. **External Secret Operators**: Use tools like Kubernetes External Secrets or Secrets Store CSI Driver

## Debugging Secrets

### Common Issues

1. **Secret Not Found**: Check if the Secret exists in the correct namespace
   ```bash
   kubectl get secret <secret-name> -n <namespace>
   ```

2. **Key Not Found**: Check if the referenced key exists in the Secret
   ```bash
   kubectl describe secret <secret-name>
   ```

3. **Permission Issues**: Check if the Pod's service account has access to the Secret
   ```bash
   kubectl auth can-i get secrets --as=system:serviceaccount:<namespace>:<serviceaccount>
   ```

4. **Volume Mount Issues**: Check if the volume is correctly mounted
   ```bash
   kubectl describe pod <pod-name>
   ```

### Useful Commands

```bash
# Check if a Secret exists
kubectl get secret <secret-name>

# View Secret metadata (without showing the values)
kubectl describe secret <secret-name>

# View Secret data (base64 encoded)
kubectl get secret <secret-name> -o yaml

# Decode a specific Secret value
kubectl get secret <secret-name> -o jsonpath='{.data.<key-name>}' | base64 --decode

# Check events related to a Pod using a Secret
kubectl get events --field-selector involvedObject.name=<pod-name>

# Check if a file from a Secret is correctly mounted
kubectl exec <pod-name> -- ls -la /path/to/mounted/directory

# View the contents of a mounted Secret file
kubectl exec <pod-name> -- cat /path/to/mounted/file

# Check environment variables in a container
kubectl exec <pod-name> -- env | grep <variable-name>
```

## External Secret Management Solutions

For enhanced security, consider using external secret management solutions:

### 1. HashiCorp Vault

```yaml
# Using the Vault Agent Injector
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault-example
spec:
  selector:
    matchLabels:
      app: vault-example
  template:
    metadata:
      labels:
        app: vault-example
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-secret-database-config.txt: "database/creds/db-role"
        vault.hashicorp.com/role: "db-app"
    spec:
      containers:
      - name: app
        image: app:1.0
```

### 2. Kubernetes External Secrets

```yaml
apiVersion: kubernetes-client.io/v1
kind: ExternalSecret
metadata:
  name: aws-secret
spec:
  backendType: secretsManager
  data:
    - key: aws/secret
      name: password
      property: password
```

### 3. Sealed Secrets

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mysecret
spec:
  encryptedData:
    password: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
```

## Conclusion

Secrets are a critical component in Kubernetes for managing sensitive information. While they provide a convenient way to store and distribute confidential data to your applications, it's important to understand their security limitations and implement additional measures for highly sensitive information.

By following best practices and considering the use of encryption at rest, RBAC, and possibly external secret management solutions, you can effectively secure your application's sensitive data in a Kubernetes environment.