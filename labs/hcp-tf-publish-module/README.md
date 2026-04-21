# terraform-aws-s3-bucket

A simple Terraform module to create an AWS S3 bucket.

## Usage

```hcl
module "s3_bucket" {
  source      = "<YOUR_ORG>/s3-bucket-{your-initials}/aws"
  bucket_name = "my-bucket"
}
``` 