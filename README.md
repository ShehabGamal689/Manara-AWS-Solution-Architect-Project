# AWS Solution Architecture

> A serverless, event-driven image processing pipeline using Amazon S3 triggers, AWS Lambda (Python + Pillow), and separate source/destination buckets.

---

## 📖 Table of Contents
1. [Overview](#overview)  
2. [Architecture Diagram](#architecture-diagram)  
3. [Component Breakdown](#component-breakdown)  
   - [Amazon S3 (Source & Destination)](#amazon-s3-source--destination)  
   - [AWS Lambda (Processor)](#aws-lambda-processor)  
   - [Lambda Layer (Pillow)](#lambda-layer-pillow)  
   - [S3 Event Notifications](#s3-event-notifications)  
   - [(Optional) API Gateway](#optional-api-gateway)  
   - [(Optional) DynamoDB](#optional-dynamodb)  
   - [(Optional) Step Functions](#optional-step-functions)  
4. [Data Flow](#data-flow)  
5. [High Availability & Fault Tolerance](#high-availability--fault-tolerance)  
6. [Security Considerations](#security-considerations)  

---

<a id="overview"></a>
## 📌 Overview

This serverless architecture processes user-uploaded images by leveraging:

- **Amazon S3** for raw uploads (source) and processed outputs (destination)  
- **AWS Lambda (Python 3.9 + Pillow)** for resizing/watermarking  
- **S3 object-created events** to trigger processing automatically  
- **Environment variables** to decouple code from infrastructure (bucket names, prefixes)  
- **Optional** integrations for uploads (API Gateway), metadata (DynamoDB), and workflows (Step Functions)

**Your deployment (concrete values):**
- **Source bucket:** `manara-image-resize-uploader` (prefix: `images/`)  
- **Destination bucket:** `manara-image-resize-reciever`  
- **Lambda runtime:** Python 3.9  
- **Env vars:** `DEST_BUCKET=manara-image-resize-reciever`, `SRC_BUCKET=manara-image-resize-uploader` *(optional guard)*

---

<a id="architecture-diagram"></a>
## 🖼️ Architecture Diagram
![Serverless Image Processing](architecture-diagram.png)  
*Figure 1: High-level diagram of the solution*

---

<a id="component-breakdown"></a>
## 🔍 Component Breakdown

<a id="amazon-s3-source--destination"></a>
### Amazon S3 (Source & Destination)
- **Source bucket (`manara-image-resize-uploader`)**  
  - Receives user uploads under `images/` (e.g., `images/cat-s3.png`).  
  - Emits **ObjectCreated:Put** events to Lambda.  
- **Destination bucket (`manara-image-resize-reciever`)**  
  - Stores resized outputs in logical folders (e.g., `desktop/`, `tablet/`, `phone/`).  
  - Block Public Access enabled; access via IAM or presigned URLs.

<a id="aws-lambda-processor"></a>
### AWS Lambda (Processor)
- **Purpose:** Read original from source bucket, create resized variants, write to destination bucket.  
- **Key settings:**  
  - Runtime **Python 3.9**, Memory ≥ **512 MB**, Timeout ≥ **30 s**.  
  - **Environment variables:**  
    - `DEST_BUCKET` (required)  
    - `SRC_BUCKET` (optional safety check)  
- **Execution role permissions (minimum):**
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      { "Effect": "Allow", "Action": ["s3:GetObject"], "Resource": "arn:aws:s3:::manara-image-resize-uploader/*" },
      { "Effect": "Allow", "Action": ["s3:ListBucket"], "Resource": "arn:aws:s3:::manara-image-resize-uploader",
        "Condition": { "StringLike": { "s3:prefix": ["images/*"] } } },
      { "Effect": "Allow", "Action": ["s3:PutObject"], "Resource": "arn:aws:s3:::manara-image-resize-reciever/*" },
      { "Effect": "Allow", "Action": ["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"], "Resource": "*" }
    ]
  }
