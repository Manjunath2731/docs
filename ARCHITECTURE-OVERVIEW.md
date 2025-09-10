# DIGIT Water Service - Architecture Overview

## System Architecture

The DIGIT Water Service is a microservices-based application designed for municipal water connection management. The system follows a cloud-native architecture pattern with clear separation of concerns.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                            │
├─────────────────────────────────────────────────────────────────┤
│  Web Frontend  │  Mobile App  │  API Gateway  │  Third-party   │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Load Balancer / Ingress                     │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster (GKE)                    │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │ ws-services │  │ws-calculator│  │   Other     │            │
│  │   (3 pods)  │  │   (3 pods)  │  │  Services   │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Data & Message Layer                        │
├─────────────────────────────────────────────────────────────────┤
│  PostgreSQL    │  Redis Cache  │  Pub/Sub      │  Cloud       │
│  (Cloud SQL)   │  (Memorystore)│  (Kafka Alt)  │  Storage     │
└─────────────────────────────────────────────────────────────────┘
```

## Service Dependencies

### Core Water Services
1. **ws-services** - Water connection management
2. **ws-calculator** - Billing and demand calculation

### Supporting Services
1. **egov-mdms-service** - Master data management
2. **property-service** - Property management
3. **egov-idgen** - ID generation
4. **egov-persister** - Data persistence
5. **egov-filestore** - File storage
6. **pdf-service** - PDF generation
7. **billing-service** - Billing and payments
8. **egov-user** - User management
9. **egov-workflow-v2** - Workflow management

## Data Flow

### Water Connection Application Flow
```
1. Citizen submits application → ws-services
2. ws-services validates data → property-service, egov-mdms
3. ws-services generates application ID → egov-idgen
4. ws-services saves application → egov-persister
5. ws-services triggers workflow → egov-workflow-v2
6. Workflow processes application → ws-services
7. ws-services calculates fees → ws-calculator
8. ws-calculator generates demand → billing-service
9. Citizen pays fees → billing-service
10. Payment updates demand → ws-calculator
11. Application approved → ws-services
12. Connection activated → ws-services
```

### Billing Flow
```
1. Meter reading submitted → ws-calculator
2. ws-calculator calculates consumption → ws-calculator
3. ws-calculator generates demand → billing-service
4. Bill generated → billing-service
5. Payment made → billing-service
6. Payment updates demand → ws-calculator
7. Notification sent → egov-user
```

## Technology Stack

### Backend
- **Language**: Java 17
- **Framework**: Spring Boot 3.2.2
- **Build Tool**: Maven
- **Database**: PostgreSQL 14
- **Cache**: Redis 6.x
- **Message Queue**: Google Cloud Pub/Sub (Kafka alternative)

### Infrastructure
- **Container Platform**: Docker
- **Orchestration**: Kubernetes (GKE)
- **Cloud Provider**: Google Cloud Platform
- **Load Balancer**: Google Cloud Load Balancing
- **DNS**: Google Cloud DNS
- **Monitoring**: Google Cloud Monitoring
- **Logging**: Google Cloud Logging

### Security
- **Authentication**: JWT tokens
- **Authorization**: Role-based access control
- **Encryption**: TLS 1.3
- **Secrets Management**: Google Cloud KMS
- **Network Security**: VPC, firewall rules

## Scalability Design

### Horizontal Scaling
- **Stateless Services**: All services are stateless
- **Load Balancing**: Automatic load distribution
- **Auto-scaling**: HPA and VPA enabled
- **Database Scaling**: Read replicas for read-heavy workloads

### Vertical Scaling
- **Resource Limits**: Configurable CPU and memory limits
- **Database Scaling**: Upgrade instance tiers
- **Cache Scaling**: Increase Redis memory

## High Availability

### Multi-Zone Deployment
- **GKE Cluster**: Deployed across multiple zones
- **Database**: Multi-zone Cloud SQL
- **Load Balancer**: Global load balancing

### Disaster Recovery
- **Backup Strategy**: Automated daily backups
- **Recovery Time**: < 4 hours
- **Recovery Point**: < 1 hour

## Security Architecture

### Network Security
- **VPC**: Private network isolation
- **Firewall Rules**: Restrictive inbound/outbound rules
- **Private Google Access**: Database access without external IP

### Application Security
- **Authentication**: JWT-based authentication
- **Authorization**: Role-based access control
- **Input Validation**: Comprehensive input sanitization
- **SQL Injection Prevention**: Parameterized queries

### Data Security
- **Encryption at Rest**: Database and storage encryption
- **Encryption in Transit**: TLS for all communications
- **Secrets Management**: Centralized secret storage

## Monitoring and Observability

### Application Monitoring
- **Health Checks**: Kubernetes liveness and readiness probes
- **Metrics**: Custom application metrics
- **Tracing**: Distributed tracing for request flows

### Infrastructure Monitoring
- **Resource Usage**: CPU, memory, disk, network
- **Database Performance**: Query performance, connections
- **Cache Performance**: Hit rates, memory usage

### Logging
- **Centralized Logging**: All logs sent to Cloud Logging
- **Log Levels**: DEBUG, INFO, WARN, ERROR
- **Log Retention**: 30 days for production

## Performance Characteristics

### Expected Performance
- **Response Time**: < 200ms for 95% of requests
- **Throughput**: 1000+ requests per second
- **Availability**: 99.9% uptime
- **Concurrent Users**: 10,000+ simultaneous users

### Performance Optimization
- **Caching**: Redis for frequently accessed data
- **Database Optimization**: Indexed queries, connection pooling
- **CDN**: Static content delivery
- **Compression**: Gzip compression for responses

## Deployment Architecture

### Development Environment
- **Single Zone**: us-central1-a
- **Minimal Resources**: 2 nodes, basic monitoring
- **Purpose**: Development and testing

### Production Environment
- **Multi-Zone**: us-central1-a, us-central1-b, us-central1-c
- **High Resources**: 3-5 nodes, full monitoring
- **Purpose**: Live production for small to medium municipalities

### High Availability Production
- **Multi-Region**: us-central1, us-east1
- **Maximum Resources**: 5-10 nodes, comprehensive monitoring
- **Purpose**: Live production for large municipalities

## API Design

### RESTful APIs
- **Base URL**: `https://water.digit.example.com`
- **Versioning**: URL-based versioning (`/v1/`, `/v2/`)
- **Content Type**: JSON
- **Authentication**: Bearer token

