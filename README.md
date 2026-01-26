# AWS Integration Setup Guide

This guide provides step-by-step instructions for setting up AWS integration with Datadog using Terragrunt.

## 📋 Prerequisites

Before starting, ensure you have:
- AWS Account ID
- AWS Account Alias Name (as per Asset Naming Convention)
- AWS regions to monitor
- AWS namespaces to monitor (e.g., AWS/SNS, AWS/Lambda, AWS/RDS)
- Access to the repository

## 🎯 Asset Naming Convention

The folder name should follow the HP Asset Naming Convention:
- Refer to: 
https://pages.github.azc.ext.hp.com/SW-R-D-Architecture/architecture/architecture/hpip/overview/#model
https://pages.github.azc.ext.hp.com/SW-R-D-Architecture/architecture/adrs/ARCH-1886/#naming-convention

- Example: `hpip-devex-infra`
- Format typically follows: `Structure : hpip.{area}.{category}.{asset}.{sub-asset}


## 📁 Directory Structure

The AWS integration follows this structure:
```
hpip-tio/
└── [service-category]/
    ├── service-category.hcl
    └── integrations/
        └── aws/
            └── [aws-account-alias]/
                └── terragrunt.hcl
```

## 🚀 Step-by-Step Setup

### Step 1: For the AWS account to integrate with Datadog, identify Your Service Category

**Example**: For HPIP Platform Infra and Engineering & Standards services, use `hpip-devex-infra`

### Step 2: Create the AWS Integration Directory Structure

Create the following directory structure:

```bash
cd hpip-tio/[service-category]/
mkdir -p integrations/aws/[aws-account-alias]
```

**Example**:
```bash
cd hpip-tio/hpip-cloud-iam/
mkdir -p integrations/aws/tropos-stratus-myid-bridge-stg-pro
```

### Step 3: Create the terragrunt.hcl File

Create a `terragrunt.hcl` file in the newly created directory:

```bash
cd integrations/aws/[aws-account-alias]
touch terragrunt.hcl
```

### Step 4: Configure terragrunt.hcl

Add the following configuration to `terragrunt.hcl`:

```hcl
locals {
  service_category_vars = read_terragrunt_config(find_in_parent_folders("service-category.hcl"))
}

terraform {
  source = "git::ssh://git@github.azc.ext.hp.com/runway-incubator/hpip-devex-runway-terraform-datadog//modules/integrations-aws-account?ref=${local.service_category_vars.locals.datadog_module_version}"
}

include {
  path = find_in_parent_folders()
}

inputs = {
  aws_account_id   = "[YOUR_AWS_ACCOUNT_ID]"
  aws_account_name = "[YOUR_AWS_ACCOUNT_ALIAS]"
  dd_team          = local.service_category_vars.locals.datadog_team_handle
  aws_regions = [
    "us-east-1",
    # Add more regions as needed
  ]
  dd_aws_namespaces = [
    "AWS/SNS",
    # Add more AWS service namespaces as needed
    # Common examples:
    # "AWS/Lambda",
    # "AWS/RDS",
    # "AWS/DynamoDB",
    # "AWS/ECS",
    # "AWS/EC2",
    # "AWS/S3",
  ]
}
```

### Step 5: Replace Placeholder Values

Update the following values in `terragrunt.hcl`:

| Parameter | Description | Example |
|-----------|-------------|---------|
| `aws_account_id` | Your AWS Account ID (12-digit number) | `523*****33` |
| `aws_account_name` | AWS Account Alias (matches folder name) | `tropos-stratus-myid-bridge-stg-pro` |
| `aws_regions` | List of AWS regions to monitor | `["us-east-1", "us-west-2"]` |
| `dd_aws_namespaces` | List of AWS service namespaces to monitor | `["AWS/SNS", "AWS/Lambda"]` |

**Important**: 
- The `aws_account_name` should match the folder name you created
- The `dd_team` is automatically pulled from the parent `service-category.hcl`

### Step 6: Verify service-category.hcl

Ensure your service category has the required variables defined in `service-category.hcl`:

```hcl
locals {
  project             = "[project-name]"
  datadog_team_handle = "[team-handle]"
  fqdn                = "[service-fqdn]"
  datadog_module_version = "v0.11.0"  # or latest version
}
```

### Step 7: Commit and Push Changes

```bash
git add integrations/aws/[aws-account-alias]/terragrunt.hcl
git commit -m "Add AWS integration for [aws-account-alias]"
git push
```

### Step 8: Apply Infrastructure Changes

Run Terragrunt to apply the changes:

```bash
cd hpip-tio/[service-category]/integrations/aws/[aws-account-alias]
terragrunt init
terragrunt plan
terragrunt apply
```


## 🔍 Common AWS Service Namespaces

| Service | Namespace |
|---------|-----------|
| Lambda | `AWS/Lambda` |
| RDS | `AWS/RDS` |
| DynamoDB | `AWS/DynamoDB` |
| ECS | `AWS/ECS` |
| EC2 | `AWS/EC2` |
| S3 | `AWS/S3` |
| SNS | `AWS/SNS` |
| SQS | `AWS/SQS` |
| API Gateway | `AWS/ApiGateway` |
| CloudFront | `AWS/CloudFront` |
| ELB | `AWS/ELB` |
| ALB | `AWS/ApplicationELB` |
| NLB | `AWS/NetworkELB` |

## ❓ Troubleshooting

### Issue: Cannot find parent folders
**Solution**: Ensure you're running terragrunt from within the correct directory structure under `hpip-tio/`

### Issue: Module version not found
**Solution**: Check that `datadog_module_version` in `service-category.hcl` points to a valid version

### Issue: AWS Account ID validation failed
**Solution**: Verify the AWS Account ID is a valid 12-digit number

### Issue: Permission denied
**Solution**: Ensure you have the necessary AWS and repository permissions

## 📚 Additional Resources

- [Datadog AWS Integration Documentation](https://docs.datadoghq.com/integrations/amazon_web_services/)
- [Terragrunt Documentation](https://terragrunt.gruntwork.io/)
- [HPIP Architecture Overview](https://pages.github.azc.ext.hp.com/SW-R-D-Architecture/architecture/architecture/hpip/overview/)

## 🤝 Support

For questions or issues:
1. Check the repository README.md
2. Review existing AWS integration examples in other service categories
3. Contact the team or your service category team lead
