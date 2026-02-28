# AWS AI Architecture Design: BharatChef AI Coach

## Executive Summary

This document outlines the AWS-based architecture for BharatChef AI Coach, a cloud-native platform that evaluates cooking skills through AI-powered video analysis. The architecture leverages AWS managed services for scalability, cost-efficiency, and performance, targeting ₹2 per evaluation with 5-minute processing time for 10-minute videos.

## Architecture Overview

The system follows a serverless, event-driven architecture with the following key components:

1. **Mobile Application Layer** - Android app for video capture and results display
2. **API Gateway Layer** - RESTful API for client-server communication
3. **Video Processing Pipeline** - Multi-stage CV and AI analysis workflow
4. **Storage Layer** - Secure video and data storage
5. **AI/ML Services** - Computer vision and LLM-based coaching
6. **Data Analytics Layer** - Performance tracking and progression analysis

## Core AWS Services

### 1. Frontend & API Layer

**Amazon API Gateway**
- RESTful API endpoints for mobile app communication
- WebSocket API for real-time upload progress
- Request validation and throttling
- API key management for authentication

**AWS Amplify**
- Mobile SDK for Android app development
- Authentication integration with Cognito
- Simplified API calls and file uploads
- Offline sync capabilities

**Amazon Cognito**
- User authentication and authorization
- User pools for trainee accounts
- Identity pools for temporary AWS credentials
- Role-based access control (trainee vs admin)

### 2. Video Storage & Management

**Amazon S3**
- Primary storage for trainee and expert videos
- Bucket structure:
  - `bharatchef-trainee-videos/` - Trainee uploads
  - `bharatchef-expert-videos/` - Benchmark videos
  - `bharatchef-processed-results/` - Analysis outputs
- S3 Transfer Acceleration for faster uploads
- Lifecycle policies for cost optimization (move to Glacier after 90 days)
- Server-side encryption (SSE-S3)
- Versioning enabled for expert videos

**Amazon CloudFront**
- CDN for expert video streaming
- Reduced latency for video playback
- Signed URLs for secure access

### 3. Video Processing Pipeline

**AWS Step Functions**
- Orchestrates the entire evaluation workflow
- Manages state transitions between processing stages
- Error handling and retry logic
- Parallel execution of independent analysis tasks

**AWS Lambda**
- Serverless compute for lightweight processing tasks:
  - Video validation and metadata extraction
  - Result aggregation and scoring calculation
  - Notification triggers
  - API request handlers
- Python 3.11 runtime
- Memory: 512MB - 3GB based on task
- Timeout: 15 minutes max

**Amazon ECS (Elastic Container Service) with Fargate**
- Container-based compute for heavy CV workloads:
  - Action recognition (SlowFast/I3D models)
  - Object detection (YOLOv8)
  - Visual quality analysis (ResNet)
  - Heat and flame detection
- Spot instances for cost optimization
- Auto-scaling based on queue depth
- GPU-enabled tasks (g4dn instances) for model inference

**Amazon SageMaker**
- Model hosting and inference:
  - Real-time endpoints for low-latency predictions
  - Batch transform for bulk processing
  - Model versioning and A/B testing
  - Automatic model scaling
- Pre-trained models:
  - SlowFast for action recognition
  - YOLOv8 for object/ingredient detection
  - ResNet-50 for visual embeddings
- Custom models for flame detection

**Amazon Rekognition**
- Supplementary video analysis:
  - Scene detection
  - Object and activity detection
  - Content moderation (safety check)
- Cost-effective for basic video analysis

### 4. AI Coaching Layer

**Amazon Bedrock**
- Multimodal LLM for AI coaching feedback:
  - Claude 3 Sonnet for text generation
  - Multimodal understanding of video frames + CV outputs
  - Contextual, personalized feedback generation
- Prompt engineering for coaching tone and structure
- RAG (Retrieval Augmented Generation) with cooking knowledge base

**Amazon Titan Embeddings**
- Generate embeddings for cooking knowledge base
- Semantic search for relevant cooking tips

**Amazon OpenSearch Service**
- Vector database for RAG implementation
- Stores cooking techniques, common mistakes, best practices
- Fast semantic search for context retrieval

### 5. Data Management

**Amazon DynamoDB**
- NoSQL database for application data:
  - User profiles and authentication data
  - Dish metadata and expert video mappings
  - Evaluation results and scores
  - Skill progression tracking
