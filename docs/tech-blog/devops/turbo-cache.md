# Setting Up Turborepo Remote Cache with S3 and GitHub Actions

Setting up a production-grade remote cache for [Turborepo](https://turbo.build/) using self hosted remote cache with AWS S3 and Lambda helps improve monorepo performance, especially in CI/CD pipelines like GitHub Actions. Below is a modular and generic Terraform setup using variables for easy customization.

![Turborepo Cache](../../assets/tech-blog/devops/turbo-cache/banner.jpg)

This guide walks you through setting up a secure and production-ready remote cache using:

- AWS S3 (with default encryption)
- GitHub Actions with IAM AssumeRole via OIDC
- Infrastructure as Code using Terraform


## S3 Bucket Requirements

- **Default encryption (AES-256)**: Enabled
- **Versioning**: Disabled (not needed for cache)
- **Public Access**: Blocked

## Terraform Setup

### Directory Structure

```
infra/
├── main.tf
├── variables.tf
├── outputs.tf
└── lambda.zip  # Your compiled Turborepo cache handler
```

## Prepare lambda.zip

Use the following commands to generate a lambda.zip file. For more information, checkout [ducktors documentation](https://ducktors.github.io/turborepo-remote-cache/running-in-lambda.html#create-the-lambda-function).

```bash
npm install turborepo-remote-cache
echo "export { handler } from 'turborepo-remote-cache/aws-lambda';" > index.js
esbuild index.js --bundle --platform=node --outfile=dist/index.js
cd dist && zip lambda.zip index.js
mv lambda.zip ..
```

### `main.tf`

```hcl
variable "bucket_name" {
  description = "Name of the S3 bucket for Turborepo cache"
  type        = string
}

variable "environment" {
  description = "Environment tag for resources"
  type        = string
  default     = "Development"
}

variable "turbo_token" {
  description = "Turbo token used by the Lambda function"
  type        = string
}

variable "github_oidc_provider_arn" {
  description = "GitHub OIDC provider ARN"
  type        = string
}

variable "github_org_or_repo_pattern" {
  description = "GitHub OIDC subject pattern"
  type        = string
}

resource "aws_s3_bucket" "turbo_cache" {
  bucket = var.bucket_name

  tags = {
    Name        = "Turborepo Cache Bucket"
    Environment = var.environment
  }
}

resource "aws_s3_bucket_ownership_controls" "turbo_cache" {
  bucket = aws_s3_bucket.turbo_cache.id

  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "turbo_cache" {
  bucket = aws_s3_bucket.turbo_cache.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "turbo_cache" {
  bucket = aws_s3_bucket.turbo_cache.id

  rule {
    id     = "cleanup-old-cache"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    expiration {
      days = 30
    }
  }
}

resource "aws_s3_bucket_public_access_block" "turbo_cache" {
  bucket = aws_s3_bucket.turbo_cache.id

  block_public_acls       = true
  ignore_public_acls      = true
  restrict_public_buckets = true
  block_public_policy     = true
}

resource "aws_s3_bucket_policy" "turbo_cache" {
  bucket = aws_s3_bucket.turbo_cache.id
  policy = data.aws_iam_policy_document.s3_secure_transport_deny.json
}

resource "aws_iam_role" "github_actions_role" {
  name               = "github-actions-turborepo-cache-role"
  assume_role_policy = data.aws_iam_policy_document.github_actions_assume_role.json
}

resource "aws_iam_role" "turbo_cache_lambda_role" {
  name               = "turborepo-cache-lambda-role"
  assume_role_policy = data.aws_iam_policy_document.lambda_assume_role.json
}

resource "aws_iam_policy" "turbo_cache_lambda_policy" {
  name   = "turborepo-cache-lambda-policy"
  policy = data.aws_iam_policy_document.lambda_policy.json
}

resource "aws_iam_role_policy_attachment" "turbo_cache_lambda_attach" {
  role       = aws_iam_role.turbo_cache_lambda_role.name
  policy_arn = aws_iam_policy.turbo_cache_lambda_policy.arn
}

resource "aws_lambda_function" "turbo_cache" {
  function_name    = "turborepo-remote-cache"
  role             = aws_iam_role.turbo_cache_lambda_role.arn
  handler          = "index.handler"
  runtime          = "nodejs22.x"
  filename         = "${path.module}/lambda.zip"
  source_code_hash = filebase64sha256("${path.module}/lambda.zip")

  environment {
    variables = {
      STORAGE_PATH     = aws_s3_bucket.turbo_cache.bucket
      STORAGE_PROVIDER = "s3"
      TURBO_TOKEN      = var.turbo_token
    }
  }
}

resource "aws_lambda_function_url" "turbo_cache_lambda_url" {
  function_name      = aws_lambda_function.turbo_cache.function_name
  authorization_type = "NONE"

  cors {
    allow_origins = ["*"]
    allow_methods = ["*"]
    allow_headers = ["*"]
  }
}

data "aws_iam_policy_document" "github_actions_assume_role" {
  statement {
    effect = "Allow"

    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [var.github_oidc_provider_arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = [var.github_org_or_repo_pattern]
    }
  }
}

data "aws_iam_policy_document" "lambda_assume_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]
  }
}

data "aws_iam_policy_document" "lambda_policy" {
  statement {
    effect = "Allow"

    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:HeadObject",
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents"
    ]

    resources = [
      aws_s3_bucket.turbo_cache.arn,
      "${aws_s3_bucket.turbo_cache.arn}/*",
      "arn:aws:logs:*:*:*"
    ]
  }
}

data "aws_iam_policy_document" "s3_secure_transport_deny" {
  statement {
    sid     = "DenyInsecureTransport"
    effect  = "Deny"
    actions = ["s3:*"]

    principals {
      type        = "*"
      identifiers = ["*"]
    }

    resources = [
      aws_s3_bucket.turbo_cache.arn,
      "${aws_s3_bucket.turbo_cache.arn}/*"
    ]

    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }
}

```

### `variables.tf`

```hcl
variable "bucket_name" {
  description = "Name of the S3 bucket for Turborepo cache"
  type        = string
}

variable "environment" {
  description = "Deployment environment tag (e.g., Development, Staging, Production)"
  type        = string
}

variable "github_org_or_repo_pattern" {
  description = "GitHub OIDC repo pattern for role assumption"
  type        = string
}

variable "github_oidc_provider_arn" {
  description = "ARN of the GitHub OIDC provider"
  type        = string
}

variable "turbo_token" {
  description = "Turborepo access token"
  type        = string
  sensitive   = true
}

```

### `outputs.tf`

```hcl
output "s3_bucket_name" {
  value = aws_s3_bucket.turbo_cache.id
}

output "github_role_arn" {
  value = aws_iam_role.github_actions.arn
}

output "lambda_url" {
  value = aws_lambda_function_url.turbo_cache_lambda_url.function_url
}
```

## terraform.tfvars Sample

```hcl
bucket_name                = "my-turbo-cache-bucket"
environment                = "Development"
github_oidc_provider_arn   = "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
github_org_or_repo_pattern = "repo:my-org/*"
turbo_token                = "your-turborepo-token"
```

## GitHub Actions Workflow

`.github/workflows/build.yml`

```yaml
name: Build with Turborepo Cache (S3)

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      TURBO_TEAM: your-team
      TURBO_TOKEN: ${{ secrets.TF_VAR_TURBO_TOKEN }}
      TURBO_API: replace-with-lambda_url-out


    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-turbo-role
          role-session-name: GitHubActionsTurboCacheSession
          aws-region: us-east-1

      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 8

      - name: Install dependencies
        run: pnpm install

      - name: Build with Turbo cache
        run: pnpm turbo run build --team="your-team" --token=${{ secrets.TF_VAR_TURBO_TOKEN }}
```

---

## turbo.json Sample

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "dev": {
      "cache": false
    },
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    }
  }
}
```

## Result:

### Without cache
```
 Tasks:    1 successful, 1 total
Cached:    0 cached, 1 total
  Time:    4.648s 
```

### With cache
```
 Tasks:    1 successful, 1 total
Cached:    1 cached, 1 total
  Time:    721ms >>> FULL TURBO
```

## Summary

This Terraform-based setup provisions a secure and production-ready Turborepo remote cache using:

- S3 for storage (with AES-256 encryption and lifecycle rules)

- Lambda to serve the cache API

- IAM/OIDC Integration with GitHub Actions for secure, short-lived access

- GitHub Actions workflow pre-wired to leverage the remote cache

### Key Benefits:
Significant speed-up in CI pipelines using cached builds
Modular and environment-agnostic Terraform for reusable infra
Security best practices enforced (e.g., S3 bucket policies, IAM roles)
Easy-to-integrate GitHub Actions support with OIDC

### Performance Gain

```
Build Time Comparison
|
|    ██████████████████  4.648s (No Cache)
|    ██                  0.721s (With Cache)
|
```
~84.5% time saved per build with the remote cache!