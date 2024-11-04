# EC2 Instance Deployment with CloudFormation and CI/CD Pipeline

This repository contains an AWS CloudFormation template for deploying an EC2 instance, along with a GitHub Actions CI/CD pipeline for automated deployment. The pipeline includes validation, security checks, manual approval, and triggers a parent stack deployment.

## Table of Contents

- [Introduction](#introduction)
- [CloudFormation Template](#cloudformation-template)
  - [Resources Created](#resources-created)
  - [Parameters](#parameters)
  - [Outputs](#outputs)
- [CI/CD Pipeline](#cicd-pipeline)
  - [Workflow Triggers](#workflow-triggers)
  - [Pipeline Overview](#pipeline-overview)
  - [Environment Variables and Secrets](#environment-variables-and-secrets)
- [Usage](#usage)
  - [Clone the Repository](#clone-the-repository)
  - [Set Up AWS Credentials](#set-up-aws-credentials)
  - [Configure the EC2 Instance](#configure-the-ec2-instance)
  - [Branch Strategy](#branch-strategy)
  - [Manual Approval](#manual-approval)
  - [Triggering the Parent Stack](#triggering-the-parent-stack)
- [Notes](#notes)

## Introduction

This project automates the deployment of an EC2 instance in AWS using CloudFormation. The instance is configured with necessary IAM roles and security groups to enable Systems Manager (SSM) access without exposing it to the internet.

The GitHub Actions CI/CD pipeline automates validation, security scanning, deployment processes, and triggers the deployment of a parent stack via webhooks.

The CI/CD pipeline is designed to:

- Validate and test CloudFormation templates on pull requests.
- Deploy infrastructure on pushes to specific branches.
- Perform security checks using CloudFormation Guard.
- Require manual approval before deployment.
- Trigger a parent stack deployment after successful deployment.

## CloudFormation Template

### Resources Created

The CloudFormation template (`template.yaml`) provisions the following resources:

1. **IAM Role for SSM Access (`SSMRole`):**

   - **Role Name:** `${SSMRoleName}_${Environment}`
   - **Assume Role Policy:** Allows EC2 service to assume the role.
   - **Managed Policies:**
     - `AmazonSSMManagedInstanceCore` for SSM access.

2. **IAM Instance Profile (`SSMInstanceProfile`):**

   - **Instance Profile Name:** `${ProfileName}-${Environment}`
   - **Roles:** Includes the `SSMRole`.

3. **Security Group for EC2 Instance (`EC2SecurityGroup`):**

   - **Group Description:** Security Group for EC2 instance.
   - **VPC ID:** Provided via `VPCId` parameter.
   - **Ingress Rules:**
     - **ICMP (Ping):** From Azure VNet CIDR (`AzureCIDR` parameter).
     - **All Traffic:** From VPC CIDR (`VPCCIDR` parameter).
   - **Egress Rule:**
     - **All Traffic:** To `0.0.0.0/0` (necessary for SSM access).
   - **Tags:** Includes `Name` with value `s2sEc2Sg_{Environment}`.

4. **EC2 Instance (`EC2Instance`):**

   - **Instance Type:** Defined by `InstanceType` parameter (default is `t2.micro`).
   - **AMI ID:** Defined by `ImageID` parameter (default is `ami-07c5ecd8498c59db5`).
   - **Network Interfaces:**
     - **Subnet ID:** Provided via `SubnetId` parameter.
     - **Security Groups:** Includes `EC2SecurityGroup`.
     - **Associate Public IP Address:** `false` (private instance).
   - **IAM Instance Profile:** `SSMInstanceProfile`.
   - **Source/Destination Check:** `false` (allows the instance to forward traffic).
   - **Tags:** Includes `Name` with value `EC2_{Environment}`.

### Parameters

The template accepts the following parameters:

- **Environment:**

  - **Type:** String
  - **Description:** The environment name (provided via Github Workflows pipeline)
  - **Allowed Values:** `development`, `production`, `default`

- **InstanceType:**

  - **Type:** String
  - **Description:** The EC2 instance type
  - **Default:** `t2.micro`

- **ImageID:**

  - **Type:** String
  - **Description:** The AMI ID for the EC2 instance
  - **Default:** `ami-07c5ecd8498c59db5` (Amazon Linux 2 AMI)

- **SubnetId:**

  - **Type:** String
  - **Description:** Subnet ID from the VPC template

- **VPCId:**

  - **Type:** String
  - **Description:** VPC ID from the VPC template

- **VPCCIDR:**

  - **Type:** String
  - **Description:** The CIDR range of the VPC

- **PolicyName:**

  - **Type:** String
  - **Description:** Name for the IAM policy
  - **Default:** `SSMPolicy`

- **ProfileName:**

  - **Type:** String
  - **Description:** Name for the IAM instance profile
  - **Default:** `S2S-EC2-SSMProfile`

- **SSMRoleName:**

  - **Type:** String
  - **Description:** Name of the SSM Role to be created
  - **Default:** `S2S_SSM_Role`

- **EC2SGName:**

  - **Type:** String
  - **Description:** Name of the security group attached to the EC2 instance
  - **Default:** `S2SEC2SG`

- **AzureCIDR:**

  - **Type:** String
  - **Description:** The CIDR block of the Azure VNet
  - **Default:** `10.0.0.0/16`

### Outputs

- **InstanceId:**

  - **Description:** ID of the created EC2 instance
  - **Value:** Reference to `EC2Instance`

## CI/CD Pipeline

The CI/CD pipeline is defined in the GitHub Actions workflow file `.github/workflows/aws-cloudformation.yml`. It automates the deployment process, ensures code quality and security, and triggers the deployment of a parent stack.

### Workflow Triggers

The pipeline is triggered on:

- **Pull Requests** to the following branches:
  - `development`
  - `production`
  - `testing`
- **Pushes** to the following branches:
  - `development`
  - `production`

### Pipeline Overview

The pipeline consists of two primary jobs:

1. **Validate and Test (`validate-and-test`):**

   - **Checkout Code:** Retrieves the repository code.
   - **Configure AWS Credentials:** Assumes an IAM role using OpenID Connect (OIDC) with the provided credentials.
   - **Validate CloudFormation Template:** Validates the syntax of the CloudFormation template.
   - **Run CloudFormation Guard:** Performs security checks using CloudFormation Guard against CIS benchmarks. Continues on error due to necessary security group configurations.
   - **Skip Apply in Pull Requests:** Ensures that deployment does not occur on pull requests.

2. **Upload and Trigger (`upload-and-trigger`):**

   - **Depends On:** The `validate-and-test` job must succeed.
   - **Runs On:** Not triggered on pull requests.
   - **Checkout Code:** Retrieves the repository code.
   - **Configure AWS Credentials:** Assumes an IAM role using OIDC with the provided credentials.
   - **Set Environment Variable:** Determines the environment and S3 bucket based on the branch.
   - **Manual Approval:** Requires manual approval via GitHub Issues before proceeding.
   - **Upload Template to S3:** Uploads the CloudFormation template to the specified S3 bucket.
   - **Trigger Parent Stack Deployment:** Makes an API call to trigger the parent stack's workflow via webhooks.

### Environment Variables and Secrets

The pipeline uses the following secrets and environment variables:

- **Secrets (Stored in GitHub Secrets):**
  - `AWS_ROLE_TO_ASSUME`: ARN of the IAM role to assume for deployment.
  - `TOKEN_PARENT_TRIGGER`: Personal Access Token (PAT) with repository permissions to trigger workflows in the parent repository.
  - `github_TOKEN`: Automatically provided by GitHub for authentication in workflows.

- **Environment Variables:**
  - `ENVIRONMENT`: Set based on the branch (`development`, `production`, or `default`).
  - `S3_BUCKET`: The S3 bucket URL, determined by the environment.

## Usage

### Clone the Repository

```bash
git clone https://github.com/CommittingLearning/Site2Site-AWS-EC2.git
```

### Set Up AWS Credentials

Ensure that the following secrets are added to your GitHub repository under **Settings > Secrets and variables > Actions**:

- `AWS_ROLE_TO_ASSUME`

- `TOKEN_PARENT_TRIGGER`

- **`AWS_ROLE_TO_ASSUME`**: The ARN of an IAM role that the GitHub Actions workflow can assume, with the necessary permissions to deploy CloudFormation stacks and interact with S3.

- **`TOKEN_PARENT_TRIGGER`**: A GitHub Personal Access Token (PAT) with permissions to trigger workflows in the parent repository.

### Configure the EC2 Instance

- **Instance Type**: Adjust the `InstanceType` parameter in the `template.yaml` file if you need a different EC2 instance type.

- **AMI ID**: Change the `ImageID` parameter to the desired AMI ID for your instance.

- **Subnet ID and VPC ID**: These are provided by the outputs from the VPC stack. Ensure that you supply the correct IDs when deploying the parent stack.

- **Environment Parameter**: The environment (`development`, `production`, or `default`) is determined based on the branch name.

### Branch Strategy

- **Development Environment**: Use the `development` branch to deploy to the development environment.

- **Production Environment**: Use the `production` branch to deploy to the production environment.

- **Default Environment**: Any other branches will use the `default` environment settings.

### Manual Approval

The pipeline requires manual approval before applying changes:

- A GitHub issue will be created prompting for approval.

- Approvers need to approve the issue to proceed with deployment.

### Triggering the Parent Stack

After successful deployment, the pipeline triggers the parent stack deployment via a webhook:

- The last step in the `upload-and-trigger` job makes an API call to the parent repository (`Site2Site-AWS-ParentStack`) to trigger its workflow.

- This ensures that the parent stack is deployed with the latest updates.

## Notes

- **Security Checks**:

  - The pipeline includes security checks using CloudFormation Guard to ensure compliance with CIS benchmarks.

  - The `continue-on-error: true` is set because the security group allows full outbound access, which is necessary for SSM but may violate CIS benchmarks.

- **Nested CloudFormation Templates**:

  - The EC2 template is intended to be uploaded to an S3 bucket and used as a child template by a parent stack.

- **Testing**:

  - Pull requests to `development`, `production`, or `testing` branches will trigger the validation and testing steps without applying changes.

- **IAM Role Configuration**:

  - Ensure that the IAM role specified in `AWS_ROLE_TO_ASSUME` has permissions to deploy CloudFormation stacks, interact with S3, and invoke API calls to other repositories.

- **S3 Bucket Configuration**:

  - The S3 bucket URL is set based on the environment. Ensure that the bucket exists and is accessible.

- **Webhook Trigger**:

  - The `TOKEN_PARENT_TRIGGER` must have appropriate permissions (usually `repo` scope) to trigger workflows in the parent repository.

  - Adjust the API endpoint and payload in the `Trigger Parent Stack Deployment` step if your repository names or structures differ.

---

**Disclaimer:** This repository is accessible in a read only format, and therefore, only the admin has the privileges to perform a push on the branches.