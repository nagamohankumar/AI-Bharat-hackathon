# BharatChef AI Coach - AWS Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    MOBILE CLIENT LAYER                                   │
│                                                                                           │
│                          ┌──────────────────────────────┐                                │
│                          │   Trainee (Android App)      │                                │
│                          │   - Video Capture            │                                │
│                          │   - Results Dashboard        │                                │
│                          └──────────────┬───────────────┘                                │
└───────────────────────────────────────┼──────────────────────────────────────────────────┘
                                        │
                                        │ HTTPS/TLS
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              API & AUTHENTICATION LAYER                                  │
│                                                                                           │
│  ┌──────────────┐      ┌──────────────────┐      ┌─────────────────┐                   │
│  │  AWS Amplify │◄────►│  Amazon Cognito  │◄────►│   API Gateway   │                   │
│  │  (Mobile SDK)│      │  (User Auth)     │      │  (REST/WebSocket)│                   │
│  └──────────────┘      └──────────────────┘      └────────┬────────┘                   │
│                                                             │                             │
│                                                    ┌────────▼────────┐                   │
│                                                    │   AWS WAF       │                   │
│                                                    │ (API Protection)│                   │
│                                                    └────────┬────────┘                   │
└─────────────────────────────────────────────────────────┼──────────────────────────────┘
                                                            │
                                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                   STORAGE LAYER                                          │
│                                                                                           │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐                  │
│  │   Amazon S3      │    │   DynamoDB       │    │  RDS PostgreSQL  │                  │
│  │ - Trainee Videos │    │ - User Profiles  │    │ - Analytics DB   │                  │
│  │ - Expert Videos  │    │ - Evaluations    │    │ - Reporting      │                  │
│  │ - Results        │    │ - Scores         │    │                  │                  │
│  └────────┬─────────┘    └────────┬─────────┘    └──────────────────┘                  │
│           │                       │                                                      │
│           │              ┌────────▼─────────┐                                            │
│           │              │  CloudFront CDN  │                                            │
│           │              │ (Expert Videos)  │                                            │
│           │              └──────────────────┘                                            │
└───────────┼───────────────────────┼──────────────────────────────────────────────────────┘
            │                       │
            │ S3 Event              │ DynamoDB Streams
            ▼                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                            PROCESSING ORCHESTRATION                                      │
│                                                                                           │
│  ┌──────────────┐      ┌──────────────────┐      ┌─────────────────┐                   │
│  │  Amazon SQS  │─────►│ Step Functions   │◄─────│  EventBridge    │                   │
│  │ (Video Queue)│      │ (Workflow Engine)│      │  (Event Router) │                   │
│  └──────────────┘      └────────┬─────────┘      └─────────────────┘                   │
│                                 │                                                        │
│                                 │ Orchestrates                                           │
│                                 ▼                                                        │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
                    ▼             ▼             ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                          COMPUTER VISION PROCESSING (GPU)                                │
