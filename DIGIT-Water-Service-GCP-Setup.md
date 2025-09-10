# DIGIT Water Service - Google Cloud Platform (GCP) Setup Guide

## Overview

This document provides a comprehensive guide for deploying the DIGIT Water Service on Google Cloud Platform (GCP). The water service is a microservices-based application that handles water connection applications, billing, and management for municipal services.

## Architecture Overview

The DIGIT Water Service consists of two main microservices:
- **ws-services**: Handles water connection applications and workflow management
- **ws-calculator**: Manages billing calculations and demand generation

### Service Dependencies

The water service depends on the following core services:
- **egov-mdms-service**: Master Data Management Service
- **property-service**: Property management service
- **egov-idgen**: ID generation service
- **egov-persister**: Data persistence service
- **egov-filestore**: File storage service
- **pdf-service**: PDF generation service
- **billing-service**: Billing and payment processing
- **egov-user**: User management service
- **egov-workflow-v2**: Workflow management service

## Infrastructure Requirements

### 1. Google Cloud Platform Services

#### Core GCP Services Required:
- **Google Kubernetes Engine (GKE)**: Container orchestration
- **Cloud SQL (PostgreSQL)**: Primary database
- **Cloud Memorystore (Redis)**: Caching layer
- **Cloud Pub/Sub**: Message queuing (Kafka alternative)
- **Cloud Storage**: File storage
- **Cloud Load Balancing**: Traffic distribution
- **Cloud DNS**: Domain management
- **Cloud IAM**: Identity and access management
- **Cloud Monitoring**: Application monitoring
- **Cloud Logging**: Centralized logging

#### Optional but Recommended:
- **Cloud CDN**: Content delivery
- **Cloud Armor**: Security and DDoS protection
- **Cloud Key Management Service (KMS)**: Secret management
- **Cloud Build**: CI/CD pipeline
- **Cloud Source Repositories**: Code repository

### 2. Hardware Requirements

#### Minimum Configuration:
- **CPU**: 8 vCPUs per service (2 services × 8 = 16 vCPUs minimum)
- **Memory**: 16 GB RAM per service (2 services × 16 = 32 GB minimum)
- **Storage**: 100 GB SSD per service
- **Network**: 1 Gbps bandwidth

#### Recommended Production Configuration:
- **CPU**: 16 vCPUs per service
- **Memory**: 32 GB RAM per service
- **Storage**: 500 GB SSD per service
- **Network**: 10 Gbps bandwidth

## Step-by-Step GCP Setup

### Phase 1: GCP Project Setup

1. **Create GCP Project**
   ```bash
   gcloud projects create digit-water-service --name="DIGIT Water Service"
   gcloud config set project digit-water-service
   ```

2. **Enable Required APIs**
   ```bash
   gcloud services enable container.googleapis.com
   gcloud services enable sqladmin.googleapis.com
   gcloud services enable redis.googleapis.com
   gcloud services enable pubsub.googleapis.com
   gcloud services enable storage.googleapis.com
   gcloud services enable dns.googleapis.com
   gcloud services enable monitoring.googleapis.com
   gcloud services enable logging.googleapis.com
   ```

3. **Set up Billing Account**
   - Link a billing account to the project
   - Configure budget alerts for cost monitoring

### Phase 2: Database Setup

1. **Create Cloud SQL PostgreSQL Instance**
   ```bash
   gcloud sql instances create digit-postgres \
     --database-version=POSTGRES_14 \
     --tier=db-standard-4 \
     --region=us-central1 \
     --storage-type=SSD \
     --storage-size=100GB \
     --backup-start-time=03:00 \
     --enable-bin-log
   ```

2. **Create Databases**
   ```sql
   -- Connect to the instance and create databases
   CREATE DATABASE ws_services;
   CREATE DATABASE ws_calculator;
   CREATE DATABASE egov_mdms;
   CREATE DATABASE property_service;
   CREATE DATABASE billing_service;
   CREATE DATABASE egov_user;
   CREATE DATABASE egov_workflow;
   ```

### Phase 3: Redis Setup

