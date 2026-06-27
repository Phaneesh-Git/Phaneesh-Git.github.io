---
layout: post
title: "Deploying a Static Website on AWS with Terraform and CloudFront"
date: 2026-06-27
categories: [Terraform, AWS]
tags: [Terraform, AWS, S3, CloudFront, Static Website]
image:
  path: /assets/headers/2026-06-27.jpg
  alt: "Deploying a Static Website on AWS with Terraform and CloudFront"
  width: 1200
  height: 630
  aspect ratio: 1.91:1
published: true
---
> **Complete Tech Blog & Setup Guide**  
> *Author:* Phaneesh | *Date:* June 27, 2026 | *Repository:* [Deploying a Static Website on AWS with Terraform and CloudFront](https://github.com/Phaneesh-Git/Deploying-a-Static-Website-on-AWS-with-Terraform-and-CloudFront)

* * *

## Introduction
In the world of web hosting, static websites are praised for their simplicity, performance, and security. When combined with the power of Infrastructure as Code (IaC) tools like Terraform, deploying and managing them becomes a breeze. This guide will walk you through a Terraform setup that automates the deployment of a static website on AWS, leveraging S3 for storage and CloudFront for content delivery.

## Purpose of the Repository
The Terraform code in this repository is designed to provision all the necessary AWS infrastructure for hosting a static website. By running a few commands, you can create an S3 bucket, configure it for web hosting, set up a CloudFront distribution for global content delivery, and even handle URL rewrites for clean, user-friendly URLs. This automation eliminates manual configuration, reduces the chance of errors, and makes the entire process repeatable and scalable.

## Architecture and Workflow
The architecture is straightforward yet robust. Here’s a breakdown of the components and how they work together:

1.  **S3 Bucket:** This is the core storage for the website's files (HTML, CSS, JavaScript, images, etc.). The Terraform code creates a private S3 bucket and configures it with versioning to keep a history of your files.

2.  **CloudFront Distribution:** A CloudFront distribution is set up to act as a Content Delivery Network (CDN). It caches the website's content at edge locations around the world, ensuring low-latency access for your users. The distribution is configured to serve content from the S3 bucket.

3.  **Origin Access Control (OAC):** To secure the S3 bucket and ensure that content is only served through CloudFront, we use Origin Access Control. This prevents direct access to the S3 bucket, enhancing security.

4.  **CloudFront Function:** A CloudFront function, `jekyll_url_rewrite`, is included to handle URL rewrites. This is particularly useful for static site generators like Jekyll that produce "pretty" URLs (e.g., `/about/` instead of `/about.html`). The function automatically appends `index.html` to such URLs, ensuring that the correct page is served.

## Code Snippets
Let's take a look at some key parts of the Terraform code.

### S3 Bucket and CloudFront Distribution
Here’s how the S3 bucket and CloudFront distribution are defined in `Main.tf`:
```terraform
resource "aws_s3_bucket" "static_website_bucket" {
  bucket = var.bucket_name
  tags = {
    Name = "delete-this"
  }
}

resource "aws_cloudfront_distribution" "s3_distribution" {
  origin {
    domain_name              = aws_s3_bucket.static_website_bucket.bucket_regional_domain_name
    origin_access_control_id = aws_cloudfront_origin_access_control.cloudfront_access_control.id
    origin_id                = local.origin_id
  }

  enabled             = true
  is_ipv6_enabled     = true
  comment             = "Some comment"
  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = local.origin_id

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
        
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
```

### CloudFront Function for URL Rewrites
The `jekyll_url_rewrite` function is a neat piece of JavaScript that runs at the edge:
```terraform
resource "aws_cloudfront_function" "jekyll_url_rewrite" {
  name    = "jekyll-url-rewrite"
  runtime = "cloudfront-js-2.0"
  comment = "Appends index.html to subdirectories for Jekyll Pretty URLs"
  publish = true
  code    = <<EOF
function handler(event) {
    var request = event.request;
    var uri = request.uri;
    
    // If the URI ends with a a slash, append index.html
    if (uri.endsWith('/')) {
        request.uri += 'index.html';
    } 
    // If the URI doesn't have an extension (like /about), append /index.html
    else if (!uri.includes('.')) {
        request.uri += '/index.html';
    }
    
    return request;
}
EOF
}
```

## Key Takeaways
- **Automation:** Terraform automates the entire infrastructure setup, making it easy to create, update, and destroy your static website hosting environment.
- **Scalability:** The use of S3 and CloudFront ensures that your website can handle high traffic loads and serve content globally with low latency.
- **Security:** Origin Access Control secures your S3 bucket, preventing unauthorized access to your files.
- **Cost-Effective:** Hosting a static website on S3 and CloudFront is a highly cost-effective solution, as you only pay for the storage and data transfer you use.
- **Flexibility:** The setup can be easily customized to suit your specific needs, such as adding a custom domain or more complex caching behaviors.

By using this Terraform setup, you can have a production-ready static website up and running in minutes.
