# GitHub Actions AWS Access Demo

This repository demonstrates how to use GitHub Actions with OpenID Connect (OIDC) to securely access AWS services without storing long-lived credentials.

## 🎯 What This Demo Shows

- How to configure GitHub Actions to use OIDC for AWS authentication
- Exchange GitHub's OIDC token for temporary AWS credentials
- Execute AWS CLI commands using the assumed role
- Verify AWS identity using `aws sts get-caller-identity`

## 🔧 Prerequisites

- An AWS account with administrative access
- A GitHub repository (this one!)
- Basic understanding of AWS IAM roles and policies

## 🚀 Setup Instructions

### Step 1: Create an IAM OIDC Identity Provider

1. Open the AWS IAM console
2. Navigate to **Identity providers** → **Add provider**
3. Select **OpenID Connect**
4. Configure the provider:
   - **Provider URL**: `https://token.actions.githubusercontent.com`
   - **Audience**: `sts.amazonaws.com`
5. Click **Add provider**

### Step 2: Create an IAM Role

1. In IAM console, go to **Roles** → **Create role**
2. Select **Web identity** as trusted entity type
3. Choose the GitHub OIDC provider you just created
4. Set **Audience** to `sts.amazonaws.com`
5. Add permissions (start with minimal permissions like `ReadOnlyAccess`)
6. Name your role (e.g., `GitHubActionsRole`)

### Step 3: Configure the Trust Policy

Edit your role's trust policy to restrict access to your specific repository:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:YOUR_USERNAME/YOUR_REPOSITORY:*"
                }
            }
        }
    ]
}
```

Replace:
- `YOUR_ACCOUNT_ID` with your AWS account ID
- `YOUR_USERNAME` with your GitHub username
- `YOUR_REPOSITORY` with this repository name

### Step 4: Configure Repository Secrets

1. In your GitHub repository, go to **Settings** → **Secrets and variables** → **Actions**
2. Add a new repository secret:
   - **Name**: `AWS_ROLE_TO_ASSUME`
   - **Value**: `arn:aws:iam::YOUR_ACCOUNT_ID:role/GitHubActionsRole`

### Step 5: Test the Workflow

1. Go to **Actions** tab in your repository
2. Select **AWS OIDC Demo** workflow
3. Click **Run workflow** to trigger manually
4. Monitor the workflow execution

## 📋 Expected Output

When the workflow runs successfully, you should see output similar to:

```
Verifying AWS identity...
{
    "UserId": "AROAEXAMPLE:GitHub-Actions-OIDC-Demo",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/GitHubActionsRole/GitHub-Actions-OIDC-Demo"
}
```

## 🛠 Workflow Details

The [`aws-demo.yml`](.github/workflows/aws-demo.yml) workflow:

1. **Checks out code** using `actions/checkout@v4`
2. **Configures AWS credentials** using `aws-actions/configure-aws-credentials@v4`
3. **Assumes the IAM role** using OIDC token exchange
4. **Verifies identity** by running `aws sts get-caller-identity`
5. **Displays session information** including AWS CLI version and region

### Key Configuration Points

- **Permissions**: `id-token: write` is required for OIDC token generation
- **Role assumption**: Uses the role ARN stored in repository secrets
- **Session naming**: Uses `GitHub-Actions-OIDC-Demo` for easy identification
- **Region**: Defaults to `us-east-1` (configurable)

## 🔒 Security Benefits

- **No long-lived credentials**: No AWS access keys stored in GitHub
- **Temporary sessions**: Credentials expire automatically
- **Fine-grained access**: Trust policy restricts access to specific repositories
- **Audit trail**: All actions are logged in AWS CloudTrail

## 🐛 Troubleshooting

### Common Issues

**1. "No assume role policy found for role"**
- Verify the IAM role exists and has a trust policy
- Check that the trust policy includes the correct OIDC provider ARN

**2. "Not authorized to perform sts:AssumeRoleWithWebIdentity"**
- Verify the trust policy conditions match your repository
- Ensure `token.actions.githubusercontent.com:sub` condition is correct

**3. "An error occurred (AccessDenied) when calling the GetCallerIdentity operation"**
- Check that the role has necessary permissions
- Verify the AWS region is correctly configured

**4. "Could not assume role with OIDC"**
- Ensure the OIDC provider URL is exactly: `https://token.actions.githubusercontent.com`
- Verify the audience is set to `sts.amazonaws.com`

### Debug Steps

1. Check AWS CloudTrail logs for detailed error messages
2. Verify the OIDC provider configuration in IAM
3. Test the trust policy using AWS CLI: `aws sts assume-role-with-web-identity`
4. Ensure repository secrets are correctly configured

## 📚 Additional Resources

- [AWS IAM OIDC Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
- [GitHub OIDC Documentation](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.