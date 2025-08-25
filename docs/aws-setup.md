# AWS Setup Guide for GitHub Actions OIDC

This guide provides detailed instructions for setting up AWS IAM resources to enable GitHub Actions OIDC authentication.

## Prerequisites

- AWS account with administrative access
- AWS CLI installed and configured (optional, for testing)
- Basic understanding of AWS IAM concepts

## Step-by-Step Setup

### 1. Create OIDC Identity Provider

#### Using AWS Console

1. Open the [AWS IAM Console](https://console.aws.amazon.com/iam/)
2. Navigate to **Identity providers** in the left sidebar
3. Click **Add provider**
4. Select **OpenID Connect**
5. Enter the following details:
   - **Provider URL**: `https://token.actions.githubusercontent.com`
   - **Audience**: `sts.amazonaws.com`
6. Click **Get thumbprint** (should auto-populate)
7. Click **Add provider**

#### Using AWS CLI

```bash
aws iam create-open-id-connect-provider \
    --url https://token.actions.githubusercontent.com \
    --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1 \
    --client-id-list sts.amazonaws.com
```

### 2. Create IAM Role

#### Using AWS Console

1. In IAM Console, go to **Roles** → **Create role**
2. Select **Web identity** as the trusted entity type
3. Choose **GitHub** as the identity provider
4. Select the OIDC provider you created
5. Set **Audience** to `sts.amazonaws.com`
6. Click **Next**
7. Attach policies (start with minimal permissions):
   - `ReadOnlyAccess` (for basic testing)
   - Or create a custom policy with only required permissions
8. Click **Next**
9. Enter role details:
   - **Role name**: `GitHubActionsRole` (or your preferred name)
   - **Description**: Role for GitHub Actions OIDC authentication
10. Click **Create role**

#### Using AWS CLI

First, create a trust policy file:

```bash
cat > trust-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:USERNAME/REPOSITORY:*"
                }
            }
        }
    ]
}
EOF
```

Replace:
- `ACCOUNT_ID` with your AWS account ID
- `USERNAME` with your GitHub username
- `REPOSITORY` with your repository name

Create the role:

```bash
aws iam create-role \
    --role-name GitHubActionsRole \
    --assume-role-policy-document file://trust-policy.json
```

Attach a policy:

```bash
aws iam attach-role-policy \
    --role-name GitHubActionsRole \
    --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

### 3. Configure Trust Policy

The trust policy is crucial for security. Here's a breakdown of the conditions:

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

#### Trust Policy Conditions Explained

- **`token.actions.githubusercontent.com:aud`**: Ensures the token audience is `sts.amazonaws.com`
- **`token.actions.githubusercontent.com:sub`**: Restricts access to specific repository patterns

#### Advanced Trust Policy Examples

**Restrict to specific branch:**
```json
"StringEquals": {
    "token.actions.githubusercontent.com:sub": "repo:username/repository:ref:refs/heads/main"
}
```

**Restrict to pull requests:**
```json
"StringLike": {
    "token.actions.githubusercontent.com:sub": "repo:username/repository:pull_request"
}
```

**Multiple repositories:**
```json
"ForAnyValue:StringLike": {
    "token.actions.githubusercontent.com:sub": [
        "repo:username/repo1:*",
        "repo:username/repo2:*"
    ]
}
```

### 4. Permission Policies

#### Minimal Policy for This Demo

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sts:GetCallerIdentity"
            ],
            "Resource": "*"
        }
    ]
}
```

#### Common Additional Permissions

For typical deployment workflows, you might need:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sts:GetCallerIdentity",
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:ListBucket",
                "cloudformation:DescribeStacks",
                "cloudformation:CreateStack",
                "cloudformation:UpdateStack",
                "lambda:UpdateFunctionCode",
                "lambda:GetFunction"
            ],
            "Resource": "*"
        }
    ]
}
```

### 5. Verification

Test your configuration using AWS CLI:

```bash
# Get your account ID
aws sts get-caller-identity

# Verify the OIDC provider exists
aws iam list-open-id-connect-providers

# Check the role
aws iam get-role --role-name GitHubActionsRole

# Test role assumption (requires GitHub token)
aws sts assume-role-with-web-identity \
    --role-arn arn:aws:iam::ACCOUNT_ID:role/GitHubActionsRole \
    --role-session-name test-session \
    --web-identity-token $GITHUB_TOKEN
```

## Security Best Practices

1. **Principle of Least Privilege**: Only grant necessary permissions
2. **Specific Repository Restrictions**: Use exact repository paths in trust policy
3. **Branch/Environment Restrictions**: Limit to specific branches or environments
4. **Regular Auditing**: Review CloudTrail logs for unexpected access
5. **Session Duration**: Set appropriate session duration limits
6. **Conditional Access**: Use additional conditions like IP restrictions if needed

## Troubleshooting

### Common Error Messages

**"No OpenIDConnect provider found"**
- Verify the OIDC provider URL is exactly: `https://token.actions.githubusercontent.com`
- Check that the provider exists in your AWS account

**"Not authorized to perform sts:AssumeRoleWithWebIdentity"**
- Verify trust policy conditions match your repository
- Check that the audience is set to `sts.amazonaws.com`

**"Invalid identity token"**
- Ensure GitHub Actions has `id-token: write` permission
- Verify the token hasn't expired

**"Access denied"**
- Check the role's permission policies
- Verify the role ARN is correct in GitHub secrets

### Debug Commands

```bash
# Check OIDC providers
aws iam list-open-id-connect-providers

# Get role details
aws iam get-role --role-name GitHubActionsRole

# List attached policies
aws iam list-attached-role-policies --role-name GitHubActionsRole

# Get policy document
aws iam get-role-policy --role-name GitHubActionsRole --policy-name PolicyName
```

## Next Steps

After completing this setup:

1. Configure repository secrets in GitHub
2. Test the workflow with a manual trigger
3. Monitor CloudTrail logs for successful authentication
4. Gradually add more permissions as needed
5. Consider implementing multiple roles for different environments

## Resources

- [AWS IAM OIDC Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
- [GitHub OIDC Documentation](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [AWS STS Documentation](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html)