# Configuring OIDC for GitHub Actions with AWS

[Configure AWS Credentials](https://github.com/aws-actions/configure-aws-credentials),
[Configure OIDC in AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)

OpenID Connect (OIDC) allows GitHub repositories to securely access AWS services by assuming an IAM role, eliminating the need for long-lived credentials. 

This approach enhances security by eliminating the need for static access keys, reducing the risk of credential leaks and unauthorized access.
## Overview
By using OIDC, each GitHub repository assumes an AWS IAM role within its Action workflows to deploy applications to AWS. This setup provides better security and easier separation of permissions per repository.

## Step 1: Grant GitHub OIDC Access to Your AWS Account
To enable GitHub OIDC authentication in AWS:

1. Navigate to **IAM** in the AWS Management Console.
2. Select **Identity providers** and click **Add provider**.
3. Choose **OpenID Connect (OIDC)** as the provider type.
4. Set the provider URL to:
   ```
   https://token.actions.githubusercontent.com
   ```
5. Retrieve the thumbprint and confirm.
6. Set the **Audience** value to:
   ```
   sts.amazonaws.com
   ```
7. Click **Add provider**.

> **Note:** AWS updates the OIDC thumbprint sometimes. Ensure it is periodically refreshed to maintain authentication.

## Step 2: Create an IAM Role with a Trust Policy
Once GitHub has access to IAM, create an IAM role and configure a trust relationship with GitHub OIDC:

1. In the AWS Management Console, go to **IAM > Roles** and create a new role.
2. Under **Trusted entity type**, select **Web identity**.
3. Choose the identity provider added earlier (`token.actions.githubusercontent.com`).
4. Define a trust policy for GitHub Actions:
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Principal": {
                   "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
               },
               "Action": "sts:AssumeRoleWithWebIdentity",
               "Condition": {
                   "StringLike": {
                       "token.actions.githubusercontent.com:sub": "repo:<GITHUB_ORG>/<REPO_NAME>:*"
                   }
               }
           }
       ]
   }
   ```
5. Replace `<ACCOUNT_ID>` with your AWS account ID and `<GITHUB_ORG>/<REPO_NAME>` with the repository name.
6. Attach the necessary IAM policies to the role to define permissions.

## Step 3: Configure GitHub Actions to Assume the Role
To use the IAM role in your GitHub Actions workflow:

1. In your GitHub repository, go to **Settings > Secrets and variables > Actions**.
2. Add a new **Repository Secret**:
    - **Name:** `AWS_ROLE_TO_ASSUME`
    - **Value:** ARN of the IAM role created in AWS.

3. Update your GitHub Actions workflow (`.github/workflows/deploy.yml`) with the following:
   ```yaml
   permissions:
     id-token: write
     contents: read
   ```
4. Before executing AWS-related actions, add a step to configure AWS credentials:
   ```yaml
   - name: Configure AWS Credentials
     uses: aws-actions/configure-aws-credentials@v1
     with:
       aws-region: <AWS_REGION>
       role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
       role-session-name: GitHub-Deploy-User
   ```

## Best Practices
- Assign a dedicated IAM role per repository to enforce strict access controls.
- Follow the principle of least privilege when attaching IAM policies to the role.
- Periodically update the OIDC thumbprint to maintain security.
- Regularly review IAM policies and GitHub repository settings to ensure compliance with security best practices.

By following this setup, GitHub Actions can securely interact with AWS using short-lived credentials, reducing the risk associated with static access keys.