│                                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────────┐       │
│  │                         ECS Fargate (GPU Tasks)                              │       │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐          │       │
│  │  │ Action Recognition│  │ Object Detection │  │ Heat & Flame     │          │       │
│  │  │ (SlowFast/I3D)   │  │ (YOLOv8)         │  │ Analysis         │          │       │
│  │  └──────────────────┘  └──────────────────┘  └──────────────────┘          │       │
│  └──────────────────────────────────────────────────────────────────────────────┘       │
│                                                                                           │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐                  │
│  │  Amazon ECR      │    │  SageMaker       │    │  Rekognition     │                  │
│  │ (Container Images)│   │ (Model Endpoints)│    │ (Video Analysis) │                  │
│  │ - SlowFast       │    │ - ResNet         │    │ - Scene Detection│                  │
│  │ - YOLOv8         │    │ - Visual Embed   │    │ - Object Detect  │                  │
│  └──────────────────┘    └──────────────────┘    └──────────────────┘                  │
│                                                                                           │
└───────────────────────────────────────┬───────────────────────────────────────────────────┘
                                        │
                                        │ CV Results
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              TEMPORAL ALIGNMENT & SCORING                                │
│                                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────────┐       │
│  │                         AWS Lambda Functions                                 │       │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐          │       │
│  │  │ DTW Alignment    │  │ Score Calculator │  │ Result Aggregator│          │       │
│  │  │ (Sequence Match) │  │ (Weighted Scores)│  │ (Dashboard Data) │          │       │
│  │  └──────────────────┘  └──────────────────┘  └──────────────────┘          │       │
│  └──────────────────────────────────────────────────────────────────────────────┘       │
│                                                                                           │
└───────────────────────────────────────┬───────────────────────────────────────────────────┘
                                        │
                                        │ Scores + CV Data
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                AI COACHING LAYER                                         │
│                                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────────┐       │
│  │                         AWS Lambda (Coaching Logic)                          │       │
│  │                    - Prompt Construction                                     │       │
│  │                    - Context Retrieval                                       │       │
│  │                    - Feedback Formatting                                     │       │
│  └────────────────────────┬─────────────────────┬─────────────────────────────┘       │
│                           │                     │                                       │
│                           ▼                     ▼                                       │
│  ┌──────────────────────────────┐    ┌──────────────────────────────┐                  │
│  │    Amazon Bedrock            │    │  Amazon OpenSearch           │                  │
│  │  - Claude 3 Sonnet           │    │  - Vector Database           │                  │
│  │  - Multimodal LLM            │◄───│  - Cooking Knowledge Base    │                  │
│  │  - Personalized Feedback     │    │  - RAG Context Retrieval     │                  │
│  └──────────────────────────────┘    └──────────────────────────────┘                  │
│                                                                                           │
└───────────────────────────────────────┬───────────────────────────────────────────────────┘
                                        │
                                        │ AI Feedback
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                            NOTIFICATION & RESULTS DELIVERY                               │
│                                                                                           │
│  ┌──────────────────┐                                                                    │
│  │   Amazon SNS     │────► Push Notification to Mobile App                              │
│  │  (Notifications) │────► Email Notification                                           │
│  └──────────────────┘                                                                    │
│                                                                                           │
│  Results stored in DynamoDB + S3, retrieved via API Gateway                             │
│                                                                                           │
└─────────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                          CROSS-CUTTING CONCERNS                                          │
│                                                                                           │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐                      │
│  │  CloudWatch      │  │  AWS IAM         │  │  AWS KMS         │                      │
│  │  - Logs          │  │  - Roles         │  │  - Encryption    │                      │
│  │  - Metrics       │  │  - Policies      │  │  - Key Mgmt      │                      │
│  │  - Alarms        │  │  - Access Control│  │                  │                      │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘                      │
│                                                                                           │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐                      │
│  │  CloudTrail      │  │  Secrets Manager │  │  AWS Glue        │                      │
│  │  - Audit Logs    │  │  - API Keys      │  │  - ETL Jobs      │                      │
│  │  - Compliance    │  │  - Credentials   │  │  - Data Catalog  │                      │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘                      │
│                                                                                           │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

## Data Flow Summary

1. **Upload Flow**: Trainee → Amplify → Cognito → API Gateway → S3 → SQS
2. **Processing Flow**: SQS → Step Functions → [ECS + SageMaker + Rekognition] → Lambda
3. **AI Coaching Flow**: Lambda → Bedrock + OpenSearch → DynamoDB
4. **Results Flow**: DynamoDB/S3 → API Gateway → CloudFront → Mobile App
5. **Notification Flow**: SNS → Push/Email → Trainee

## Key Architecture Patterns

- **Event-Driven**: S3 events trigger processing pipeline
- **Serverless-First**: Lambda for lightweight tasks, Fargate for heavy compute
- **Parallel Processing**: Step Functions orchestrates concurrent CV tasks
- **Async Processing**: SQS decouples upload from processing
- **Microservices**: Each CV task runs independently in containers
- **RAG Pattern**: OpenSearch + Bedrock for contextual coaching
- **Multi-Tier Storage**: S3 (videos) + DynamoDB (metadata) + RDS (analytics)

## Cost Optimization Strategies

- Use Fargate Spot for 70% cost reduction on CV tasks
- S3 Intelligent-Tiering for automatic storage optimization
- Lambda for pay-per-use compute
- DynamoDB on-demand billing for variable workloads
- CloudFront caching to reduce S3 requests
- Batch processing where possible

## Security Layers

1. **Network**: VPC, Security Groups, WAF
2. **Authentication**: Cognito, IAM
3. **Encryption**: KMS (at rest), TLS (in transit)
4. **Audit**: CloudTrail, CloudWatch Logs
5. **Secrets**: Secrets Manager for credentials
