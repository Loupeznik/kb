# Infisical Secret Management in Kubernetes

Infisical is a secrets management platform that integrates with Kubernetes to automatically inject secrets into pods. This guide covers how to configure and use Infisical with Kubernetes deployments.

## Prerequisites

- Infisical instance deployed in the cluster (typically in `infisical` namespace)
- Infisical Kubernetes Operator installed
- Access to Infisical UI for project and secret management

## Architecture Overview

```text
+---------------------+
|  Infisical UI       |  <- Manage projects, environments, and secrets
+----------+----------+
           |
           v
+---------------------+
|  Infisical Server   |  <- Central secret storage
+----------+----------+
           |
           v
+---------------------+
|  K8s Operator       |  <- Watches InfisicalSecret CRDs
+----------+----------+
           |
           v
+---------------------+
|  Kubernetes Secret  |  <- Auto-generated and synced
+----------+----------+
           |
           v
+---------------------+
|  Application Pod    |  <- Consumes secrets via envFrom
+---------------------+
```

## Step-by-Step Setup

### 1. Access Infisical UI

Access your Infisical instance:

```bash
# If using ingress
https://infisical.your-domain.com

# If using port-forward
kubectl port-forward -n infisical svc/infisical-infisical-standalone-infisical 8080:8080
# Then access: http://localhost:8080
```

### 2. Create Project in Infisical

1. Log in to Infisical UI
2. Click **"Create Project"** or navigate to **Projects**
3. Create a new project with a meaningful name (e.g., `my-application`)
4. Note the **Project Slug** (e.g., `my-application`) - this will be used in Kubernetes

### 3. Create Environments

Within your project, create environments:

- **`dev`** - Development
- **`stg`** - Staging
- **`prd`** - Production

Each environment will have its own set of secrets.

### 4. Add Secrets to Environment

Navigate to an environment and add your secrets as key-value pairs:

**Example secrets:**

```text
DATABASE_URL=postgresql://user:password@host:5432/dbname
REDIS_URL=redis://redis:6379
API_KEY=your-api-key-here
APP_SECRET=your-app-secret
```

### 5. Create Kubernetes Identity

This is the critical authentication step:

1. In Infisical UI, go to: **Project Settings → Access Control → Machine Identities**
2. Click **"Create Identity"** or **"Add Identity"**
3. Configure:
   - **Name**: `<app-name>-k8s-<env>` (e.g., `myapp-k8s-dev`)
   - **Type**: **Kubernetes Auth**

4. Kubernetes Auth Configuration:

```text
Kubernetes Host: https://kubernetes.default.svc
Token Reviewer JWT: (leave empty)
CA Certificate: (leave empty)
Allowed Namespaces: my-app-namespace
Allowed Service Accounts: my-app-serviceaccount
Allowed Names: (optional - pod name patterns)
```

5. **Copy the Identity ID** after creation (looks like a UUID):

```text
c884359c-6ca9-4e83-965d-c00583a031ad
```

6. Grant the identity **read** permissions to your project's environment

### 6. Create Kubernetes Resources

#### Service Account

First, create a ServiceAccount for your application:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  namespace: myapp-namespace
automountServiceAccountToken: true
```

#### InfisicalSecret CRD

Create the InfisicalSecret resource:

```yaml
apiVersion: secrets.infisical.com/v1alpha1
kind: InfisicalSecret
metadata:
  name: myapp-infisicalsecret
  namespace: myapp-namespace
spec:
  # Infisical API endpoint (in-cluster service)
  hostAPI: "http://infisical-infisical.infisical:8080"

  # Resync interval in seconds
  resyncInterval: 60

  # Authentication configuration
  authentication:
    kubernetesAuth:
      # Identity ID from Infisical UI
      identityId: "c884359c-6ca9-4e83-965d-c00583a031ad"
      autoCreateServiceAccountToken: true
      serviceAccountRef:
        name: myapp
        namespace: myapp-namespace

      # Which secrets to fetch
      secretsScope:
        projectSlug: "my-application"
        envSlug: "dev"
        secretsPath: "/"
        recursive: true

  # Kubernetes Secret to create/update
  managedKubeSecretReferences:
    - secretName: myapp-secret
      secretNamespace: myapp-namespace
      creationPolicy: "Orphan"
      template:
        includeAllSecrets: true
```

**Key Parameters Explained:**

- **`hostAPI`**: Infisical server endpoint
  - In-cluster: `http://infisical-infisical.infisical:8080`
  - External: `https://your-infisical-domain.com`

- **`identityId`**: The UUID from Kubernetes Identity creation

- **`autoCreateServiceAccountToken`**: Let operator create token automatically

