# ArgoCD Trocks Apps - App of Apps Pattern

A Helm-based ArgoCD deployment using the **App of Apps** pattern for managing multiple microservices in a Kubernetes cluster. This project provides a centralized way to deploy and manage Spring Boot microservices using ArgoCD's GitOps approach.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    ArgoCD App of Apps                           │
│                                                                 │
│  ┌─────────────────┐                                           │
│  │   Bootstrap     │ ──┐                                       │
│  │  (Parent App)   │   │                                       │
│  └─────────────────┘   │                                       │
│                        │                                       │
│                        ├── ┌─────────────────┐                │
│                        │   │ Gateway Service │                │
│                        │   │   (Child App)   │                │
│                        │   └─────────────────┘                │
│                        │                                       │
│                        ├── ┌─────────────────┐                │
│                        │   │ Person Service  │                │
│                        │   │   (Child App)   │                │
│                        │   └─────────────────┘                │
│                        │                                       │
│                        ├── ┌─────────────────┐                │
│                        │   │ Address Service │                │
│                        │   │   (Child App)   │                │
│                        │   └─────────────────┘                │
│                        │                                       │
│                        └── ┌─────────────────┐                │
│                            │ Config Server   │                │
│                            │   (Child App)   │                │
│                            └─────────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

## Project Structure

```
argocd-trocks-apps/
├── bootstrap/                    # Parent App (App of Apps)
│   ├── templates/
│   │   ├── apps.yaml            # Child app definitions
│   │   ├── access-control.yml   # RBAC configuration
│   │   ├── configmap.yaml       # Shared configuration
│   │   └── secret.yaml          # Shared secrets
│   ├── Chart.yaml               # Bootstrap chart metadata
│   └── values.yaml              # Bootstrap configuration
├── charts/
│   └── base-service/            # Reusable Helm chart template
│       ├── templates/
│       │   ├── deployment.yaml  # Kubernetes deployment
│       │   ├── service.yaml     # Kubernetes service
│       │   └── ingress.yaml     # Ingress configuration
│       ├── Chart.yaml           # Base service chart metadata
│       └── values.yaml          # Default values
└── apps/                        # Individual service configurations
    ├── gateway-service/
    │   └── values.yaml          # Gateway-specific values
    ├── person-service/
    │   └── values.yaml          # Person service values
    ├── address-service/
    │   └── values.yaml          # Address service values
    └── cloud-config-server/
        └── values.yaml          # Config server values
```

## Key Components

### 1. Bootstrap (Parent App)
The bootstrap application manages all child applications using ArgoCD's App of Apps pattern.

**Features:**
- Automatically creates child applications for each service
- Manages shared resources (RBAC, ConfigMaps, Secrets)
- Centralized configuration management
- Automated sync policies with self-healing

### 2. Base Service Chart
A reusable Helm chart template that provides common Kubernetes resources for Spring Boot microservices.

**Includes:**
- Kubernetes Deployment with configurable replicas
- Service for internal communication
- Ingress with AWS ALB integration
- Spring profiles configuration
- Service discovery settings

### 3. Individual Service Apps
Each microservice has its own `values.yaml` file that overrides the base service defaults.

**Configuration includes:**
- Docker image repository and tag
- Replica count
- Spring profiles
- Ingress settings
- Service-specific environment variables

## Quick Start

### Prerequisites
- ArgoCD installed in Kubernetes cluster
- Access to the Git repository
- Kubernetes cluster with appropriate permissions

### Deployment Steps

1. **Deploy the Bootstrap Application**
   ```bash
   # Apply the bootstrap app to ArgoCD
   kubectl apply -f - <<EOF
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: bootstrap-parent-app
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: 'https://github.com/btamilselvan/argocd-trocks-apps.git'
       targetRevision: develop
       path: bootstrap
     destination:
       server: https://kubernetes.default.svc
       namespace: terraform-trocks-namespace
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
   EOF
   ```

2. **Verify Deployment**
   ```bash
   # Check ArgoCD applications
   kubectl get applications -n argocd
   
   # Check deployed services
   kubectl get pods,svc,ingress -n terraform-trocks-namespace
   ```

## Configuration

### Bootstrap Configuration (`bootstrap/values.yaml`)
```yaml
namespace: terraform-trocks-namespace
apps:
  - name: gateway-service
  - name: person-service
  - name: address-service
  - name: cloud-config-server
```

### Base Service Defaults (`charts/base-service/values.yaml`)
```yaml
spring:
  profiles:
    active: dev,kubernetes
namespace: terraform-trocks-namespace
replicas: 1
containerPort: 8080
discovery:
  enabled: false
ingress:
  enabled: false
alb:
  group_name: trocks-k8-shared-alb
  certificate_arn: "arn:aws:acm:us-east-2:194205500841:certificate/..."
```