- Tables:
  - `Users` - Trainee profiles
  - `Dishes` - Dish metadata and expert video references
  - `Evaluations` - Evaluation results with timestamps
  - `ActionSequences` - Detected actions with timestamps
  - `Feedback` - AI-generated coaching feedback
- Global secondary indexes for efficient queries
- Point-in-time recovery enabled
- DynamoDB Streams for real-time analytics

**Amazon RDS (PostgreSQL)**
- Relational database for complex analytics:
  - Skill progression analysis
  - Cross-user performance benchmarking
  - Reporting and dashboard queries
- Multi-AZ deployment for high availability
- Read replicas for analytics workloads

**AWS Glue**
- ETL jobs for data transformation
- Data catalog for analytics
- Crawlers for schema discovery

### 6. Message Queue & Event Processing

**Amazon SQS (Simple Queue Service)**
- Decouples video upload from processing
- Queues:
  - `video-processing-queue` - Main processing queue
  - `cv-analysis-queue` - Computer vision tasks
  - `ai-coaching-queue` - LLM feedback generation
  - `dlq-queue` - Dead letter queue for failed jobs
- FIFO queues for ordered processing where needed
- Visibility timeout: 15 minutes

**Amazon SNS (Simple Notification Service)**
- Pub/sub messaging for event notifications
- Topics:
  - `evaluation-complete` - Notify when results ready
  - `processing-failed` - Alert on failures
- Email and mobile push notifications

**Amazon EventBridge**
- Event-driven architecture coordination
- Rules for triggering workflows
- Integration with Step Functions

### 7. Monitoring & Logging

**Amazon CloudWatch**
- Centralized logging for all services
- Custom metrics for business KPIs:
  - Processing time per video
  - Cost per evaluation
  - Model accuracy metrics
  - API latency
- Alarms for operational issues
- Dashboards for real-time monitoring

**AWS X-Ray**
- Distributed tracing for request flows
- Performance bottleneck identification
- Service map visualization

**Amazon CloudWatch Logs Insights**
- Log analysis and querying
- Troubleshooting and debugging

### 8. Security & Compliance

**AWS IAM (Identity and Access Management)**
- Fine-grained access control
- Service roles for Lambda, ECS, SageMaker
- Least privilege principle
- MFA for admin access

**AWS KMS (Key Management Service)**
- Encryption key management
- Customer-managed keys for sensitive data
- Automatic key rotation

**AWS WAF (Web Application Firewall)**
- Protects API Gateway from common attacks
- Rate limiting and IP filtering
- SQL injection and XSS protection

**AWS Secrets Manager**
- Secure storage for API keys and credentials
- Automatic rotation of secrets
- Integration with Lambda and ECS

**AWS CloudTrail**
- Audit logging for all AWS API calls
- Compliance and governance
- Security analysis

### 9. Cost Optimization

**AWS Cost Explorer**
- Cost analysis and forecasting
- Budget alerts
- Resource optimization recommendations

**AWS Compute Optimizer**
- Right-sizing recommendations for compute resources

**S3 Intelligent-Tiering**
- Automatic cost optimization for storage

## Detailed Architecture Flow

### Video Upload Flow

1. **Trainee initiates upload** via Android app
2. **Cognito** authenticates user and provides temporary credentials
3. **S3 Transfer Acceleration** receives video upload with progress tracking
4. **Lambda** triggered on S3 upload event:
   - Validates video format, duration, file size
   - Extracts metadata (resolution, codec, duration)
   - Creates evaluation record in DynamoDB
5. **SQS message** sent to `video-processing-queue`
6. **SNS notification** sent to trainee confirming upload

### Video Processing Flow

1. **Step Functions** workflow initiated from SQS message
2. **Parallel Processing Stage**:
   
   a. **Action Recognition** (ECS Fargate with GPU):
      - Downloads video from S3
      - Runs SlowFast/I3D model inference
      - Detects cooking actions with timestamps
      - Stores results in DynamoDB `ActionSequences` table
   
   b. **Object/Ingredient Detection** (ECS Fargate with GPU):
      - Runs YOLOv8 model on video frames
      - Detects ingredients and utensils
      - Compares with expected ingredients from dish metadata
      - Stores detection results in DynamoDB
   
   c. **Heat & Flame Analysis** (ECS Fargate):
      - Analyzes side camera video (if available)
      - Brightness spectrum analysis for flame detection
      - Visual cue analysis for heat estimation
      - Stores heat control data in DynamoDB
   
   d. **Visual Quality Analysis** (SageMaker endpoint):
      - Extracts final dish frame
      - Generates ResNet embeddings
      - Compares with expert video embeddings
      - Calculates cosine similarity score

