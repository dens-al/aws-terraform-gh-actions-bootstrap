# Terraform Bootstrap Infrastructure

This CloudFormation `terraform-bootstrap.yml` template sets up the foundation for managing Terraform state and GitHub Actions integration.

## Overview

```mermaid
graph LR
    %% Define styles
    classDef github fill:#f5f5f5,stroke:#333,stroke-width:1px;
    classDef aws fill:#F2F7FD,stroke:#232F3E,stroke-width:1px;
    classDef iamRole fill:#fff,stroke:#FF9900,stroke-width:2px;
    classDef s3 fill:#E9F8FE,stroke:#3B48CC,stroke-width:2px;
    classDef dynamo fill:#F8F1F9,stroke:#3B48CC,stroke-width:2px;
    
    %% Define nodes
    GH[GitHub Repository] --> GA[GitHub Actions]
    AWS[AWS Account]
    
    %% IAM resources
    GA -- "OIDC" --> IAM[IAM OIDC Provider]
    IAM --> ROLE[GitHubActionsRole]
    
    %% Terraform backend resources
    ROLE --> S3[S3 Bucket<br/>Terraform State]
    ROLE --> DDB[DynamoDB<br/>State Locking]
    
    %% Subgraphs
    subgraph GitHub
        GH
        GA
    end
    
    subgraph AWS
        IAM
        ROLE
        S3
        DDB
    end
    
    %% Apply classes
    class GitHub github;
    class AWS aws;
    class ROLE iamRole;
    class S3 s3;
    class DDB dynamo;
    
    %% Add annotations
    S3 -.- S3Note[Versioned<br/>Encrypted<br/>SSL-enforced]
    DDB -.- DDBNote[LockID<br/>Pay-per-request]
    ROLE -.- ROLENote[Has permissions<br/>to access S3 & DynamoDB]
```

This stack creates:

- **S3 bucket** for Terraform state (versioned, encrypted, SSL-enforced)
- **DynamoDB table** for state locking
- **OIDC provider** for GitHub Actions integration
- **IAM role** with necessary permissions

## Quick Start

### Deploy the CloudFormation Stack

```bash
aws cloudformation create-stack \
  --stack-name terraform-backend-bootstrap \
  --template-body file://terraform-bootstrap.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

### Configure Terraform Backend

Use the stack outputs to configure your Terraform backend:

```terraform
terraform {
  backend "s3" {
    bucket         = "<TerraformStateBucketName-from-outputs>"
    key            = "<your-state-file-path>"
    region         = "<your-region>"
    dynamodb_table = "<TerraformLockDynamoDBTableName-from-outputs>"
    encrypt        = true
  }
}
```

### Configure GitHub Actions

Add this to your GitHub Actions workflow:

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: <GitHubActionsRoleARN-from-outputs>
          aws-region: <your-region>
```

## Security Features

- Server-side encryption for S3 and DynamoDB
- SSL enforcement for S3 access
- Public access blocking
- OIDC authentication (no long-lived credentials)

## Parameters

| Parameter | Description | Default |
|-----------|-------------|--------|
| BucketNamePrefix | Prefix for S3 bucket name | dens-al |
| DynamoDBTableNamePrefix | Prefix for DynamoDB table name | dens-al |
| GitHubOrg | GitHub organization name | dens-al |
| GitHubRepo | Repository name/pattern | aws-terraform-gh-actions-bootstrap |
| GitHubRoleName | IAM role name | GitHubActionsRole |

## AWS Organisation

In case of using `AWS Organisation` it is good idea to use [Telophase](https://docs.telophase.dev/introduction) which can use CloudFormation or Terraform code as bootstrap script
