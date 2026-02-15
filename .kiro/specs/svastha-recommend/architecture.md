# Svastha-Recommend: System Architecture

## 1. Overview

Svastha-Recommend implements a privacy-first, edge-computing architecture that processes sensitive medical data on user devices while leveraging cloud services for recommendation generation. The system uses AWS services for scalability and reliability while maintaining zero cloud storage of raw medical data.

## 2. High-Level Architecture

### 2.1 Three-Tier Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    EDGE LAYER (Mobile Device)               │
│  • Prescription Capture                                     │
│  • On-Device OCR & Entity Extraction                        │
│  • PII Masking Engine                                       │
│  • Anonymized Health Persona Generation                     │
│  • Encrypted Persona ID & Vector Generation                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
                    TLS 1.3 Encrypted
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    CLOUD LAYER (AWS)                        │
│  • Amazon API Gateway (Entry Point)                         │
│  • AWS Lambda (Serverless Compute)                          │
│  • Amazon Aurora PostgreSQL (Unified Data Tier)             │
│  • Amazon ElastiCache Redis (Real-time Cache)               │
│  • Amazon Bedrock GenAI (Nudge Generation)                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    VALUE DELIVERY LAYER                     │
│  • Hyper-Personalized Shopping Nudges                       │
│  • ONDC Integration                                         │
│  • Vendor Intelligence                                      │
└─────────────────────────────────────────────────────────────┘
```

## 3. Edge Intelligence Layer (Mobile Device)

### 3.1 Edge Processing Pipeline

The edge layer ensures complete privacy by processing all medical data locally on the user's device.

#### 3.1.1 Prescription Capture
- **Input Methods**: Camera capture, file upload, ABDM/ABHA integration
- **Supported Formats**: JPEG, PNG, PDF
- **Processing**: Local image preprocessing and optimization

#### 3.1.2 On-Device OCR & Entity Extraction
- **OCR Engine**: Tesseract OCR for text extraction
- **Entity Extraction**: 
  - Small Language Models (GLiNER/MedSpaCy)
  - ONNX Runtime for cross-platform inference
  - Extracts: Conditions, medications, lab values, dietary restrictions
- **Performance**: <3 seconds on mid-range devices

#### 3.1.3 PII Masking Engine
The PII Masking Engine implements a multi-stage anonymization process:

**Stage 1: Redaction**
- Remove: Names, hospital IDs, exact addresses, phone numbers
- Method: Pattern matching and entity recognition

**Stage 2: Generalization**
- Age: Convert to 5-year bins (e.g., 35 → "30-35")
- Location: Convert to 3-digit pincode prefix (e.g., 560001 → "560")
- Dates: Convert to month/year only

**Stage 3: Local Differential Privacy (LDP)**
- Add controlled Laplace noise to health parameters
- Privacy budget: ε = 1.0
- Maintains utility while ensuring privacy

**Stage 4: k-Anonymity Validation**
- Ensure profile matches ≥5 other users
- If not met, apply additional generalization

#### 3.1.4 Anonymized Health Persona Generation
- **Persona Mapping**: Map masked profile to public taxonomy
- **Persona ID Format**: `{category}-{subcategory}-{year}-{hash}`
  - Example: `LS-HF-MC-2024-A3F9`
- **Vector Generation**: Create embedding vector for semantic matching

#### 3.1.5 Encrypted Persona ID & Final Action
- **Encryption**: AES-256 encryption with device-specific keys
- **Storage**: Secure local storage (iOS Keychain, Android Keystore)
- **Transmission**: Only encrypted persona ID sent to cloud
- **Permanent Image Deletion**: Original prescription images deleted immediately after processing

### 3.2 Zero-Trust Transfer
- **Protocol**: TLS 1.3 with certificate pinning
- **Payload**: Only encrypted persona ID and vector (no medical data)
- **Authentication**: JWT tokens with short expiry (15 minutes)

## 4. Cloud Intelligence Layer (AWS)

### 4.1 Entry Point

#### Amazon API Gateway
- **Role**: Single entry point for all API requests
- **Features**:
  - Request validation and throttling
  - JWT authentication
  - Rate limiting (per user/IP)
  - CORS configuration
  - API versioning
- **Security**: WAF integration for DDoS protection

### 4.2 Compute Layer

#### AWS Lambda (Serverless Functions)
- **Architecture**: Event-driven, serverless compute
- **Functions**:
  1. **Recommendation Handler**: Processes recommendation requests
  2. **Persona Matcher**: Matches personas to guidelines
  3. **Nudge Generator**: Creates personalized nudges
  4. **Vendor Analytics**: Aggregates demand data
- **Benefits**:
  - Auto-scaling based on demand
  - Pay-per-use pricing
  - No server management
  - Built-in fault tolerance

### 4.3 Unified Data Tier

#### Amazon Aurora PostgreSQL
The unified persistence layer stores all structured data with advanced capabilities:

**Database Capabilities**:

1. **SQL: ICMR-NIN Guidelines**
   - Structured storage of dietary guidelines
   - Versioned guideline data
   - Fast querying by persona type
   - Schema:
     ```sql
     guidelines (
       id, guideline_code, source, parameter,
       limit_value, unit, applicable_personas,
       recommendation_text, version, created_at
     )
     ```

2. **JSONB: ONDC Product Metadata**
   - Flexible schema for product catalog
   - Nested nutritional information
   - Health tags and compliance markers
   - Schema:
     ```sql
     products (
       id, product_id, name, vendor_id,
       metadata JSONB, health_tags TEXT[],
       nutritional_info JSONB, created_at
     )
     ```

3. **pgvector: Persona Matching**
   - Vector embeddings for semantic similarity
   - Fast nearest-neighbor search
   - Persona-to-product matching
   - Schema:
     ```sql
     persona_vectors (
       persona_id, embedding vector(384),
       metadata JSONB, created_at
     )
     ```

**Aurora Features**:
- **High Availability**: Multi-AZ deployment with automatic failover
- **Performance**: Up to 5x faster than standard PostgreSQL
- **Scalability**: Auto-scaling storage (10GB to 128TB)
- **Backup**: Continuous backup to S3, point-in-time recovery
- **Security**: Encryption at rest and in transit

**Data Access Patterns**:
- **pgvector: Semantic Matching**: Find similar personas and products using vector similarity
- **JSONB: Retail Knowledge**: Query flexible product metadata with JSON operators
- **Real-time Nudge Cache**: ElastiCache Redis for frequently accessed recommendations

### 4.4 Cache Layer

#### Amazon ElastiCache: Redis
- **Purpose**: High-performance caching for real-time responses
- **Cached Data**:
  - Frequently accessed guidelines
  - Popular product recommendations
  - Persona-to-guideline mappings
  - Session data
- **Cache Strategy**:
  - TTL: 1 hour for recommendations
  - Cache-aside pattern
  - Cache warming for popular personas
- **Performance**: Sub-millisecond latency

### 4.5 AI & Logic Layer

#### Amazon Bedrock: GenAI Nudges
- **Purpose**: Generate personalized, contextual health nudges
- **Model**: Foundation models (Claude, Titan) for natural language generation
- **Input**: Persona ID, search query, applicable guidelines
- **Output**: Personalized nudge text with guideline citations
- **Features**:
  - Context-aware generation
  - Multi-language support
  - Tone customization (suggestive, not restrictive)
  - A/B testing for effectiveness

**Nudge Generation Pipeline**:
1. Retrieve applicable guidelines from Aurora
2. Get product context from search query
3. Generate nudge using Bedrock with prompt template
4. Cache generated nudge in Redis
5. Return nudge with guideline citations

## 5. Value Delivery Layer

### 5.1 Hyper-Personalized Shopping Nudges
- **Delivery**: Real-time injection into ONDC buyer apps
- **Format**: Banner with health tip + recommended products
- **Personalization**: Based on persona, search context, location
- **Effectiveness Tracking**: A/B testing, click-through rates

### 5.2 ONDC Integration
- **Integration Type**: API-based integration with ONDC buyer apps
- **Capabilities**:
  - Search interception
  - Product reranking based on health scores
  - Nudge injection
  - Order tracking (optional)

### 5.3 Vendor Intelligence
- **Aggregation**: Weekly batch jobs aggregate anonymized persona data by pincode
- **Insights**:
  - Persona distribution by area
  - Trending health needs
  - Demand forecasts for product categories
- **Delivery**: Vendor dashboard with actionable recommendations

## 6. Data Flow Architecture

### 6.1 User Onboarding Flow

```
User → Capture Prescription → On-Device OCR → Entity Extraction
  → PII Masking → Persona Generation → Encrypted Storage
  → (Original Image Deleted)