3. **Temporal Alignment Stage** (Lambda):
   - Retrieves expert video action sequence from DynamoDB
   - Applies DTW algorithm for sequence alignment
   - Identifies missing, extra, out-of-order steps
   - Calculates timing differences
   - Stores alignment results in DynamoDB

4. **Scoring Stage** (Lambda):
   - Aggregates all analysis results
   - Calculates category scores (step sequence, timing, technique, heat, visual, plating)
   - Applies weighted aggregation for overall performance score
   - Determines skill certification level
   - Stores scores in DynamoDB `Evaluations` table

5. **AI Coaching Stage** (Bedrock + Lambda):
   - Lambda retrieves all CV outputs and scores
   - Constructs multimodal prompt with:
     - Video frame samples
     - Action sequence data
     - Detection results
     - Scores and deviations
   - Bedrock Claude 3 generates personalized feedback:
     - Identifies top 3-5 critical mistakes
     - Explains what went wrong and why
     - Provides specific corrective actions
     - Maintains encouraging tone
   - OpenSearch RAG retrieves relevant cooking tips
   - Stores feedback in DynamoDB `Feedback` table

6. **Results Aggregation** (Lambda):
   - Compiles all results into dashboard format
   - Generates skill progression chart (if multiple attempts)
   - Stores final results in S3 as JSON
   - Updates DynamoDB with completion status

7. **Notification** (SNS):
   - Sends push notification to trainee
   - Email notification with results summary

### Results Retrieval Flow

1. **Trainee requests results** via API Gateway
2. **Lambda** retrieves data from DynamoDB and S3
3. **CloudFront** serves expert video for side-by-side comparison
4. **API Gateway** returns results to mobile app
5. **Mobile app** displays dashboard with scores, feedback, progression

## Data Models

### DynamoDB Tables

**Users Table**
```
PK: userId (String)
Attributes:
- email (String)
- name (String)
- createdAt (Timestamp)
- skillLevel (String)
```

**Dishes Table**
```
PK: dishId (String)
Attributes:
- name (String)
- expectedDuration (Number)
- ingredients (List)
- expectedSteps (List)
- expertVideoS3Key (String)
- expertActionSequence (List)
- createdAt (Timestamp)
```

**Evaluations Table**
```
PK: evaluationId (String)
SK: userId (String)
GSI: userId-timestamp-index
Attributes:
- dishId (String)
- traineeVideoS3Key (String)
- sideVideoS3Key (String, optional)
- overallScore (Number)
- categoryScores (Map)
- skillLevel (String)
- status (String) - PROCESSING, COMPLETED, FAILED
- createdAt (Timestamp)
- completedAt (Timestamp)
```

**ActionSequences Table**
```
PK: evaluationId (String)
Attributes:
- actions (List of Maps):
  - actionType (String)
  - startTime (Number)
  - endTime (Number)
  - confidence (Number)
- stepDeviations (List)
- sequenceAccuracy (Number)
```

**Feedback Table**
```
PK: evaluationId (String)
Attributes:
- mistakes (List of Maps):
  - description (String)
  - impact (String)
  - correction (String)
  - priority (Number)
- encouragement (String)
- nextSteps (List)
```

## Cost Estimation (per evaluation)

Based on 10-minute video processing:

| Service | Usage | Cost (₹) |
|---------|-------|----------|
| S3 Storage (30 days) | 500MB | 0.05 |
| S3 Transfer | 500MB upload | 0.02 |
| ECS Fargate (GPU) | 3 minutes | 0.80 |
| SageMaker Inference | 10 requests | 0.30 |
| Bedrock Claude 3 | 5K tokens | 0.40 |
| Lambda | 50 invocations | 0.05 |
| DynamoDB | 100 WCU, 200 RCU | 0.10 |
| API Gateway | 20 requests | 0.02 |
| CloudWatch | Logs & metrics | 0.05 |
| **Total** | | **₹1.79** |

Target: ₹2 per evaluation ✓

## Performance Optimization

### Processing Time Optimization

1. **Parallel Processing**: Run CV tasks concurrently using Step Functions parallel states
2. **Model Optimization**:
   - Use quantized models (INT8) for faster inference
   - Batch processing where applicable
   - Model caching in ECS containers
