# Terraform Project Example: AWS Glue + S3 Infrastructure

This tutorial gives you a **complete, practical example** of a Terraform project that deploys:
- An **S3 bucket** for scripts and data
- An **IAM Role** for AWS Glue
- An **AWS Glue Job** that runs a Python script stored in S3

It’s designed to be **beginner-friendly but realistic**. You can copy, adapt, and deploy your own version easily.

---

## Typical Terraform Project Structure

```
terraform-exercise-1/
├── main.tf              # Core infrastructure definitions
├── variables.tf         # Input variables
├── outputs.tf           # Useful outputs (names, ARNs, URIs)
├── terraform.tfvars     # Values assigned to variables
├── glue-scripts/
│   └── demo-script.py   # Glue ETL script uploaded to S3
```

---

## main.tf — Core Infrastructure

This is the **heart** of the project. It defines which provider you’re using and the AWS resources Terraform should create.

```hcl
# Use AWS as the provider
provider "aws" {
  region = var.aws_region
}

# Create an S3 bucket for Glue scripts
resource "aws_s3_bucket" "glue_bucket" {
  bucket = var.bucket_name
}

# IAM Role for AWS Glue to assume
resource "aws_iam_role" "glue_service_role" {
  name = var.iam_role_name

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect = "Allow",
      Principal = { Service = "glue.amazonaws.com" },
      Action = "sts:AssumeRole"
    }]
  })
}

# Attach AWS managed policy for Glue service access
resource "aws_iam_role_policy_attachment" "glue_policy" {
  role       = aws_iam_role.glue_service_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole"
}

# Define a Glue Job
resource "aws_glue_job" "demo_job" {
  name     = var.glue_job_name
  role_arn = aws_iam_role.glue_service_role.arn

  command {
    name            = "glueetl"
    python_version  = "3"
    script_location = "s3://${aws_s3_bucket.glue_bucket.bucket}/glue-scripts/demo-script.py"
  }

  default_arguments = {
    "--job-language" = "python"
    "--TempDir"      = "s3://${aws_s3_bucket.glue_bucket.bucket}/tmp/"
  }
}
```

**Explanation**
- **provider**: Defines AWS as the cloud provider.
- **aws_s3_bucket**: Creates the S3 bucket that stores the Glue script.
- **aws_iam_role**: Grants Glue permission to access AWS services.
- **aws_glue_job**: Defines a Glue ETL job that runs your Python script.

---

## variables.tf — Declaring Variables

This file declares parameters that can be customized for different environments.

```hcl
variable "aws_region" {
  description = "AWS region where resources will be created"
  type        = string
  default     = "eu-west-1"
}

variable "bucket_name" {
  description = "Name of the S3 bucket for Glue scripts"
  type        = string
}

variable "iam_role_name" {
  description = "Name of the IAM role for AWS Glue"
  type        = string
}

variable "glue_job_name" {
  description = "AWS Glue Job name"
  type        = string
}
```

**Explanation**
- Keeps your project flexible and reusable.
- Variables can be set directly in `terraform.tfvars` or passed from CLI.

---

## terraform.tfvars — Variable Values

This file contains real values for the variables declared above.

```hcl
aws_region    = "eu-west-1"
bucket_name   = "my-glue-script-bucket"
iam_role_name = "glue-service-role-demo"
glue_job_name = "demo-glue-job"
```

**Explanation**
- You can have multiple `.tfvars` files (like `dev.tfvars`, `prod.tfvars`) to manage environments easily.

---

## outputs.tf — Useful Resource Outputs

Outputs make it easy to retrieve resource names, ARNs, and URIs after deployment.

```hcl
output "bucket_name" {
  value = aws_s3_bucket.glue_bucket.bucket
}

output "bucket_uri" {
  value = "s3://${aws_s3_bucket.glue_bucket.bucket}"
}

output "iam_role_arn" {
  value = aws_iam_role.glue_service_role.arn
}

output "glue_job_name" {
  value = aws_glue_job.demo_job.name
}

output "script_location" {
  value = aws_glue_job.demo_job.command.script_location
}
```

**Explanation**
- Outputs are shown after every `terraform apply`.
- They help integrate your Terraform outputs with other tools or automation scripts.

---

## glue-scripts/demo-script.py — Glue ETL Script

This is the Python script that runs inside the Glue job.

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

args = getResolvedOptions(sys.argv, ["JOB_NAME"])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args["JOB_NAME"], args)

# Example transform
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="default",
    table_name="sample_data"
)
glueContext.write_dynamic_frame.from_options(
    frame=datasource,
    connection_type="s3",
    connection_options={"path": "s3://my-glue-script-bucket/output/"},
    format="csv"
)
job.commit()
```

**Explanation**
- Reads a dataset from Glue Catalog.
- Writes results back to S3 as CSV.

---

## 🪄 Commands to Run the Project

Initialize, plan, and apply your infrastructure:

```bash
terraform init
terraform plan
terraform apply -auto-approve
```

When finished, clean everything up:

```bash
terraform destroy -auto-approve
```

---

## Adapting for Your Own Needs

| Part | What You Change | Why |
|------|-----------------|-----|
| `bucket_name` | Any unique S3 bucket name | Buckets must be globally unique |
| `iam_role_name` | Company/project-specific name | To separate permissions |
| `glue_job_name` | Your job’s logical name | Helps with identification |
| `script_location` | Path to your script in S3 | Glue needs this to run |
| `aws_region` | Your AWS region | Must match your AWS account setup |

---

## Summary

By following this structure, you now have a **real, working Terraform project** that provisions an **AWS Glue Job**, **S3 bucket**, and **IAM role** — and it’s easily adaptable to your own projects.

Next steps you could try:
- Add multiple Glue jobs for ETL pipelines  
- Create an Athena database to query output data  
- Add Lambda triggers for Glue workflows

---

### Key Terraform Commands Recap
| Command | Purpose |
|----------|----------|
| `terraform init` | Initialize provider plugins |
| `terraform plan` | Preview infrastructure changes |
| `terraform apply` | Deploy resources |
| `terraform destroy` | Tear down everything |

---

Now you have a **solid Terraform base project** you can learn from, extend, and adapt. 
