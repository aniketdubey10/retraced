# Complete Retraced Deployment Guide

## Getting Started

### Clone the Repository
First, clone this forked repository which contains the necessary fixes and deployment configurations:

```bash
git clone https://github.com/aniketdubey10/retraced.git
cd retraced
# Switch to the deploy branch
git checkout deploy
```

## Prerequisites

- Kubernetes cluster
- Helm 3.x
- kubectl configured
- PostgreSQL database
- Elasticsearch cluster

## 1. Deploy Retraced

### Step 1: Create namespace
```bash
kubectl create namespace <NAMESPACE>
```

### Step 2: Configure values-<ENV>.yaml
Create environment-specific values file (e.g., `values-production.yaml`, `values-staging.yaml`, `values-dev.yaml`):
```yaml
# Configuration for Retraced
api:
  type: ClusterIP
auditlog:
  # Database configuration
  postgresDatabase: "<DATABASE_NAME>"
  postgresHost: "<POSTGRES_HOST>"
  postgresPassword: "<RAW_PASSWORD>"
  postgresPort: "5432"
  postgresUser: "<POSTGRES_USER>"

  # Elasticsearch configuration
  elasticsearchNodes: "<ELASTICSEARCH_URL>"
  
  # Environment settings
  stage: "<ENVIRONMENT>"
  retracedApiBase: "https://<YOUR_DOMAIN>/auditlog"

# Fix cron image tag inconsistency
cron:
  cron:
    image:
      tag: "1.12.1"
```

### Step 3: Deploy Retraced
```bash
helm install retraced ./helm/retraced \
  --namespace <NAMESPACE> \
  --values values-<ENV>.yaml \
  --set-string auditlog.postgresDatabase="<DATABASE_NAME>" \
  --set-string auditlog.postgresHost="<POSTGRES_HOST>" \
  --set-string auditlog.postgresPassword="<RAW_PASSWORD>" \
  --set-string auditlog.elasticsearchNodes="<ELASTICSEARCH_URL>"
```

### Step 4: Verify Retraced deployment
```bash
kubectl get pods -n <NAMESPACE> | grep retraced
kubectl logs deployment/retraced-api -n <NAMESPACE>
```

## 2. Deploy Admin Portal

### Step 1: Create admin portal secrets
Create `admin-portal-secrets-<ENV>.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: adminportal
  namespace: <NAMESPACE>
  labels:
    app: adminportal
stringData:
  # Jackson settings - PostgreSQL connection
  DB_URL: "postgres://<POSTGRES_USER>:<RAW_PASSWORD>@<POSTGRES_HOST>:5432/<DATABASE_NAME>?sslmode=require"
  SAML_AUDIENCE: ""
  JACKSON_API_KEYS: "<RANDOM_SECRET>"
  DB_TYPE: ""
  DB_TTL: ""
  DB_CLEANUP_LIMIT: ""
  DB_ENCRYPTION_KEY: ""
  SMTP_HOST: "<SMTP_HOST>"
  SMTP_PORT: "<SMTP_PORT>"
  SMTP_USER: "<SMTP_USER>"
  SMTP_PASSWORD: "<SMTP_PASSWORD>"
  SMTP_FROM: "<FROM_EMAIL>"
  NEXTAUTH_JWT_SIGNING_PRIVATE_KEY: ""
  NEXTAUTH_URL: "https://<ADMIN_PORTAL_DOMAIN>/"
  NEXTAUTH_SECRET: "<RANDOM_SECRET>"
  NEXTAUTH_ACL: ""
  NEXTAUTH_ADMIN_CREDENTIALS: "<ADMIN_EMAIL>:<ADMIN_PASSWORD>"
  BOXYHQ_NO_ANALYTICS: "1"
  RETRACED_HOST_URL: "http://retraced-api-internal:3000/auditlog"
  RETRACED_EXTERNAL_URL: "https://<RETRACED_DOMAIN>/auditlog"
  RETRACED_ADMIN_ROOT_TOKEN: "<ADMIN_ROOT_TOKEN>"
  BOXYHQ_LICENSE_KEY: ""
  NODE_TLS_REJECT_UNAUTHORIZED: "0"
```

### Step 2: Create admin portal deployment
Create `admin-portal-deployment-<ENV>.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retraced-admin-portal
  namespace: <NAMESPACE>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auditlog
      tier: admin-portal
  template:
    metadata:
      labels:
        app: auditlog
        tier: admin-portal
    spec:
      containers:
      - name: admin-portal
        image: boxyhq/jackson:1.30.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5225
          name: http
          protocol: TCP
        envFrom:
        - secretRef:
            name: adminportal
        readinessProbe:
          httpGet:
            path: /api/health
            port: 5225
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 5
          successThreshold: 2
        startupProbe:
          httpGet:
            path: /api/health
            port: 5225
          periodSeconds: 10
          timeoutSeconds: 10
          failureThreshold: 5
        resources:
          requests:
            cpu: 100m
          limits:
            cpu: 1000m
```