3. **GPU Acceleration**: Use g4dn.xlarge instances for CV workloads
4. **Async Processing**: Decouple upload from processing using SQS
5. **Caching**: Cache expert video embeddings and action sequences

### Cost Optimization

1. **Spot Instances**: Use Fargate Spot for 70% cost reduction
2. **Reserved Capacity**: Reserve SageMaker endpoints for predictable workloads
3. **S3 Lifecycle**: Move old videos to Glacier after 90 days
4. **Lambda Memory Tuning**: Right-size Lambda functions
5. **DynamoDB On-Demand**: Use on-demand billing for variable workloads

## Scalability

### Horizontal Scaling

- **ECS Auto Scaling**: Scale tasks based on SQS queue depth
- **SageMaker Auto Scaling**: Scale endpoints based on invocation rate
- **Lambda Concurrency**: Automatic scaling up to account limits
- **DynamoDB Auto Scaling**: Scale read/write capacity automatically

### Load Handling

- **API Gateway Throttling**: Protect backend from overload
- **SQS Buffering**: Queue videos during peak times
- **CloudFront Caching**: Reduce origin load for expert videos
- **Multi-Region**: Deploy in multiple regions for global scale (future)

## Security Architecture

### Data Encryption

- **In Transit**: TLS 1.2+ for all API calls
- **At Rest**: 
  - S3 SSE-S3 for videos
  - DynamoDB encryption at rest
  - RDS encryption at rest
  - KMS for sensitive data

### Access Control

- **Cognito**: User authentication and authorization
- **IAM Roles**: Service-to-service authentication
- **S3 Bucket Policies**: Restrict access to videos
- **API Gateway Authorizers**: Validate JWT tokens
- **VPC**: Isolate ECS tasks and RDS in private subnets

### Compliance

- **Data Residency**: Store data in India region (ap-south-1)
- **GDPR Compliance**: Support data deletion requests
- **Audit Logging**: CloudTrail for all API calls
- **Data Retention**: 90-day retention policy

## Disaster Recovery

### Backup Strategy

- **S3 Versioning**: Protect against accidental deletion
- **DynamoDB PITR**: Point-in-time recovery (35 days)
- **RDS Automated Backups**: Daily snapshots (7 days retention)
- **Cross-Region Replication**: Replicate critical data to secondary region

### High Availability

- **Multi-AZ Deployment**: RDS and ECS across multiple AZs
- **S3 Durability**: 99.999999999% durability
- **Lambda**: Inherently highly available
- **API Gateway**: Managed service with SLA

## Monitoring & Alerting

### Key Metrics

- **Processing Time**: Average time per evaluation
- **Success Rate**: Percentage of successful evaluations
- **Cost per Evaluation**: Track against ₹2 target
- **Model Accuracy**: CV model confidence scores
- **API Latency**: P50, P95, P99 latencies
- **Error Rate**: Failed evaluations and reasons

### Alerts

- **Processing Time > 7 minutes**: Investigate bottlenecks
- **Cost per Evaluation > ₹2.50**: Cost optimization needed
- **Error Rate > 5%**: System health issue
- **Queue Depth > 100**: Scaling needed
- **Model Confidence < 60%**: Model retraining needed

## Deployment Strategy

### CI/CD Pipeline

1. **AWS CodeCommit**: Source code repository
2. **AWS CodeBuild**: Build Docker images and Lambda packages
3. **AWS CodePipeline**: Orchestrate deployment
4. **AWS CodeDeploy**: Blue/green deployment for ECS
5. **CloudFormation/CDK**: Infrastructure as Code

### Environments

- **Development**: Minimal resources, manual approval
- **Staging**: Production-like, automated testing
- **Production**: Full scale, blue/green deployment

## Future Enhancements

1. **Real-time Feedback**: Stream processing for live coaching during cooking
2. **Multi-Language Support**: Internationalization for global reach
3. **Social Features**: Share achievements, compete with peers
4. **Advanced Analytics**: ML-based skill prediction and personalized learning paths
5. **Edge Processing**: On-device inference for privacy and offline capability
6. **Voice Coaching**: Audio feedback during cooking
7. **AR Overlays**: Augmented reality guidance on smartphone

## Conclusion

This AWS architecture provides a scalable, cost-effective, and secure platform for BharatChef AI Coach. By leveraging managed services and serverless technologies, the system achieves the target cost of ₹2 per evaluation while maintaining 5-minute processing time. The event-driven design ensures flexibility for future enhancements and global scale.
