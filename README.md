# Configure AWS Credential Profiles for GitHub Actions

The primary reason this action exists is to address using multiple profiles at the same time. The default region is set to `us-east-1`

This action uses the official `aws-actions/configure-aws-credentials@v3` action. Almost all inputs of the original action are supported. The main difference is that the `profile` input is required. This action will create the `~/.aws/config` and `~/.aws/credentials` files with the specified profile name (can be overridden with `AWS_CONFIG_FILE` & `AWS_SHARED_CREDENTIALS_FILE`)

The official action is not sufficient for multiple account usage as it can only set one set of AWS environment variables at a time.



## Additional Inputs

Additional inputs over the official action are:

| Input Name   | Description                                                                    | Required | default values |
| :----------- | :----------------------------------------------------------------------------- | :------: | :------------: |
| profile      | Name of the profile to be created                                              |  `true`  |    default     |
| only-profile | This will unset the AWS env vars to empty string. Necessary for using profiles | `false`  |     `true`     |
| whoami       | Run additional `aws sts get-caller-identity` to check if login was success     |  `true`  |    `false`     |

# Unsupported Inputs

|     Input Name     | Description                                                     |
| :----------------: | :-------------------------------------------------------------- |
| output-credentials | If you need profiles you probably don't need the action outputs |


# Usage
```yaml
env:
  PROFILE_TEST: test
  PROFILE_PRODUCTION: production

jobs:
  test_new_action:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Repo
      uses: actions/checkout@v3

    - uses: mcblair/configure-aws-profile-action@v3
      with:
        role-to-assume: arn:aws:iam::987654321987:role/<ROLE_NAME>
        profile: ${{ env.PROFILE_TEST }}
        whoami: true

    - uses: mcblair/configure-aws-profile-action@v3
      with:
        role-to-assume: arn:aws:iam::123456789123:role/<ROLE_NAME>
        profile: ${{ env.PROFILE_PRODUCTION }}
        region: us-east-2

    - run: aws s3 ls --profile ${{ env.PROFILE_TEST }}

    - run: aws s3 ls --profile ${{ env.PROFILE_PRODUCTION }}
```

# Custom location for profiles

If you need to use a custom location for the profile files, you can use the `AWS_CONFIG_FILE` and `AWS_SHARED_CREDENTIALS_FILE` environment variables.

This sometimes becomes necessary for actions running in docker containers, where the default location is different that what the AWS SDK expects.

```yaml
env:
  PROFILE_TEST: test
  AWS_CONFIG_FILE: ${{ github.workspace }}/.aws/config # Custom path that is passed on to docker container /home/runner/work/<OWNER>/<REPO_NAME>/.aws/config
  AWS_SHARED_CREDENTIALS_FILE: ${{ github.workspace }}/.aws/credentials # Similar to above

jobs:
  test_new_action:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Repo
      uses: actions/checkout@v3

    - uses: mcblair/configure-aws-profile-action@v3
      with:
        role-to-assume: arn:aws:iam::987654321987:role/<ROLE_NAME>
        profile: ${{ env.PROFILE_TEST }}
        whoami: true

    - run: aws s3 ls --profile ${{ env.PROFILE_TEST }}

    - name: Terragrunt plan
      id: terragrunt-plan
      uses: gruntwork-io/terragrunt-action@v1
      env:
        AWS_CONFIG_FILE: /github/workspace/.aws/config # And this is the place the AWS config file ends up mounted
        AWS_SHARED_CREDENTIALS_FILE: /github/workspace/.aws/credentials
        AWS_PROFILE: ${{ env.PROFILE_TEST }}
```
