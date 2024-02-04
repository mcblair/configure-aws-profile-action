# Configure AWS Credential Profiles for GitHub Actions

The primary reason this action exists is to address using multiple profiles at the same time in GitHub Actions. The [official action](https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions) is not sufficient for multiple account usage as it only sets one set of AWS environment variables at a time. Trying to reuse the action simply overwrites the environment variables with the new credentials.

This action uses the official `aws-actions/configure-aws-credentials@v4` under the hood. Almost all inputs of the original action are supported. The main difference is that the `profile` input is required. This action will create the `~/.aws/config` and `~/.aws/credentials` files with the specified profile name (can be overridden with `AWS_CONFIG_FILE` & `AWS_SHARED_CREDENTIALS_FILE`, details below).


## Additional Inputs

The below inputs are in additional to the [official action inputs](https://github.com/aws-actions/configure-aws-credentials/blob/v4/action.yml#L11) are:

| Input Name   | Description                                                                     | Required | default values |
| :----------- | :------------------------------------------------------------------------------ | :------: | :------------: |
| profile      | Name of the profile to be created                                               |  `true`  |    default     |
| only-profile | This will unset the AWS env vars to empty string. Necessary for using profiles  | `false`  |     `true`     |
| whoami       | Run additional `aws sts get-caller-identity` to check if the profile is working |  `true`  |    `false`     |

# Unsupported Inputs

| Input Name         | Description                                                                                                                                  |
| :----------------- | :------------------------------------------------------------------------------------------------------------------------------------------- |
| output-credentials | This is set to true so that the next steps in this action can create the profile. The actual env vars are then overridden with a dummy value |

# Outputs

| Output Name           | Description                                            |
| :-------------------- | :----------------------------------------------------- |
| aws-account-id        | The AWS account ID for the provided credentials        |
| aws-access-key-id     | The AWS access key ID for the provided credentials     |
| aws-secret-access-key | The AWS secret access key for the provided credentials |
| aws-session-token     | The AWS session token for the provided credentials     |
| profile               | Name of the profile that was created                   |


# Usage
```yaml
env:
  PROFILE_ROLE_1: PROFILE_ROLE_1
  PROFILE_ROLE_2: PROFILE_ROLE_2

jobs:
  assume-multiple-roles:
    runs-on: ubuntu-latest
    permissions:
      contents: read # This is required for actions/checkout
      id-token: write # This is required for requesting the token from AWS STS
    steps:
      - name: Checkout Repo
        id: checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials ROLE 1
        id: configure-aws-credentials-role-1
        uses: Moulick/configure-multiple-aws-roles@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/role-1
          role-session-name: GithubActions-${{ env.REPO_NAME }}-${{ github.workflow }}-${{ github.run_id }}
          aws-region: ${{ env.AWS_DEFAULT_REGION }}
          profile: ${{ env.PROFILE_ROLE_1 }}
          only-profile: true
          whoami: true

      - name: Configure AWS credentials ROLE 2
        id: configure-aws-credentials-role-2
        uses: Moulick/configure-multiple-aws-roles@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/role-2
          role-session-name: GithubActions-${{ env.REPO_NAME }}-${{ github.workflow }}-${{ github.run_id }}
          aws-region: ${{ env.AWS_DEFAULT_REGION }}
          profile: ${{ env.PROFILE_ROLE_2 }}
          only-profile: true
          whoami: true

      - name: Check AWS credentials
        id: check-aws-credentials
        run: |
          aws sts get-caller-identity --profile ${{ env.PROFILE_ROLE_1 }}
          aws sts get-caller-identity --profile ${{ env.PROFILE_ROLE_2 }}

```

# Custom location for AWS config and credentials files

If you need to use a custom location for the profile files, you can use the `AWS_CONFIG_FILE` and `AWS_SHARED_CREDENTIALS_FILE` environment variables.

This sometimes becomes necessary for actions running in docker containers, where the default location is different that what the AWS SDK expects.

> [!IMPORTANT]
> The `-v ${{ github.workspace }}:/github/workspace/` is necessary to mount the workspace into the docker container. The `AWS_CONFIG_FILE` and `AWS_SHARED_CREDENTIALS_FILE` are set to the same path as the mounted workspace.

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
        id: checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials role test
        id: configure-aws-credentials-role-test
        uses: Moulick/configure-multiple-aws-roles@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/role-test
          role-session-name: GithubActions-${{ env.REPO_NAME }}-${{ github.workflow }}-${{ github.run_id }}
          aws-region: ${{ env.AWS_DEFAULT_REGION }}
          profile: ${{ env.PROFILE_TEST }}
          only-profile: true
          whoami: true

      - run: aws s3 ls --profile ${{ env.PROFILE_TEST }}

      # Example of using the profile with terragrunt
      - name: Terragrunt plan
        id: terragrunt-plan
        uses: gruntwork-io/terragrunt-action@v1
        env:
          AWS_CONFIG_FILE: /github/workspace/.aws/config # And this is the place the AWS config file ends up mounted
          AWS_SHARED_CREDENTIALS_FILE: /github/workspace/.aws/credentials
          AWS_PROFILE: ${{ env.PROFILE_TEST }}

      # Example with generic Docker image
      # Notice the workspace mount and both the env vars, that is the place the AWS config file ends up mounted
      - name: use in docker
        id: use-in-docker
        uses: addnab/docker-run-action@v3
        with:
          image: amazon/aws-cli
          options: -v ${{ github.workspace }}:/github/workspace/ -e AWS_CONFIG_FILE=/github/workspace/.aws/config -e AWS_SHARED_CREDENTIALS_FILE=/github/workspace/.aws/credentials
          run: |
            if ! [ -f ${AWS_CONFIG_FILE} ]; then echo "${AWS_CONFIG_FILE} does not exist." fi
            if ! [ -f  ${AWS_SHARED_CREDENTIALS_FILE} ]; then echo " ${AWS_SHARED_CREDENTIALS_FILE} does not exist." fi

            aws sts get-caller-identity --profile ${{ env.PROFILE_TEST }}
```