- **`secretsScope`**:
  - `projectSlug`: Project identifier from Infisical
  - `envSlug`: Environment (dev/stg/prd)
  - `secretsPath`: Path within environment (`/` for all secrets)
  - `recursive`: Include secrets in subdirectories

- **`creationPolicy`**:
  - `"Orphan"`: Keep Kubernetes Secret if InfisicalSecret is deleted
  - `"Owner"`: Delete Kubernetes Secret when InfisicalSecret is deleted

#### Deployment Using Secrets

Reference the auto-generated Kubernetes Secret in your Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      serviceAccountName: myapp

      containers:
      - name: myapp
        image: myapp:latest

        # Method 1: Inject ALL secrets as environment variables
        envFrom:
        - secretRef:
            name: myapp-secret

        # Method 2: Inject specific secrets as environment variables
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: myapp-secret
              key: DATABASE_URL
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: myapp-secret
              key: API_KEY

        # Method 3: Mount secrets as files
        volumeMounts:
        - name: secrets
          mountPath: /etc/secrets
          readOnly: true

      volumes:
      - name: secrets
        secret:
          secretName: myapp-secret
```

## Helm Chart Integration

### Template Structure

**`templates/secret.yaml`:**

```yaml
apiVersion: secrets.infisical.com/v1alpha1
kind: InfisicalSecret
metadata:
  name: {{ include "myapp.fullname" . }}-infisicalsecret
spec:
  hostAPI: {{ .Values.infisical.hostAPI | quote }}
  resyncInterval: {{ .Values.infisical.resyncInterval }}

  authentication:
    kubernetesAuth:
      identityId: {{ .Values.secret.infisicalIdentityId | quote }}
      autoCreateServiceAccountToken: true
      serviceAccountRef:
        name: {{ include "myapp.serviceAccountName" . }}
        namespace: {{ .Release.Namespace }}

      secretsScope:
        projectSlug: {{ .Values.secret.projectSlug | quote }}
        envSlug: {{ .Values.secret.envSlug | quote }}
        secretsPath: "/"
        recursive: true

  managedKubeSecretReferences:
    - secretName: {{ include "myapp.fullname" . }}-secret
      secretNamespace: {{ .Release.Namespace }}
      creationPolicy: "Orphan"
      template:
        includeAllSecrets: true
```

**`templates/deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  template:
    spec:
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}

      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

        envFrom:
        - secretRef:
            name: {{ include "myapp.fullname" . }}-secret
```

**`values.yaml`:**

```yaml
secret:
  projectSlug: "my-application"
  envSlug: "dev"
  infisicalIdentityId: ""

infisical:
  hostAPI: "http://infisical-infisical.infisical:8080"
  resyncInterval: 60
```

**`values.dev.yaml`:**

```yaml
secret:
  projectSlug: "my-application"
  envSlug: "dev"
  infisicalIdentityId: "your-dev-identity-id"
```

**`values.prd.yaml`:**

```yaml
secret:
  projectSlug: "my-application"
  envSlug: "prd"
  infisicalIdentityId: "your-prd-identity-id"
```

### Deployment

```bash
helm upgrade --install myapp . \
  -f values.yaml \
  -f values.dev.yaml \
  --set image.tag=v1.0.0 \
  --namespace myapp-namespace
```

## Verification

### Check InfisicalSecret Status

```bash
# List InfisicalSecrets
kubectl get infisicalsecret -n myapp-namespace

# Describe InfisicalSecret
kubectl describe infisicalsecret myapp-infisicalsecret -n myapp-namespace
```

**Expected status:**

```yaml
Status:
  Conditions:
    Last Transition Time:  2024-01-01T00:00:00Z
    Message:               Infisical secrets synced
    Reason:                OK
    Status:                True
    Type:                  secrets.infisical.com/AutoRedeployReady
```

### Check Generated Kubernetes Secret

```bash
# Verify secret exists
kubectl get secret myapp-secret -n myapp-namespace

# List secret keys (without values)
kubectl get secret myapp-secret -n myapp-namespace -o jsonpath='{.data}' | jq 'keys'

# View a specific secret value (base64 decoded)
kubectl get secret myapp-secret -n myapp-namespace -o jsonpath='{.data.DATABASE_URL}' | base64 -d
```

### Test in Pod

```bash
# Get pod name
POD=$(kubectl get pod -n myapp-namespace -l app=myapp -o jsonpath='{.items[0].metadata.name}')

# Check environment variables in pod
kubectl exec -n myapp-namespace $POD -- env | grep -E "DATABASE_URL|API_KEY"

# Check mounted secret files
kubectl exec -n myapp-namespace $POD -- ls -la /etc/secrets
kubectl exec -n myapp-namespace $POD -- cat /etc/secrets/DATABASE_URL
```

## Troubleshooting

### InfisicalSecret Shows Errors

```bash
# Check InfisicalSecret events
kubectl describe infisicalsecret myapp-infisicalsecret -n myapp-namespace