```

### 6.2 Recommendation Request Flow

```
User Search → Device sends {encrypted_persona_id, query, pincode}
  → API Gateway → Lambda → Check Redis Cache
  → (Cache Miss) → Query Aurora (Guidelines + Products)
  → Bedrock GenAI (Generate Nudge) → Cache in Redis
  → Return {nudge, recommended_products} → Display to User
```

### 6.3 Vendor Intelligence Flow

```
Weekly Batch Job → Aggregate Personas by Pincode (Aurora)
  → Calculate Distribution → Identify Trends
  → Generate Forecast → Store in Aurora
  → Vendor Dashboard API → Display Insights
```

## 7. Security Architecture

### 7.1 Edge Security
- **Data Encryption**: AES-256 for local storage
- **Key Management**: Device-specific keys in secure enclave
- **Biometric Protection**: Optional biometric authentication
- **Secure Deletion**: Cryptographic erasure of original images

### 7.2 Transport Security
- **Protocol**: TLS 1.3 with perfect forward secrecy
- **Certificate Pinning**: Prevent MITM attacks
- **Mutual TLS**: Optional for high-security scenarios

### 7.3 Cloud Security
- **Authentication**: JWT with short expiry, OAuth 2.0 for ABDM
- **Authorization**: Role-based access control (RBAC)
- **API Security**: Rate limiting, request validation, WAF
- **Data Encryption**: Encryption at rest (Aurora, S3) and in transit
- **Network Security**: VPC isolation, security groups, NACLs
- **Audit Logging**: CloudTrail for all API calls

### 7.4 Compliance
- **DPDP Act 2023**: Edge-first architecture ensures data residency
- **GDPR**: Privacy by design, explicit consent, data minimization
- **EU AI Act**: Transparent AI, verified data sources, limited risk classification
- **HIPAA-Ready**: Encryption, access controls, audit trails (if needed)

## 8. Scalability Architecture

### 8.1 Horizontal Scaling
- **Lambda**: Auto-scales based on request volume
- **Aurora**: Read replicas for read-heavy workloads
- **ElastiCache**: Redis cluster mode for distributed caching
- **API Gateway**: Handles millions of requests per second

### 8.2 Performance Optimization
- **Edge Processing**: Reduces server load by 90%
- **Caching Strategy**: 80% cache hit rate for recommendations
- **Database Indexing**: B-tree indexes on persona_id, product_id
- **Vector Indexing**: HNSW index for fast similarity search
- **Connection Pooling**: RDS Proxy for efficient database connections

### 8.3 Cost Optimization
- **Serverless**: Pay only for actual compute time
- **Aurora Serverless**: Auto-pause during low activity
- **S3 Intelligent Tiering**: Automatic cost optimization for guidelines
- **Reserved Capacity**: For predictable baseline load

## 9. Monitoring & Observability

### 9.1 Application Monitoring
- **CloudWatch**: Metrics, logs, alarms
- **X-Ray**: Distributed tracing for Lambda functions
- **Custom Metrics**: Recommendation latency, cache hit rate, nudge effectiveness

### 9.2 Performance Monitoring
- **Lambda Insights**: Function performance and errors
- **Aurora Performance Insights**: Database query performance
- **ElastiCache Metrics**: Cache hit rate, evictions, latency

### 9.3 Security Monitoring
- **GuardDuty**: Threat detection
- **CloudTrail**: API audit logs
- **Config**: Compliance monitoring
- **Security Hub**: Centralized security findings

### 9.4 Alerting
- **CloudWatch Alarms**: Automated alerts for anomalies
- **SNS**: Multi-channel notifications (email, SMS, Slack)
- **PagerDuty Integration**: On-call incident management

## 10. Disaster Recovery & Business Continuity

### 10.1 Backup Strategy
- **Aurora**: Continuous backup to S3, 35-day retention
- **Point-in-Time Recovery**: Restore to any second within retention period
- **Cross-Region Replication**: Optional for global availability

### 10.2 High Availability
- **Multi-AZ Deployment**: Aurora spans multiple availability zones
- **Automatic Failover**: <30 seconds for Aurora
- **Lambda**: Inherently multi-AZ
- **ElastiCache**: Multi-AZ with automatic failover

### 10.3 Disaster Recovery Plan
- **RTO (Recovery Time Objective)**: <1 hour
- **RPO (Recovery Point Objective)**: <5 minutes
- **Backup Testing**: Monthly restore drills
- **Runbook**: Documented recovery procedures

## 11. Deployment Architecture

### 11.1 Infrastructure as Code
- **AWS CDK**: Define infrastructure in TypeScript/Python
- **CloudFormation**: Automated stack deployment
- **Version Control**: Infrastructure code in Git

### 11.2 CI/CD Pipeline
- **GitHub Actions**: Automated testing and deployment
- **Stages**: Build → Test → Deploy to Dev → Deploy to Prod
- **Blue-Green Deployment**: Zero-downtime deployments
- **Rollback**: Automated rollback on failure

### 11.3 Environment Strategy
- **Development**: Isolated environment for testing
- **Staging**: Production-like environment for final validation
- **Production**: Live environment with full monitoring

## 12. Technology Stack Summary

### Edge Layer
- **OCR**: Tesseract OCR
- **ML Models**: GLiNER, MedSpaCy (ONNX Runtime)
- **Encryption**: AES-256, TLS 1.3
- **Storage**: iOS Keychain, Android Keystore

### Cloud Layer
- **Compute**: AWS Lambda (Node.js, Python)
- **API**: Amazon API Gateway
- **Database**: Amazon Aurora PostgreSQL (with pgvector)
- **Cache**: Amazon ElastiCache (Redis)
- **AI**: Amazon Bedrock (GenAI)
- **Storage**: Amazon S3
- **Monitoring**: CloudWatch, X-Ray, GuardDuty

### Integration Layer
- **ABDM/ABHA**: OAuth 2.0 integration
- **ONDC**: REST API integration
- **Guidelines**: ICMR-NIN, EFSA (stored in Aurora)

## 13. Architecture Principles

1. **Privacy by Design**: Medical data never leaves the device
2. **Zero-Trust Architecture**: Verify every request, encrypt everything
3. **Serverless-First**: Minimize operational overhead
4. **Event-Driven**: Asynchronous processing where possible
5. **Microservices**: Loosely coupled, independently deployable
6. **API-First**: Well-defined interfaces between components
7. **Observability**: Comprehensive monitoring and logging
8. **Resilience**: Fault-tolerant with graceful degradation
9. **Scalability**: Horizontal scaling for all components
10. **Cost-Effective**: Pay-per-use, optimize for efficiency

## 14. Future Architecture Enhancements

### 14.1 Global Expansion
- **Multi-Region Deployment**: Aurora Global Database
- **Edge Locations**: CloudFront for static assets
- **Localization**: Region-specific guidelines and languages

### 14.2 Advanced AI
- **Federated Learning**: Improve models without centralizing data
- **Real-Time Personalization**: Streaming analytics with Kinesis
- **Predictive Analytics**: Forecast health trends

### 14.3 Enhanced Integration
- **Telemedicine**: Integration with virtual consultation platforms
- **Wearables**: Sync with fitness trackers for holistic profiles
- **Pharmacy**: Direct prescription fulfillment integration

## 15. Architecture Decision Records (ADRs)

### ADR-001: Edge-First Processing
**Decision**: Process all medical data on user devices
**Rationale**: Ensures privacy, reduces compliance burden, minimizes cloud costs
**Trade-offs**: Requires capable devices, limits server-side analytics

### ADR-002: AWS Serverless Architecture
**Decision**: Use Lambda, Aurora Serverless, and managed services
**Rationale**: Reduces operational overhead, auto-scales, pay-per-use
**Trade-offs**: Vendor lock-in, cold start latency

### ADR-003: Unified Data Tier with Aurora PostgreSQL
**Decision**: Use single Aurora PostgreSQL with SQL, JSONB, and pgvector
**Rationale**: Simplifies architecture, reduces data movement, leverages advanced features
**Trade-offs**: Single point of failure (mitigated by Multi-AZ)

### ADR-004: Amazon Bedrock for Nudge Generation
**Decision**: Use GenAI for personalized nudge text
**Rationale**: Natural language generation, context-aware, multi-language support
**Trade-offs**: Cost per request, latency (mitigated by caching)

### ADR-005: Redis for Real-Time Caching
**Decision**: Use ElastiCache Redis for frequently accessed data
**Rationale**: Sub-millisecond latency, reduces database load
**Trade-offs**: Cache invalidation complexity, additional cost
