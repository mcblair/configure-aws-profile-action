# Configure AWS Credential Profiles for GitHub Actions
This action uses the official `aws-actions/configure-aws-credentials@v2` action. This action only supports assuming roles via OIDC.

The official action is not sufficient for multiple account usage as it can only set one set of AWS environment variables at a time.

> The primary reason this action exists is to address using multiple profiles at the same time. Region defaults to `us-west-2`.

# Usage
```yaml
jobs:  
  test_new_action:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Repo
      uses: actions/checkout@v3

    - uses: mcblair/configure-aws-profile@v0.1.0
      with:
        role-arn: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
        profile-name: test

    - uses: mcblair/configure-aws-profile@v0.1.0
      with:
        role-arn: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
        profile-name: production
        region: us-east-2

    - run: aws s3 ls --profile test

    - run: aws s3 ls --profile production
```