### Step 3: Create admin portal service
Create `admin-portal-service-<ENV>.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: retraced-admin-portal
  namespace: <NAMESPACE>
  labels:
    app: auditlog
    tier: admin-portal
spec:
  type: ClusterIP
  ports:
  - name: http
    port: 5225
    targetPort: 5225
  selector:
    app: auditlog
    tier: admin-portal
```

### Step 4: Apply admin portal resources
```bash
kubectl apply -f admin-portal-secrets-<ENV>.yaml
kubectl apply -f admin-portal-deployment-<ENV>.yaml
kubectl apply -f admin-portal-service-<ENV>.yaml
```

### Step 5: Verify admin portal deployment
```bash
kubectl get pods -n <NAMESPACE> | grep admin-portal
kubectl logs deployment/retraced-admin-portal -n <NAMESPACE>
```

## 3. Expose Services (Optional)

### Option 1: Port forwarding (for testing)
```bash
# Retraced API
kubectl port-forward service/retraced-api 3000:3000 -n <NAMESPACE>

# Admin Portal
kubectl port-forward service/retraced-admin-portal 5225:5225 -n <NAMESPACE>
```

### Option 2: Ingress (for production)
Create ingress resources for external access:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retraced-ingress
  namespace: <NAMESPACE>
spec:
  rules:
  - host: <RETRACED_DOMAIN>
    http:
      paths:
      - path: /auditlog
        pathType: Prefix
        backend:
          service:
            name: retraced-api
            port:
              number: 3000
  - host: <ADMIN_PORTAL_DOMAIN>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: retraced-admin-portal
            port:
              number: 5225
```

## 6. Troubleshooting

### Check pod status
```bash
kubectl get pods -n <NAMESPACE>
```

### Check logs
```bash
kubectl logs deployment/retraced-api -n <NAMESPACE>
kubectl logs deployment/retraced-processor -n <NAMESPACE>
kubectl logs deployment/retraced-admin-portal -n <NAMESPACE>
```

### Test connectivity
```bash
# Test Retraced API health
kubectl exec -it deployment/retraced-api -n <NAMESPACE> -- curl http://localhost:3000/healthz

# Test Admin Portal health
kubectl exec -it deployment/retraced-api -n <NAMESPACE> -- curl http://retraced-admin-portal:5225/api/health
```

---

## Special Case: PostgreSQL Password with Special Characters

When deploying Retraced with PostgreSQL passwords containing special characters, use this two-phase deployment process to avoid URL encoding issues.

### Phase 1: Deploy with URL-encoded password

```bash
helm install retraced ./helm/retraced \
  --namespace <NAMESPACE> \
  --values values-<ENV>.yaml \
  --set-string auditlog.postgresDatabase='<DATABASE_NAME>' \
  --set-string auditlog.postgresHost="<POSTGRES_HOST>" \
  --set-string auditlog.postgresPassword="<URL_ENCODED_PASSWORD>" \
  --set-string auditlog.elasticsearchNodes="<ELASTICSEARCH_URL>"
```

### Phase 2: Upgrade with raw password after migrations complete

```bash
kubectl delete job retraced-migrate-pg -n <NAMESPACE>

helm upgrade retraced ./helm/retraced \
  --namespace <NAMESPACE> \
  --values values-<ENV>.yaml \
  --set-string auditlog.postgresDatabase='<DATABASE_NAME>' \
  --set-string auditlog.postgresHost="<POSTGRES_HOST>" \
  --set-string auditlog.postgresPassword="<RAW_PASSWORD>" \
  --set-string auditlog.elasticsearchNodes="<ELASTICSEARCH_URL>"

kubectl rollout restart deployment retraced-api -n <NAMESPACE>
```

### Why This Process?

- **Phase 1**: Database migrations require URL-encoded passwords in connection strings
- **Phase 2**: Main application components require raw passwords for PostgreSQL connections
- **Restart**: Ensures pods pick up the updated password environment variables

### URL Encoding for Special Characters
If your PostgreSQL password contains special characters, encode them:
- `/` → `%2F`
- `^` → `%5E`
- `(` → `%28`
- `)` → `%29`
- `@` → `%40`
- `:` → `%3A`

### Admin Portal with Special Characters
Also update the admin portal secret to use URL-encoded password:
```yaml
stringData:
  DB_URL: "postgres://<POSTGRES_USER>:<URL_ENCODED_PASSWORD>@<POSTGRES_HOST>:5432/<DATABASE_NAME>?sslmode=require"
``` 