---
date: 2025-06-04
categories:
  - terraform
  - cloudfront
  - aws
  - route53
  - ssl
  - s3
  - devops
---
# Building a Reusable Terraform Static Site Module with CloudFront, S3, and Route 53

## Overview

A common need in modern cloud infrastructure is hosting static websites — whether it's marketing sites, documentation portals, or Single Page Applications (SPAs) built with React, Vue, or Svelte.

At first, the AWS building blocks for this are fairly simple:

- S3 for object storage
- CloudFront for CDN
- ACM for HTTPS
- Route 53 for DNS

But quickly, managing this setup by hand or duplicating configs across environments (prod, staging, QA) becomes painful:

- Too many copy/paste Terraform files
- Hard to apply consistent policies
- Complicated to manage uploads (especially when some sites are CI/CD and some are manual content sites)

![Terraform Static Site Module](../../assets/blog/static-site-module/banner.jpg)

<!-- more -->

## Why a Reusable Module?

I wanted a simple, composable way to manage:

- Multiple static sites across environments
- Both "content" sites (manual file uploads)
- And React apps (deployed by CI/CD)
- With a consistent CloudFront + ACM + Route 53 setup
- Using Terraform modules to avoid duplication

## Architecture

The module will:

1. Create an S3 bucket
1. (Optional) Enable versioning
1. Create a CloudFront distribution with Origin Access Identity (OAI)
1. Request an ACM certificate (DNS validated via Route 53)
1. Create an A/ALIAS record in Route 53 for the domain
1. (Optional) Upload local files using Terraform

## Module Main Code (`modules/static-site/main.tf`)

```hcl
resource "aws_s3_bucket" "bucket" {
  bucket        = var.bucket_name
  force_destroy = true
  tags = {
    Name        = var.bucket_name_tag
    Environment = var.environment
  }
}

resource "aws_s3_bucket_versioning" "bucket_versioning" {
  count  = var.enable_versioning ? 1 : 0
  bucket = aws_s3_bucket.bucket.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_cloudfront_origin_access_identity" "oai" {
  comment = "OAI for ${var.domain_name}"
}

resource "aws_s3_bucket_policy" "allow_cf" {
  bucket = aws_s3_bucket.bucket.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = aws_cloudfront_origin_access_identity.oai.iam_arn
        }
        Action   = "s3:GetObject"
        Resource = "${aws_s3_bucket.bucket.arn}/*"
      }
    ]
  })
}

resource "aws_acm_certificate" "cert" {
  domain_name       = var.domain_name
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.cert.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      type   = dvo.resource_record_type
      record = dvo.resource_record_value
    }
  }

  zone_id = var.route53_zone_id
  name    = each.value.name
  type    = each.value.type
  ttl     = 300
  records = [each.value.record]
}

resource "aws_acm_certificate_validation" "validate_cert" {
  certificate_arn         = aws_acm_certificate.cert.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}

resource "aws_cloudfront_distribution" "cdn" {
  depends_on = [aws_acm_certificate_validation.validate_cert]

  origin {
    domain_name = aws_s3_bucket.bucket.bucket_regional_domain_name
    origin_id   = "s3-origin"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.oai.cloudfront_access_identity_path
    }
  }

  enabled             = true
  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-origin"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate_validation.validate_cert.certificate_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  aliases = [var.domain_name]

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  custom_error_response {
    error_code         = 404
    response_code      = 200
    response_page_path = "/index.html"
  }
}

resource "aws_route53_record" "cf_alias" {
  zone_id = var.route53_zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.cdn.domain_name
    zone_id                = aws_cloudfront_distribution.cdn.hosted_zone_id
    evaluate_target_health = false
  }
}

# -------------------- Upload logic (optional) --------------------

locals {
  upload_path  = var.upload_path != null ? var.upload_path : "${path.root}/upload/${var.bucket_name}"
  upload_files = var.enable_uploads ? fileset(local.upload_path, "*") : []
}

resource "aws_s3_object" "uploads" {
  for_each     = { for f in local.upload_files : f => f }
  bucket       = aws_s3_bucket.bucket.id
  key          = each.key
  source       = "${var.upload_path}/${each.key}"
  source_hash  = filemd5("${var.upload_path}/${each.key}")
  content_type = lookup(var.mime_types, regex("[^.]+$", each.key), "application/octet-stream")
}
```

## Module Outputs (`modules/static-site/outputs.tf`)

```hcl
output "cloudfront_domain_name" {
  value = aws_cloudfront_distribution.cdn.domain_name
}

output "s3_bucket_name" {
  value = aws_s3_bucket.bucket.bucket
}

output "public_file_urls" {
  value = can(aws_s3_object.uploads) ? {
    for f, obj in aws_s3_object.uploads :
    f => "https://${var.domain_name}/${obj.key}"
  } : {}
}
```
## Module Variables

```hcl
variable "bucket_name" {}
variable "bucket_name_tag" {}
variable "environment" {}
variable "domain_name" {}
variable "route53_zone_id" {}
variable "enable_versioning" { default = false }
variable "enable_uploads" { default = false }
variable "upload_path" { default = null }
variable "mime_types" {
  description = "Map of file extensions to MIME types"
  type        = map(string)
  default = {
    json = "application/json"
    txt  = "text/plain"
    jpg  = "image/jpeg"
    jpeg = "image/jpeg"
    png  = "image/png"
    pdf  = "application/pdf"
  }
}
```

## Example Usage (`static-sites.tf` in root)

### Example 1 - Static File Site (uploads enabled)

```hcl
module "files_site" {
  source            = "./modules/static-site"
  bucket_name       = "files-site"
  bucket_name_tag   = "files-site"
  environment       = "prod"
  domain_name       = "files.example.com"
  route53_zone_id   = data.aws_route53_zone.example_com.zone_id
  enable_versioning = true
  enable_uploads    = true
  upload_path       = "${path.root}/upload/files_site"
}
```

### Example 2 - React App (no uploads, no versioning)

```hcl
module "dev_web" {
  source            = "./modules/static-site"
  bucket_name       = "dev-web"
  bucket_name_tag   = "dev-web"
  environment       = "dev"
  domain_name       = "dev.example.com"
  route53_zone_id   = data.aws_route53_zone.example_com.zone_id
  enable_versioning = false
  enable_uploads    = false
}
```

### `upload_path` default behavior

If `upload_path` is not specified, it defaults to:

```hcl
locals {
  upload_path = var.upload_path != null ? var.upload_path : "${path.root}/upload/${var.bucket_name}"
}
```

---

## Directory Structure

```
project-root/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
├── upload/
│   └── files_site/
└── modules/
    └── static-site/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        └── README.md
```

## Uploading or Replacing Files

For modules with `enable_uploads = true`, files will be uploaded from:

```
upload/<bucket_name>/
```

Or from:

```
upload/<custom path>
```

Terraform will handle uploading new files, updating changed files, or deleting removed files.

## URL for Uploaded Files

For static resource Site (uploads enabled), eg. with domain_name `files.example.com`

The files will be served at:

```
https://files.example.com/<filename>
```

## Conclusion

This reusable module pattern has been a very helpful addition to my Terraform workflows:

- I can spin up new static sites in minutes
- I can mix manual content sites and CI/CD React apps seamlessly
- Everything stays consistent across environments
- It is safe, extensible, and easy to maintain

If you need to manage multiple static sites with Terraform, I highly recommend building or adopting a module like this:

It keeps your configuration clean, avoids "copy/paste debt," and makes it easy to scale as your team or product grows.

## Future Improvements

Some future ideas to improve this module:

- Add CloudFront logging
- Add S3 lifecycle policies
