# UIDAM Web Admin Portal Helm Chart

This Helm chart deploys the UIDAM Web Admin Portal - a React-based administration interface for the UIDAM platform.

## Overview

The UIDAM Web Admin Portal is a frontend web application built with:
- **React 18** with TypeScript
- **Material-UI (MUI)** for UI components
- **Redux Toolkit** for state management
- **Nginx** for serving static content

## Prerequisites

- Kubernetes 1.19+
- Helm 3.2.0+
- Access to Artifactory Docker registry (`artifactory-fr.harman.com:5062`)
- Image pull secrets configured (`artifactory`, `ghcr-secret`)

## Architecture

### Components

- **Deployment**: Runs nginx serving the React application
- **Service**: ClusterIP service exposing port 80
- **Ingress**: AWS ALB ingress for external access
- **ConfigMap**: Contains nginx configuration and API endpoint URLs
- **ServiceAccount**: Kubernetes service account for the pod
- **HPA** (optional): Horizontal Pod Autoscaler for scaling

### Container

- **Image**: `artifactory-fr.harman.com:5062/ignite/uidam-web-admin-portal`
- **Base**: nginx:alpine
- **Port**: 80 (HTTP)
- **Health Check**: `/health` endpoint

## Configuration

### Key Values

#### Image Configuration
```yaml
image:
  repository: arti.dev.ignite.harman.com:5061/ignite/uidam-web-admin-portal
  tag: "latest"
  pullPolicy: IfNotPresent
```

**Note**: 
- CI/CD pushes to external registry: `artifactory-fr.harman.com:5062/ignite/uidam-web-admin-portal:<tag>`
- ArgoCD pulls from internal registry: `arti.dev.ignite.harman.com:5061/ignite/uidam-web-admin-portal:<tag>`
- Image replication between registries is handled automatically by infrastructure

#### Replica Count
```yaml
replicaCount: 2
```

#### Service Configuration
```yaml
service:
  type: ClusterIP
  port: 80
  targetPort: 80
```

#### Ingress Configuration
```yaml
ingress:
  enabled: true
  className: alb
  scheme: internal
  hosts:
  - hostPrefix: uidam-web-admin-portal
    paths:
    - /
```

#### API Endpoints
The portal needs to communicate with backend services:
```yaml
apiEndpoints:
  authorizationServer: "https://uidam-authorization-server.local"
  userManagement: "https://uidam-user-management.local"
  entityManagement: "https://uidam-entity-management.local"
```

#### Resource Limits
```yaml
resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

## Installation

### Using Helm

```bash
# Install the chart
helm install uidam-web-admin-portal ./charts/uidam-web-admin-portal \
  --namespace identity-server \
  --set image.tag=1.0.0 \
  --set environmentDomainName=dev.harman.com

# Upgrade the chart
helm upgrade uidam-web-admin-portal ./charts/uidam-web-admin-portal \
  --namespace identity-server \
  --set image.tag=1.0.1

# Uninstall the chart
helm uninstall uidam-web-admin-portal --namespace identity-server
```

### Using ArgoCD

The chart is designed to be deployed via ArgoCD using the app-of-apps pattern:

1. Enable the application in `app-of-apps/values.yaml`:
```yaml
applications:
  uidam-web-admin-portal:
    enabled: true
```

2. Configure environment-specific values in ArgoCD application overrides

3. ArgoCD will automatically sync and deploy the application

## Environment-Specific Configuration

### Development
```yaml
environmentDomainName: dev.harman.com
replicaCount: 1
ingress:
  scheme: internal
apiEndpoints:
  authorizationServer: "https://uidam-authorization-server.dev.harman.com"
```

### Staging
```yaml
environmentDomainName: staging.harman.com
replicaCount: 2
ingress:
  scheme: internal
```

### Production
```yaml
environmentDomainName: harman.com
replicaCount: 3
ingress:
  scheme: internet-facing
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
```

## Nginx Configuration

The chart includes a custom nginx configuration with:
- Gzip compression for static assets
- Security headers (X-Frame-Options, X-Content-Type-Options, etc.)
- React Router support (SPA routing)
- Cache headers for static assets
- Health check endpoint at `/health`

## Health Checks

### Liveness Probe
- Path: `/health`
- Initial Delay: 10s
- Period: 10s
- Timeout: 5s

### Readiness Probe
- Path: `/health`
- Initial Delay: 5s
- Period: 10s
- Timeout: 5s

## Autoscaling (Optional)

Enable HPA by setting:
```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

## Troubleshooting

### Check Pod Status
```bash
kubectl get pods -n identity-server -l app.kubernetes.io/name=uidam-web-admin-portal
```

### View Logs
```bash
kubectl logs -n identity-server -l app.kubernetes.io/name=uidam-web-admin-portal
```

### Check Ingress
```bash
kubectl get ingress -n identity-server uidam-web-admin-portal
```

### Test Health Endpoint
```bash
kubectl port-forward -n identity-server svc/uidam-web-admin-portal 8080:80
curl http://localhost:8080/health
```

### Common Issues

1. **Image Pull Errors**: Ensure image pull secrets are configured
2. **404 Errors**: Check nginx configuration and React Router setup
3. **API Connection Issues**: Verify apiEndpoints configuration
4. **Ingress Not Working**: Check ALB annotations and security groups

## Maintenance

### Updating the Application

1. Build and push new Docker image via CI/CD
2. Update image tag in values or via ArgoCD
3. ArgoCD will automatically sync if auto-sync is enabled

### Rolling Back

```bash
helm rollback uidam-web-admin-portal <revision> --namespace identity-server
```

## Security Considerations

- Image pull secrets are required for accessing Artifactory
- Ingress is configured as internal by default
- Security headers are enabled in nginx configuration
- Container runs as non-root user (nginx default)

## Support

For issues or questions, contact the HARMAN Auto UIDAM Team.

## References

- [UIDAM Authorization Server](../uidam-authorization-server/)
- [UIDAM User Management](../uidam-user-management/)
- [ArgoCD App-of-Apps Pattern](../../app-of-apps/)
