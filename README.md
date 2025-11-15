```markdown
# SwiftHaul Logistics — Serverless Logistics Tracking Platform

⚠️ ACADEMIC PROJECT / PROOF OF CONCEPT  
This was a student capstone project demonstrating cloud architecture concepts.  
Implementation was done in a single AWS region (ap-southeast-1) as a working prototype.  
The architecture document describes a proposed 5-region production design.

IMPORTANT SECURITY NOTE: terraform.tfstate and terraform.tfstate.backup are present in this repository. Terraform state can contain sensitive information. See "Cleanup & safety" below for immediate remediation steps.

---

## Table of contents
- Overview
- High-level architecture (diagram & description)
- Prerequisites
- Repository layout
- Setup (local + CI)
- Deployment (design vs implementation)
- Lambda function descriptions
- Configuration variables (examples)
- Usage examples
- Monitoring, testing & troubleshooting
- Cost & scaling considerations
- Cleanup & safety / next steps
- Future Enhancements (Not Implemented)
- License

---

## Overview
SwiftHaul is a multi-region, serverless, event-driven logistics platform design that provides real-time parcel tracking, GPS telemetry ingestion, and traffic-aware routing.

- Designed for 99.99% uptime (not tested at production scale)
- Designed tracking-update latency: < 5s (prototype demonstrates the end-to-end flow but not production-scale latency guarantees)
- Designed for: ~170,000 parcels/day (prototype tested with mock data)
- Designed for 2,500+ vehicles (prototype tested with simulated devices)

Key principles:
- Serverless compute (Lambda) for elasticity
- Event-driven architecture with SQS + EventBridge
- DynamoDB Global Tables for cross-region replication in the proposed design
- IoT Core for MQTT telemetry ingestion (prototype uses simulated device data)
- Amazon Location Service for traffic-aware routing and optimization
- React SPA frontend hosted via S3 + CloudFront (prototype artifacts included)

This repository contains the Terraform code and Lambda packaging used to build the prototype implementation (single-region ap-southeast-1). The architecture sections describe a proposed 5-region production design.

---

## High-level architecture (diagram & description)
Conceptual ASCII diagram:

                     +--------------------+        +----------------------+
                     |   Delivery Device  |  MQTT  |      AWS IoT Core     |
    GPS telemetry -->| (vehicle tracker)  |------->| (Authentication &     |
                     +--------------------+        |  message ingestion)   |
                                                     +---+---+---+---+---+
                                                         |   |
                                                         |   V
                                                   +-----+   +------+
                                                   | SQS |   | Lambda |
                                                   |(buffer)|  processFilterGPSData
                                                   +-----+   +---------+
                                                         |         |
                                                         V         V
               Warehouse scan -> API Gateway -> extractParcelDetails Lambda -> DynamoDB (Global Tables - design)
                                                         |
                                                         V
                                              triggerRouteGeneration Lambda -> Location Service API
                                                         |
                                                         V
                                           retrieveRealTimeData Lambda (WebSocket) -> Web Clients
                                                         |
                                                  +------+-------+
                                                  | CloudWatch    |
                                                  | Alarms/Logs   |
                                                  +---------------+
                                                  + Route53 + WAF
                                                  + CloudFront + S3
                                                  + Cognito + ACM

Description:
- Devices publish to IoT Core over MQTT. For the prototype, telemetry is typically simulated and fed into the stack.
- IoT rules can forward messages into SQS for durable buffering.
- processFilterGPSData Lambda consumes SQS messages, validates and enriches telemetry, and writes to DynamoDB (prototype uses single-region tables; the repository includes configuration for Global Tables as a proposed design).
- API Gateway exposes REST and WebSocket endpoints. extractParcelDetails handles parcel intake; retrieveRealTimeData handles live subscription and push of updates.
- triggerRouteGeneration calls Amazon Location Service to compute optimized, traffic-aware routes.

---

## Prerequisites
- Terraform (recommended >= 1.4.x; respect required_version in TF files if set)
- AWS CLI configured with a user/role that can manage the resources (for the prototype, deployment used ap-southeast-1)
- Node.js & npm (to build frontend and/or package Lambdas if needed)
- zip (or similar) for packaging Lambda artifacts
- Optional: jq, aws-vault or SSO tooling for secure credential handling

Recommended permissions for initial bootstrapping: elevated deployer role. After bootstrapping, adopt least-privilege roles for CI.

---

## Repository layout
Top-level Terraform files are split by AWS service. Root files include:
- api_gateway.tf
- cloudfront.tf
- cloudwatch.tf
- cognito.tf
- dynamodb.tf
- eventbridge.tf
- iam.tf
- iot.tf
- lambda.tf
- main.tf
- data.tf
- location.tf
- outputs.tf
- route53.tf
- s3.tf
- sqs.tf
- budget.tf
- frontend/ (React SPA source or artifacts)
- lambda/ (packaged function code and/or build artifacts)
- terraform.tfstate & terraform.tfstate.backup (committed — see Cleanup & safety)

These files reflect the prototype implementation and the design artifacts for the multi-region proposal.

---

## Setup (local & CI)
1. Secure credentials
   - Use environment-based credentials or a secure helper (aws-vault, SSO).
2. Immediately address committed state (see Cleanup & safety).
3. Create a remote backend for state before collaborative work (recommended: S3 + DynamoDB lock).
4. Example backend snippet (backend.tf) — add this before running terraform init:
   terraform {
     backend "s3" {
       bucket         = "swifthaul-terraform-state-<account-id>"
       key            = "swifthaul/prod/terraform.tfstate"
       region         = "us-west-2"
       dynamodb_table = "swifthaul-terraform-locks"
       encrypt        = true
     }
   }

5. CI (recommended for future work):
   - Validate formatting and run terraform validate on PRs.
   - For authenticated plan & apply in CI, use short-lived role assumption.

---

## Deployment (design vs implementation)
Important clarifications for this repository:
- Multi-region deployment is DESIGNED but not implemented in the prototype.
- Actual implementation is single-region (Singapore / ap-southeast-1).
- Blue/Green deployment strategy is PROPOSED, not implemented in automation for the prototype.

Suggested multi-region approach (DESIGN):
- Use modules with provider aliases and for_each over a regions list to create replicated resources.
- DynamoDB Global Tables to replicate telemetry and parcel data across regions.
- Centralized CI/CD with environment-specific backends and state separation.

Prototype deployment steps (single-region ap-southeast-1):
1. Configure backend (or use local state for quick tests).
2. terraform init
3. terraform fmt && terraform validate
4. terraform plan -var-file=prod.tfvars -out=prod.plan
5. terraform apply "prod.plan"
6. terraform destroy when finished

Blue/Green (PROPOSED):
- Publish Lambda versions, use aliases (candidate/live), shift traffic via API Gateway canary or Route53 weighted records. This is a proposed deployment pattern and not implemented in automated pipelines in this repository.

---

## Lambda functions (detailed)
These Lambda functions were implemented and tested with mock/simulated data in a single region.

1. extractParcelDetails
   - Trigger: API Gateway (REST)
   - Purpose: Receive parcel creation events (scans), validate payloads, create items in DynamoDB.
   - Env vars: PARCELS_TABLE, REGION, ENABLE_EVENTS
   - Timeout: 10s, Memory: 512MB

2. updateParcelStatus
   - Trigger: Scheduled (EventBridge) or batch process via API
   - Purpose: Batch updates and reconciliation
   - Timeout: 60s, Memory: 1024MB

3. processFilterGPSData
   - Trigger: SQS (from IoT Core rules)
   - Purpose: Validate and filter GPS telemetry, enrich and write to telemetry table
   - Concurrency/Scaling: Prototype uses basic settings; use reserved concurrency for production

4. retrieveRealTimeData
   - Trigger: API Gateway WebSocket
   - Purpose: Manage subscriptions and push live telemetry to clients

5. triggerRouteGeneration
   - Trigger: Event or API
   - Purpose: Call Amazon Location Service to compute optimized routes and store results

Operational notes:
- Lambdas in the prototype were exercised with simulated devices and mock data flows.
- For production, use Layers, provisioned concurrency for critical handlers, and granular IAM policies.

---

## Configuration variables (examples)
Example variables.tf fragment:
```hcl
variable "environment" {
  type    = string
  default = "prod"
}

