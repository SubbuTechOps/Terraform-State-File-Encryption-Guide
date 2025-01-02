# Terraform State File Encryption Guide

## Overview
This guide demonstrates how to enable encryption at rest for Terraform state files using AWS S3 as the backend.

## Configuration Steps

### 1. Backend Configuration
Create a backend configuration file `backend.tf`:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "prod/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-west-2:ACCOUNT-ID:key/KEY-ID"
    dynamodb_table = "terraform-state-lock"
  }
}
```

### 2. KMS Key Setup
Create a KMS key for state file encryption:

```hcl
resource "aws_kms_key" "terraform_state_key" {
  description             = "KMS key for Terraform state encryption"
  deletion_window_in_days = 10
  enable_key_rotation     = true
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      }
    ]
  })

  tags = {
    Environment = "Production"
    Purpose     = "Terraform State Encryption"
  }
}

resource "aws_kms_alias" "terraform_state_key_alias" {
  name          = "alias/terraform-state-key"
  target_key_id = aws_kms_key.terraform_state_key.key_id
}
```

### 3. S3 Bucket Configuration
Configure the S3 bucket with encryption:

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-bucket"
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.terraform_state_key.arn
      sse_algorithm     = "aws:kms"
    }
  }
}
```

### 4. DynamoDB Table for State Locking
Create a DynamoDB table for state locking:

```hcl
resource "aws_dynamodb_table" "terraform_state_lock" {
  name           = "terraform-state-lock"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## Best Practices

1. **Access Control**
   - Implement strict IAM policies for KMS key and S3 bucket access
   - Use separate KMS keys for different environments
   - Enable bucket versioning for state file recovery

2. **Monitoring**
   - Enable CloudTrail logging for S3 and KMS operations
   - Set up alerts for unauthorized access attempts
   - Monitor KMS key usage and rotation

3. **Security Measures**
   - Enable MFA delete for the S3 bucket
   - Use VPC endpoints for S3 access
   - Regularly audit KMS key policies

## Usage Example

Initialize Terraform with the encrypted backend:

```bash
terraform init \
  -backend-config="bucket=my-terraform-state-bucket" \
  -backend-config="key=prod/terraform.tfstate" \
  -backend-config="region=us-west-2" \
  -backend-config="encrypt=true" \
  -backend-config="kms_key_id=arn:aws:kms:us-west-2:ACCOUNT-ID:key/KEY-ID"
```

## Verification

Verify encryption status:

```bash
aws s3api head-object \
  --bucket my-terraform-state-bucket \
  --key prod/terraform.tfstate
```

The output should show the SSE configuration:
```json
{
    "ServerSideEncryption": "aws:kms",
    "SSEKMSKeyId": "arn:aws:kms:us-west-2:ACCOUNT-ID:key/KEY-ID"
}
```