# Check operator logs
kubectl logs -n infisical deployment/infisical-operat-controller-manager -c manager --tail=100

# Follow operator logs
kubectl logs -n infisical deployment/infisical-operat-controller-manager -c manager -f
```

### Common Issues

**1. Authentication Failed**

```text
Error: failed to authenticate with Infisical
```

**Solution:**

- Verify `identityId` is correct
- Check that namespace is allowed in Infisical identity
- Verify service account name matches allowed service accounts

**2. Secrets Not Syncing**

```text
Error: failed to fetch secrets from Infisical
```

**Solution:**

- Check `projectSlug` and `envSlug` match Infisical project/environment
- Verify identity has read permissions on the environment
- Check Infisical API endpoint is accessible from the cluster

**3. Service Account Token Issues**

```text
Error: failed to get service account token
```

**Solution:**

- Ensure `automountServiceAccountToken: true` on ServiceAccount
- Verify ServiceAccount exists in the correct namespace
- Check `autoCreateServiceAccountToken: true` in InfisicalSecret

**4. Secret Not Created**

```text
Error: managed secret not found
```

**Solution:**

- Check operator logs for detailed errors
- Verify CRD is installed: `kubectl get crd infisicalsecrets.secrets.infisical.com`
- Ensure operator has permissions to create secrets in target namespace

### Debug Checklist

- [ ] Infisical server is accessible from cluster
- [ ] Project and environment exist in Infisical
- [ ] Secrets are added to the environment
- [ ] Kubernetes Identity created with correct namespace/service account
- [ ] Identity has read permissions on environment
- [ ] Identity ID is correct in InfisicalSecret
- [ ] ServiceAccount exists and automounts token
- [ ] InfisicalSecret has no errors in status
- [ ] Kubernetes Secret is created with expected keys
- [ ] Pod references the correct secret name

## Best Practices

### Security

1. **Use separate identities per environment**

```text
app-k8s-dev   -> dev environment
app-k8s-stg   -> stg environment
app-k8s-prd   -> prd environment
```

2. **Restrict namespace and service account access**
   - Only allow specific namespaces
   - Only allow specific service accounts
   - Use least-privilege principle

3. **Use different projects for different sensitivity levels**
   - Separate project for highly sensitive data
   - Different access controls per project

### Operations

1. **Set appropriate resync intervals**

```yaml
resyncInterval: 60   # Default: sync every 60 seconds
resyncInterval: 300  # Less frequent: every 5 minutes
```

2. **Use `creationPolicy: "Orphan"`**
   - Prevents accidental secret deletion
   - Allows manual cleanup if needed

3. **Monitor InfisicalSecret health**

```bash
kubectl get infisicalsecret -A -o wide
```

4. **Use secret paths for organization**

```text
/database/primary
/database/readonly
/api/external
/api/internal
```

### CI/CD Integration

1. **Never commit Identity IDs to Git**
   - Use `values.yaml` with empty `infisicalIdentityId`
   - Override via `values.<env>.yaml` (not in Git)
   - Or pass via `--set` in CI/CD pipeline

2. **Rotate identities regularly**
   - Create new identity
   - Update deployment
   - Revoke old identity

3. **Test secret sync in staging first**
   - Deploy to staging environment
   - Verify secrets are correct
   - Then promote to production

## Advanced Patterns

### Multiple Secret Sources

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        envFrom:
        # Infisical secrets
        - secretRef:
            name: app-infisical-secret
        # Manual secrets (non-sensitive config)
        - configMapRef:
            name: app-config
```

### Selective Secret Injection

```yaml
# Only specific secrets from Infisical
env:
- name: DATABASE_PASSWORD
  valueFrom:
    secretKeyRef:
      name: app-secret
      key: DB_PASSWORD
- name: API_TOKEN
  valueFrom:
    secretKeyRef:
      name: app-secret
      key: API_TOKEN
# Other config from ConfigMap
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: LOG_LEVEL
```

### Secret Paths Organization

```yaml
# Fetch only specific path
secretsScope:
  projectSlug: "my-app"
  envSlug: "prd"
  secretsPath: "/database"
  recursive: false
```

## Migration from Manual Secrets

1. **Add secrets to Infisical** (but don't delete manual secrets yet)
2. **Deploy InfisicalSecret** with different name
3. **Test with one pod** using new secret
4. **Gradually roll out** to all pods
5. **Delete manual secrets** after verification

## Summary

Infisical integration provides:

- Centralized secret management
- Automatic secret rotation and sync
- Kubernetes-native integration
- Environment-specific configurations
- Audit logging and access control
- No secrets in Git repositories