### Service-Specific Configuration Example (`apps/gateway-service/values.yaml`)
```yaml
spring:
  profiles:
    active: dev,kubernetes
replicas: 1
discovery:
  enabled: false
ingress:
  enabled: true
appName: gateway-service
image:
  repository: "194205500841.dkr.ecr.us-east-2.amazonaws.com/gateway-service"
  tag: "5542feb"
```

## Adding New Services

To add a new microservice:

1. **Create service directory**
   ```bash
   mkdir apps/new-service
   ```

2. **Create values.yaml**
   ```yaml
   # apps/new-service/values.yaml
   appName: new-service
   image:
     repository: "your-registry/new-service"
     tag: "latest"
   replicas: 2
   ingress:
     enabled: true
   ```

3. **Update bootstrap configuration**
   ```yaml
   # bootstrap/values.yaml
   apps:
     - name: gateway-service
     - name: person-service
     - name: address-service
     - name: cloud-config-server
     - name: new-service  # Add new service
   ```

4. **Commit and push changes**
   ```bash
   git add .
   git commit -m "Add new-service"
   git push origin develop
   ```

ArgoCD will automatically detect the changes and deploy the new service.

## CI/CD Integration

### Automated Deployment Pipeline

The project integrates with CI/CD pipelines to enable fully automated deployments:

1. **Code Changes** → Developer pushes code to service repository
2. **Build Pipeline** → CI/CD builds new Docker image and pushes to ECR
3. **Image Tag Update** → Build pipeline updates `image.tag` in the respective `apps/{service}/values.yaml`
4. **GitOps Trigger** → ArgoCD detects the Git change and automatically deploys the new version

### Build Pipeline Integration

Your web service build pipelines should include a step to update the image tag:

```bash
# Example CI/CD pipeline step
# After building and pushing Docker image
NEW_TAG=$(git rev-parse --short HEAD)

# Update the values.yaml file
sed -i "s/tag: .*/tag: \"$NEW_TAG\"/" apps/gateway-service/values.yaml

# Commit and push the change
git add apps/gateway-service/values.yaml
git commit -m "Update gateway-service image tag to $NEW_TAG"
git push origin develop
```

### Deployment Flow

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Code Commit   │───▶│   Build & Push  │───▶│  Update Tag in  │
│  (Service Repo) │    │   Docker Image  │    │   values.yaml   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                        │
                                                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   ArgoCD Sync   │◀───│  Git Repository │◀───│   Git Commit    │
│   & Deploy      │    │     Change      │    │   & Push        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Benefits

- **Zero-Touch Deployment**: No manual intervention required
- **Audit Trail**: All deployments tracked through Git commits
- **Rollback Capability**: Easy rollback by reverting Git commits
- **Environment Consistency**: Same process across dev, qa, and prod

## Features

### Automated GitOps
- **Self-Healing**: Automatically reverts manual changes
- **Auto-Sync**: Deploys changes from Git automatically
- **Pruning**: Removes resources deleted from Git

### AWS Integration
- **ALB Ingress**: Shared Application Load Balancer
- **ECR Integration**: Private container registry
- **SSL/TLS**: Automatic certificate management

### Spring Boot Optimized
- **Multi-Profile Support**: dev, qa, prod environments
- **Service Discovery**: Kubernetes-native discovery
- **Configuration Management**: External config server integration

## Monitoring & Management

### ArgoCD UI
Access the ArgoCD UI to monitor application status:
- Application health and sync status
- Resource tree visualization
- Deployment history and rollbacks

### Useful Commands
```bash
# Check application status
argocd app list

# Sync specific application
argocd app sync gateway-service

# View application details
argocd app get gateway-service

# Manual sync all applications
argocd app sync bootstrap-parent-app
```

## Troubleshooting

### Common Issues

1. **Application Not Syncing**
   - Check Git repository access
   - Verify targetRevision (branch/tag)
   - Review ArgoCD application logs

2. **Image Pull Errors**
   - Verify ECR repository permissions
   - Check image tag exists
   - Validate AWS credentials

3. **Ingress Issues**
   - Confirm ALB controller is installed
   - Check certificate ARN validity
   - Verify security group configurations

### Debug Commands
```bash
# Check ArgoCD application status
kubectl describe application gateway-service -n argocd

# View pod logs
kubectl logs -f deployment/gateway-service -n terraform-trocks-namespace

# Check ingress status
kubectl describe ingress gateway-service -n terraform-trocks-namespace
```

## Best Practices

### Repository Management
- Use feature branches for development
- Tag releases for production deployments
- Implement proper Git workflow with reviews

### Security
- Store sensitive data in Kubernetes Secrets
- Use least-privilege RBAC policies
- Regularly update base images and dependencies

### Configuration Management
- Keep environment-specific values in separate files
- Use meaningful naming conventions
- Document configuration changes

## Related Projects
- [Spring Cloud Kubernetes Integration](../k8/README.md)
- [Ordering Service](../ordering-service/README.md)
- [Notification Service](../notification-service/README.md)
- [Recipe Service](../recipeservice/README.md)

## References
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Helm Charts Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)