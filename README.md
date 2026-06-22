# 🚀 AWS OIDC Integration Setup Guide (Azure DevOps)

This guide contains the configurations required to establish a secure, keyless OpenID Connect (OIDC) handshake between Azure DevOps and global AWS regions (e.g., `us-east-1`). This setup eliminates the need for long-lived AWS Access Keys.

*Please refer to the official repo for more knowledge*: [OIDC in AWS - AWS OIDC Documentation](https://aws.amazon.com/blogs/modernizing-with-aws/how-to-federate-into-aws-from-azure-devops-using-openid-connect/)

## 🛠️ Part 1: AWS IAM Configuration
Before running the pipeline, you must configure AWS to trust your specific Azure DevOps organization as an Identity Provider.

### 1. Find your Organization GUID
Azure DevOps hosts the OIDC configuration endpoint under your organization's unique `instanceId` (GUID). The easiest way to find this is to query the Azure DevOps API directly from your browser:

Open: 
```url
https://dev.azure.com/{org-name}/_apis/connectionData
```
Copy the value of instanceId from the JSON response.

- **Your Instance ID:** `3a37fa8e-a3ce-4d4f-a839-778b53003020`

### 2. Create the Identity Provider
In the AWS Console, navigate to **IAM > Identity providers > Add provider**:
- **Provider type:** OpenID Connect
- **Provider URL:** ```https://vstoken.dev.azure.com/{your-org-GUID}```
- **Audience:** `api://AzureADTokenExchange`

### 3. Create the IAM Role Trust Policy
Create an IAM Role named `ado-role-test` (`ARN: arn:aws:iam::088862082874:role/ado-role-test`) and attach the following Trust Policy.
> ⚠️ Security Note: The Condition block explicitly restricts this role so it can only be assumed by the `aws-oidc-sc` service connection inside the `AWS-complete-infra-project` project of the `satyam-lko` organization.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::088862082874:oidc-provider/vstoken.dev.azure.com/3a37fa8e-a3ce-4d4f-a839-778b53003020"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "vstoken.dev.azure.com/3a37fa8e-a3ce-4d4f-a839-778b53003020:aud": "api://AzureADTokenExchange",
          "vstoken.dev.azure.com/3a37fa8e-a3ce-4d4f-a839-778b53003020:sub": "sc://satyam-lko/AWS-complete-infra-project/aws-oidc-sc"
        }
      }
    }
  ]
}

```

## 💻 Part 2: Azure DevOps Pipeline Workflow

Before running this YAML file, create a Service Connection in your Azure DevOps project (AWS-complete-infra-project):

Authentication Method: Workload Identity Federation (OIDC)

- Type: `AWS`
- **Role ARN:**  `arn:aws:iam::123456789:role/ado-role-test`
- Create a file named `azure-pipelines.yml` in the root of your repository and use the configuration below:

```yaml
name: Azure DevOps AWS OIDC

trigger:
    - main

pool:
  name: 'laptop-runner'

variables:
  aws.rolecredential.maxduration: "3600" # Can be set from 900 - 43200s
  
steps:
    - task: AWSCLI@1
      displayName: "Running aws-cli get-caller-identity"
      # continueOnError: true # If you need the pipeline to succeed
      inputs:
        awsCredentials: "aws-oidc-sc"
        regionName: 'us-east-1'
        awsCommand: 'sts'
        awsSubCommand: 'get-caller-identity'
```
---
### Service Connection Name: aws-oidc-sc (This must match the name in your YAML and Trust Policy)

<img width="598" height="873" alt="AWD_OIDC_Connection" src="https://github.com/user-attachments/assets/c45a2c40-f97d-4863-9567-0c59c7957b57" />