1. **Create Cloud Memorystore Redis Instance**
   ```bash
   gcloud redis instances create digit-redis \
     --size=4 \
     --region=us-central1 \
     --redis-version=redis_6_x
   ```

### Phase 4: Kubernetes Cluster Setup

1. **Create GKE Cluster**
   ```bash
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
   ```

2. **Get Cluster Credentials**
   ```bash
   gcloud container clusters get-credentials digit-cluster --zone=us-central1-a
   ```

### Phase 5: Container Registry Setup

1. **Create Container Registry**
   ```bash
   gcloud auth configure-docker
   ```

2. **Build and Push Docker Images**
   ```bash
   # Build ws-services image
   docker build -t gcr.io/digit-water-service/ws-services:latest ./URBAN/water-connection/ws-services/
   docker push gcr.io/digit-water-service/ws-services:latest

   # Build ws-calculator image
   docker build -t gcr.io/digit-water-service/ws-calculator:latest ./URBAN/water-connection/ws-calculator/
   docker push gcr.io/digit-water-service/ws-calculator:latest
   ```

### Phase 6: Pub/Sub Setup (Kafka Alternative)

1. **Create Pub/Sub Topics**
   ```bash
   # Water service topics
   gcloud pubsub topics create save-ws-connection
   gcloud pubsub topics create update-ws-connection
   gcloud pubsub topics create update-ws-workflow
   gcloud pubsub topics create create-meter-reading
   gcloud pubsub topics create ws-generate-demand
   gcloud pubsub topics create ws-demand-saved
   gcloud pubsub topics create ws-demand-failure
   gcloud pubsub topics create save-ws-meter
   gcloud pubsub topics create bill-generation

   # Common topics
   gcloud pubsub topics create egov-core-notification-sms
   gcloud pubsub topics create persist-user-events-async
   gcloud pubsub topics create egov-collection-payment-create
   gcloud pubsub topics create egov-collection-payment-cancel
   ```

2. **Create Subscriptions**
   ```bash
   gcloud pubsub subscriptions create ws-services-subscription --topic=save-ws-connection
   gcloud pubsub subscriptions create ws-calculator-subscription --topic=ws-generate-demand
   ```

### Phase 7: Kubernetes Deployments

#### 7.1 Create Namespace
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: digit-water
```

#### 7.2 Create ConfigMaps
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: digit-config
  namespace: digit-water
data:
  # Database configuration
  DB_HOST: "10.x.x.x"  # Cloud SQL private IP
  DB_PORT: "5432"
  
  # Redis configuration
  REDIS_HOST: "10.x.x.x"  # Redis private IP
  REDIS_PORT: "6379"
  
  # Pub/Sub configuration
  PUBSUB_PROJECT_ID: "digit-water-service"
  
  # Service URLs
  EGOV_MDMS_HOST: "http://egov-mdms-service:8080"
  PROPERTY_SERVICE_HOST: "http://property-service:8080"
  EGOV_IDGEN_HOST: "http://egov-idgen:8080"
  EGOV_PERSISTER_HOST: "http://egov-persister:8080"
  EGOV_FILESTORE_HOST: "http://egov-filestore:8080"
  PDF_SERVICE_HOST: "http://pdf-service:8080"
  BILLING_SERVICE_HOST: "http://billing-service:8080"
  EGOV_USER_HOST: "http://egov-user:8080"
  EGOV_WORKFLOW_HOST: "http://egov-workflow-v2:8080"
```

#### 7.3 Create Secrets
```yaml
# secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: digit-secrets
  namespace: digit-water
type: Opaque
data:
  DB_USERNAME: <base64-encoded-username>
  DB_PASSWORD: <base64-encoded-password>
  REDIS_PASSWORD: <base64-encoded-redis-password>
```

