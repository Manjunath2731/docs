# GCP Configuration Requirements for DIGIT Water Service Deployment

This document outlines what your organization needs to configure and provide for deploying the DIGIT Water Service on Google Cloud Platform (GCP).

## 1. GCP Account & Project Setup

### Required from Your Organization:

- **Google Cloud Account** with billing enabled
- **Project Name** (e.g., "digit-water-service")
- **Billing Account** linked to the project
- **Budget Alerts** configured for cost monitoring

### Actions Required:

```bash
# Your team needs to run these commands or provide access:
gcloud projects create digit-water-service --name="DIGIT Water Service"
gcloud config set project digit-water-service
```

## 2. API Access & Permissions

### Required APIs to Enable:

Your organization needs to enable these GCP APIs:

- `container.googleapis.com` (Kubernetes Engine)
- `sqladmin.googleapis.com` (Cloud SQL)
- `redis.googleapis.com` (Memorystore)
- `pubsub.googleapis.com` (Pub/Sub)
- `storage.googleapis.com` (Cloud Storage)
- `dns.googleapis.com` (Cloud DNS)
- `monitoring.googleapis.com` (Cloud Monitoring)
- `logging.googleapis.com` (Cloud Logging)

### Service Account Permissions:

Your organization needs to provide a service account with these roles:

- **Kubernetes Engine Admin**
- **Cloud SQL Admin**
- **Redis Admin**
- **Pub/Sub Admin**
- **Storage Admin**
- **DNS Admin**
- **Monitoring Admin**
- **Logging Admin**

## 3. Infrastructure Configuration

### Database Requirements:

- **Cloud SQL PostgreSQL 14** instance
- **Machine Type**: db-standard-4 (minimum)

### Redis Cache:

- **Cloud Memorystore Redis 6.x**
- **Memory**: 4GB (minimum)
- **Region**: Same as database

### Kubernetes Cluster:

- **GKE Cluster** with 3 nodes minimum
- **Machine Type**: e2-standard-8
- **Auto-scaling**: 2-10 nodes
- **Region**: asia-south1 (Mumbai)

## 4. Network & Security

### Security Requirements:

- **VPC Network** with private subnets
- **Private Google Access** enabled for database connectivity

## 5. Access Credentials to Provide

### Service Account Key:

- **JSON Service Account Key** file
- **Project ID**: digit-water-service
- **Region**: asia-south1

### API Endpoints:

Your organization needs to provide URLs for:

- Property Service
- User Management Service
- Billing Service
- Workflow Service
- MDMS Service

## 6. Backup & Disaster Recovery

### Backup Strategy:

- **Database Backups**: Daily automated backups
- **Application Data**: Cloud Storage backups
- **Configuration**: Version-controlled in Git

## 7. Compliance & Security

### Security Requirements:

- **Data Encryption** at rest and in transit
- **Access Logging** enabled
- **Audit Trails** for all operations
- **Compliance Standards** (as per Indian government requirements)

### Access Control:

- **IAM Policies** for team access
- **Role-based Access Control (RBAC)**
- **Multi-factor Authentication (MFA)**

### Network Configuration:

- **VPC Peering**: If connecting to government networks
- **Private Connectivity**: Use private Google access
- **Firewall Rules**: Comply with government security policies

### **Primary Region:**

- **asia-south1** (Mumbai) - Recommended for most Indian deployments