### Key Endpoints
- **POST** `/ws-services/wc/_create` - Create water connection
- **PUT** `/ws-services/wc/_update` - Update water connection
- **GET** `/ws-services/wc/_search` - Search water connections
- **POST** `/ws-calculator/waterCalculator/_calculate` - Calculate bill
- **GET** `/ws-calculator/meterConnection/_search` - Search meter readings

## Data Model

### Core Entities
- **WaterConnection**: Main water connection entity
- **MeterReading**: Water meter reading data
- **Demand**: Billing demand information
- **Bill**: Generated bill details
- **Payment**: Payment information

### Relationships
- WaterConnection → MeterReading (1:many)
- WaterConnection → Demand (1:many)
- Demand → Bill (1:many)
- Bill → Payment (1:many)

## Integration Points

### External Systems
- **Payment Gateway**: For processing payments
- **SMS Gateway**: For sending notifications
- **Email Service**: For sending notifications
- **Document Management**: For storing documents

### Internal Systems
- **Property Tax System**: For property validation
- **User Management**: For user authentication
- **Workflow Engine**: For process management
- **Notification Service**: For sending alerts

## Future Enhancements

### Planned Features
- **Mobile App**: Native mobile application
- **Real-time Notifications**: WebSocket support
- **Advanced Analytics**: Business intelligence dashboard
- **API Gateway**: Centralized API management

### Scalability Improvements
- **Microservices**: Further service decomposition
- **Event Sourcing**: Event-driven architecture
- **CQRS**: Command Query Responsibility Segregation
- **GraphQL**: Flexible API querying

## Conclusion

The DIGIT Water Service architecture provides a robust, scalable, and maintainable solution for municipal water connection management. The cloud-native design ensures high availability, security, and performance while maintaining cost-effectiveness.

The modular architecture allows for easy extension and modification to meet changing requirements, making it suitable for various municipal contexts and scales.