variable "regions" {
  type    = list(string)
  default = ["ap-southeast-1"]
}

variable "account_id" {
  type = string
}

variable "lambda_memory_mb" {
  type    = number
  default = 512
}

variable "lambda_timeout" {
  type    = number
  default = 30
}
```

Example prod.tfvars:
```hcl
environment        = "prod"
regions            = ["ap-southeast-1"]
account_id         = "123456789012"
lambda_memory_mb   = 512
lambda_timeout     = 30
```

Environment variables for Lambdas (examples):
- PARCELS_TABLE=swifthaul-parcels
- TELEMETRY_TABLE=swifthaul-telemetry
- ROUTES_TABLE=swifthaul-routes
- REGION=ap-southeast-1

Secrets: Use AWS Secrets Manager or SSM Parameter Store; do not hardcode.

---

## Usage examples
Note: These are examples based on the designed architecture. Actual implementation includes basic versions of these endpoints for demonstration purposes.

1. Create a plan and apply (example):
   terraform init -backend-config="bucket=swifthaul-terraform-state-123456789012" \
                  -backend-config="key=swifthaul/prod/terraform.tfstate" \
                  -backend-config="region=us-west-2"
   terraform plan -var-file=prod.tfvars -out=prod.plan
   terraform apply "prod.plan"

2. Publish new Lambda version and shift traffic (manual example - proposed):
   aws lambda publish-version --function-name extractParcelDetails --description "v2025-11-15"
   aws lambda update-alias --function-name extractParcelDetails --name candidate --function-version <new-version>

3. Sample REST call to create a parcel (prototype endpoint may differ):
   curl -X POST "https://<api-id>.execute-api.ap-southeast-1.amazonaws.com/prod/parcels" \
     -H "Authorization: Bearer <cognito_token>" \
     -H "Content-Type: application/json" \
     -d '{"parcelId":"P123456","weight":2.3,"origin":"SGSIN","destination":"USLAX","scannedBy":"WH01"}'

4. Publish GPS telemetry to IoT Core (simulated device payload):
   aws iot-data publish --topic "swifthaul/gps" --cli-binary-format raw-in-base64-out \
     --payload '{"deviceId":"VEH-001","lat":1.3521,"lon":103.8198,"ts":1699999999}' --region ap-southeast-1

5. Connect to WebSocket for live updates (JS example):
   const ws = new WebSocket('wss://<websocket-api-id>.execute-api.ap-southeast-1.amazonaws.com/prod');
   ws.onopen = () => ws.send(JSON.stringify({ action: 'subscribe', parcelId: 'P123456' }));
   ws.onmessage = (msg) => console.log('live update', msg.data);

---

## Monitoring, testing & troubleshooting
- CloudWatch (logs & metrics): Lambda durations, errors, throttles; SQS approximate age; DynamoDB consumed capacity
- X-Ray: Useful for tracing API Gateway → Lambda → DynamoDB → Location Service paths
- Canary tests: synthetic API checks and Route53 health checks (design)
- Load testing: Prototype uses simulated loads; comprehensive load testing is a future enhancement

---

## Cost & scaling considerations
- DynamoDB: consider On-Demand or provisioned throughput tuned to telemetry write patterns
- Location Service: per-route billing—cache results where possible
- Data retention: use TTLs to limit telemetry retention and offload old data to S3/Glacier

---

## Cleanup & safety (immediate actions)
1. Remove committed terraform.tfstate from repo:
   git rm --cached terraform.tfstate terraform.tfstate.backup
   Add to .gitignore:
     terraform.tfstate
     terraform.tfstate.backup
     .terraform/
2. Scrub history if sensitive information was exposed (BFG or git filter-repo).
3. Rotate any credentials that may be present in state or referenced by roles.
4. Migrate to a remote backend and run `terraform init -migrate-state`.

---

## Future Enhancements (Not Implemented)
- Multi-region deployment
- Blue/Green deployment automation
- Production CI/CD pipeline
- Full IoT device integration
- Comprehensive load testing

These are planned or proposed capabilities to bring the design to production readiness.

---

## License
Add a LICENSE file (e.g., Apache-2.0 or MIT) to clarify permitted use.

---

This README documents the prototype implementation (single-region) and the proposed multi-region production design. Treat the repository as an educational capstone/proof-of-concept rather than a production-ready codebase.
```
