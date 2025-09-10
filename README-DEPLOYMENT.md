# DIGIT Water Service - GCP Deployment

This repository contains everything needed to deploy the DIGIT Water Service on Google Cloud Platform (GCP).

## Quick Start

### Prerequisites

1. **Google Cloud SDK** - Install from [here](https://cloud.google.com/sdk/docs/install)
2. **kubectl** - Install from [here](https://kubernetes.io/docs/tasks/tools/)
3. **Docker** - Install from [here](https://docs.docker.com/get-docker/)
4. **Java 17** - For building the services locally
5. **Maven** - For building the services

### One-Command Deployment

```bash
./deploy-to-gcp.sh
```

This script will:
- Set up GCP project and enable required APIs
- Create GKE cluster
- Build and push Docker images
- Deploy to Kubernetes
- Verify deployment

## Manual Deployment Steps

### 1. GCP Project Setup

```bash
# Create project
gcloud projects create digit-water-service --name="DIGIT Water Service"
gcloud config set project digit-water-service

# Enable APIs
gcloud services enable container.googleapis.com
gcloud services enable sqladmin.googleapis.com
gcloud services enable redis.googleapis.com
gcloud services enable pubsub.googleapis.com
gcloud services enable storage.googleapis.com
gcloud services enable dns.googleapis.com
gcloud services enable monitoring.googleapis.com
gcloud services enable logging.googleapis.com
```

### 2. Database Setup

```bash
# Create Cloud SQL PostgreSQL instance
gcloud sql instances create digit-postgres \
  --database-version=POSTGRES_14 \
  --tier=db-standard-4 \
  --region=us-central1 \
  --storage-type=SSD \
  --storage-size=100GB \
  --backup-start-time=03:00 \
  --enable-bin-log

# Create databases
gcloud sql databases create ws_services --instance=digit-postgres
gcloud sql databases create ws_calculator --instance=digit-postgres
gcloud sql databases create egov_mdms --instance=digit-postgres
gcloud sql databases create property_service --instance=digit-postgres
gcloud sql databases create billing_service --instance=digit-postgres
gcloud sql databases create egov_user --instance=digit-postgres
gcloud sql databases create egov_workflow --instance=digit-postgres
```

### 3. Redis Setup

```bash
# Create Cloud Memorystore Redis instance
gcloud redis instances create digit-redis \
  --size=4 \
  --region=us-central1 \
  --redis-version=redis_6_x
```

### 4. Kubernetes Cluster

```bash
# Create GKE cluster
gcloud container clusters create digit-cluster \
  --zone=us-central1-a \
  --machine-type=e2-standard-8 \
  --num-nodes=3 \
  --enable-autoscaling \
  --min-nodes=2 \
  --max-nodes=10 \
  --enable-autorepair \
  --enable-autoupgrade \
  --disk-size=100GB \
  --disk-type=pd-ssd

# Get credentials
gcloud container clusters get-credentials digit-cluster --zone=us-central1-a
```

### 5. Build and Deploy

```bash
# Build Docker images
docker build -t gcr.io/digit-water-service/ws-services:latest ./URBAN/water-connection/ws-services/
docker build -t gcr.io/digit-water-service/ws-calculator:latest ./URBAN/water-connection/ws-calculator/

# Push to GCR
gcloud auth configure-docker
docker push gcr.io/digit-water-service/ws-services:latest
docker push gcr.io/digit-water-service/ws-calculator:latest

# Deploy to Kubernetes
kubectl apply -f k8s-configs/
```

## Configuration

### Update Configuration Files

Before deploying, update the following files with your actual values:

1. **k8s-configs/configmap.yaml**
   - Update database IP addresses
   - Update Redis IP addresses
   - Update service URLs

2. **k8s-configs/secrets.yaml**
   - Update database credentials
   - Update Redis password
   - Add JWT secrets

3. **k8s-configs/ingress.yaml**
   - Update domain name
   - Update SSL certificate configuration

### Environment Variables

The services use the following environment variables:

#### Database Configuration
- `DB_HOST`: Database host IP
- `DB_PORT`: Database port (default: 5432)
- `DB_NAME`: Database name
- `DB_USERNAME`: Database username
- `DB_PASSWORD`: Database password

#### Redis Configuration
- `REDIS_HOST`: Redis host IP
- `REDIS_PORT`: Redis port (default: 6379)
- `REDIS_PASSWORD`: Redis password

#### Service URLs
- `EGOV_MDMS_HOST`: MDMS service URL
- `PROPERTY_SERVICE_HOST`: Property service URL
- `EGOV_IDGEN_HOST`: ID generation service URL
- `EGOV_PERSISTER_HOST`: Persister service URL
- `EGOV_FILESTORE_HOST`: File store service URL
- `PDF_SERVICE_HOST`: PDF service URL
- `BILLING_SERVICE_HOST`: Billing service URL
- `EGOV_USER_HOST`: User service URL
- `EGOV_WORKFLOW_HOST`: Workflow service URL

## Monitoring and Logging

### View Logs

```bash
# View ws-services logs
kubectl logs -f deployment/ws-services -n digit-water

# View ws-calculator logs
kubectl logs -f deployment/ws-calculator -n digit-water
```

### Check Status

```bash
# Check pod status
kubectl get pods -n digit-water

# Check service status
kubectl get services -n digit-water

# Check ingress status
kubectl get ingress -n digit-water
```

### Health Checks

The services expose health check endpoints:
- `http://your-domain/ws-services/actuator/health`
- `http://your-domain/ws-calculator/actuator/health`

## Scaling

### Horizontal Scaling

```bash
# Scale ws-services
kubectl scale deployment ws-services --replicas=5 -n digit-water

# Scale ws-calculator
kubectl scale deployment ws-calculator --replicas=5 -n digit-water
```

### Vertical Scaling

Update resource limits in the deployment YAML files:
- `resources.requests.memory`
- `resources.requests.cpu`
- `resources.limits.memory`
- `resources.limits.cpu`

## Troubleshooting

### Common Issues

1. **Pod not starting**
   - Check logs: `kubectl logs <pod-name> -n digit-water`
   - Check resource limits
   - Verify environment variables

2. **Database connection issues**
   - Verify database IP and credentials
   - Check firewall rules
   - Ensure database is accessible from GKE

3. **Service communication issues**
   - Verify service URLs in configmap
   - Check network policies
   - Review DNS configuration

### Debug Commands

```bash
# Describe pod
kubectl describe pod <pod-name> -n digit-water

# Get events
kubectl get events -n digit-water

# Port forward for local testing
kubectl port-forward service/ws-services-service 8080:80 -n digit-water
```

## Resource Optimization

### Resource Optimization
- Use appropriate machine types
- Enable cluster and node autoscaling
- Use preemptible instances for non-critical workloads
- Set resource requests and limits

### Storage Optimization
- Use appropriate storage classes
- Enable compression
- Regular cleanup of old data

### Network Optimization
- Use Cloud CDN for static content
- Deploy closer to users
- Minimize cross-region traffic

### Monitoring and Alerting
- Set up resource monitoring
- Monitor resource usage
- Regular performance reviews

## Security

### Network Security
- Use private GKE clusters
- Configure firewall rules
- Enable Cloud Armor

### Application Security
- Use Cloud KMS for secrets
- Enable workload identity
- Regular security updates

## Support

For issues and questions:
- Check the [DIGIT documentation](https://digit-discuss.atlassian.net/)
- Create issues in the [GitHub repository](https://github.com/egovernments/municipal-services)
- Contact the development team

## License

This project is licensed under the MIT License - see the LICENSE file for details.
