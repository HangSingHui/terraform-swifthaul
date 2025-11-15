# SwiftHaul Logistics — Serverless Logistics Tracking Platform

⚠️ Academic Project / Proof of Concept  
Student capstone demonstrating cloud architecture concepts.  
Prototype deployed in single AWS region (ap-southeast-1).  
Proposed multi-region production design described below.

> Important: terraform.tfstate and terraform.tfstate.backup are included. These files may contain sensitive info.

---

## Table of Contents
- Overview
- High-level Architecture
- Prerequisites
- Repository Layout
- Deployment
- Lambda Functions
- Future Enhancements

---

## Overview

SwiftHaul is a serverless, event-driven logistics platform providing:

- Real-time parcel tracking
- GPS telemetry ingestion
- Traffic-aware routing

Prototype goals:

- Uptime: ~99.99% (not tested at production scale)
- Tracking latency: <5s (prototype demonstrates flow)
- Load: ~170,000 parcels/day (mock-tested)
- Vehicles: ~2,500+ (simulated devices)

Key principles:

- Serverless compute (AWS Lambda)
- Event-driven architecture (SQS + EventBridge)
- DynamoDB Global Tables (design for multi-region replication)
- AWS IoT Core for telemetry ingestion (prototype uses simulation)
- Amazon Location Service for routing
- React SPA frontend via S3 + CloudFront

---

## High-level Architecture

Flow overview:

1. Devices publish telemetry to IoT Core over MQTT.
2. IoT rules forward messages to SQS for buffering.
3. processFilterGPSData Lambda consumes SQS messages, validates/enriches telemetry, writes to DynamoDB.
4. API Gateway exposes endpoints:
   - extractParcelDetails: parcel intake
   - retrieveRealTimeData: live telemetry updates
5. triggerRouteGeneration Lambda calls Amazon Location Service for routing.

Components:

- IoT Core — device telemetry ingestion
- SQS — buffering and durability
- Lambdas — processing, routing, WebSocket updates
- DynamoDB — storage (prototype: single region)
- API Gateway — REST & WebSocket
- CloudFront + S3 — frontend hosting
- CloudWatch / X-Ray — monitoring and tracing
- Route53 + WAF — DNS and security

---

## Prerequisites

- Terraform >= 1.4.x
- AWS CLI configured with deploy permissions
- Node.js & npm (frontend build / Lambda packaging)
- zip (or similar) for Lambda packaging
- Optional: jq, aws-vault or SSO

Recommended: elevated deployer role for initial bootstrap; use least-privilege roles after.

---

## Repository Layout

Top-level files:

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
- frontend/ (React SPA source or build artifacts)
- lambda/ (Lambda packaged code)
- terraform.tfstate & terraform.tfstate.backup

> Note: State files are committed.

---

## Deployment

- Prototype: Single-region (ap-southeast-1)  
- Design: Multi-region with provider aliases, DynamoDB Global Tables  
- Blue/Green Deployment (Proposed): Lambda versions + aliases, API Gateway canary, Route53 weighted records  

Prototype deployment:

terraform init  
terraform fmt && terraform validate  
terraform plan -var-file=prod.tfvars -out=prod.plan  
terraform apply "prod.plan"  
terraform destroy

---

## Lambda Functions

| Lambda Function        | Trigger                 | Purpose                                           |
|------------------------|------------------------|-------------------------------------------------|
| extractParcelDetails   | API Gateway (REST)      | Parcel intake & DynamoDB write                 |
| updateParcelStatus     | EventBridge / batch     | Batch updates & reconciliation                |
| processFilterGPSData   | SQS                     | Validate, enrich, write telemetry             |
| retrieveRealTimeData   | API Gateway WebSocket   | Push live updates                              |
| triggerRouteGeneration | Event/API               | Call Location Service for routing             |

---

## Future Enhancements

- Multi-region deployment  
- Blue/Green deployment automation  
- Production CI/CD pipeline  
- Full IoT device integration  
- Comprehensive load testing  

---


*This README documents the prototype (single-region) and proposed multi-region design. Educational / proof-of-concept only.*