#### 7.4 Deploy ws-services
```yaml
# ws-services-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ws-services
  namespace: digit-water
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ws-services
  template:
    metadata:
      labels:
        app: ws-services
    spec:
      containers:
      - name: ws-services
        image: gcr.io/digit-water-service/ws-services:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: digit-config
              key: DB_HOST
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: digit-secrets
              key: DB_USERNAME
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: digit-secrets
              key: DB_PASSWORD
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

#### 7.5 Deploy ws-calculator
```yaml
# ws-calculator-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ws-calculator
  namespace: digit-water
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ws-calculator
  template:
    metadata:
      labels:
        app: ws-calculator
    spec:
      containers:
      - name: ws-calculator
        image: gcr.io/digit-water-service/ws-calculator:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: digit-config
              key: DB_HOST
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: digit-secrets
              key: DB_USERNAME
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: digit-secrets
              key: DB_PASSWORD
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
```

#### 7.6 Create Services
```yaml
# services.yaml
apiVersion: v1
kind: Service
metadata:
  name: ws-services-service
  namespace: digit-water
spec:
  selector:
    app: ws-services
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: ws-calculator-service
  namespace: digit-water
spec:
  selector:
    app: ws-calculator
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

#### 7.7 Create Ingress
```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: digit-water-ingress
  namespace: digit-water
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "digit-water-ip"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - water.digit.example.com
    secretName: digit-water-tls
  rules:
  - host: water.digit.example.com
    http:
      paths:
      - path: /ws-services
        pathType: Prefix
        backend:
          service:
            name: ws-services-service
            port:
              number: 80
      - path: /ws-calculator
        pathType: Prefix
        backend:
          service:
            name: ws-calculator-service
            port:
              number: 80
```

### Phase 8: Monitoring and Logging Setup

1. **Enable Cloud Monitoring**
   ```bash
   gcloud services enable monitoring.googleapis.com
   ```

2. **Create Monitoring Dashboard**
   - CPU and Memory usage
   - Request latency and throughput
   - Error rates
   - Database performance metrics

3. **Set up Alerting**
   - High CPU usage (>80%)
   - High memory usage (>90%)
   - Error rate >5%
   - Database connection failures

### Phase 9: Security Configuration

1. **Network Security**
   - Configure VPC with private subnets
   - Set up firewall rules
   - Enable Cloud Armor for DDoS protection

2. **Application Security**
   - Use Cloud KMS for secret management
   - Enable workload identity
   - Configure RBAC for Kubernetes

3. **Data Security**
   - Enable database encryption at rest
   - Use SSL/TLS for all connections
   - Regular security updates

## Deployment Commands

### Deploy to GKE
```bash
# Apply all configurations
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secrets.yaml
kubectl apply -f ws-services-deployment.yaml
kubectl apply -f ws-calculator-deployment.yaml
kubectl apply -f services.yaml
kubectl apply -f ingress.yaml
```

### Verify Deployment
```bash
# Check pod status
kubectl get pods -n digit-water

# Check service status
kubectl get services -n digit-water

# Check ingress status
kubectl get ingress -n digit-water

# View logs
kubectl logs -f deployment/ws-services -n digit-water
kubectl logs -f deployment/ws-calculator -n digit-water
```

## Maintenance and Operations

### Regular Tasks
1. **Database Maintenance**
   - Regular backups
   - Performance optimization
   - Security updates

2. **Application Updates**
   - Rolling deployments
   - Health checks
   - Rollback procedures

3. **Monitoring**
   - Daily health checks
   - Weekly performance reviews
   - Monthly security audits

### Scaling Guidelines
- **Horizontal Scaling**: Increase replica count based on load
- **Vertical Scaling**: Increase resource limits for individual pods
- **Database Scaling**: Upgrade Cloud SQL instance tier
- **Auto-scaling**: Configure HPA and VPA for automatic scaling

## Troubleshooting

### Common Issues
1. **Database Connection Issues**
   - Check Cloud SQL connectivity
   - Verify firewall rules
   - Check credentials

2. **Service Communication Issues**
   - Verify service discovery
   - Check network policies
   - Review DNS configuration

3. **Performance Issues**
   - Monitor resource usage
   - Check database performance
   - Review application logs

### Support Contacts
- **GCP Support**: Available through GCP Console
- **DIGIT Community**: GitHub issues and discussions
- **Documentation**: DIGIT official documentation
