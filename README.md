# aws-deploy-and-init-action

This action must be used when you want to deploy a fresh new set of stacks for all the resources. It will fails if stacks already exist (even if is only one of them) so be sure to delete all of them if you want to launch this action with success.

## Example workflows file
```yaml
name: "deploy_and_init_reach_stacks"
on:
  workflow_dispatch:
    branches:
    - awsstaging
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1
    - name: Extract branch name
      shell: bash
      run: echo "::set-output name=branch::$(echo ${GITHUB_REF#refs/heads/})"
      id: extract_branch
    - uses: mirkesx/aws-sam-deploy-emblematic-action@main
      env:
        AWS_FOLDER: 'aws_cf_stack'
        AWS_STACK_PREFIX: ${{ steps.extract_branch.output.branch }}
        AWS_REGION_VALUE: 'eu-north-1'
        AWS_ACCESS_KEY_ID_VALUE: ${{ secrets.AWS_ACCESS_KEY_ID_S3USER }}
        AWS_SECRET_ACCESS_KEY_VALUE: ${{ secrets.AWS_SECRET_ACCESS_KEY_S3USER }}
        AWS_DEPLOY_BUCKET: 'robot-reach-test-deploy'
        PGUSER: ${{ secrets.PG_MASTER_USER }}
        PGPASSWORD: ${{ secrets.PG_MASTER_PASSWORD }}
```

## Example break-down
This actions can be triggered only from the Action tab since it is a workflow_dispatch.

The only branch allowed to dispatch this action is aws-staging (there will be more in the future).

### Steps:
- actions/checkout@v1 is used to allow the action to pull the repo
-“Extract branch name” is used to retrieve the name of the branch
- mirkesx/aws-sam-deploy-emblematic-action@main runs AWS CLI/SAM commands to deploy the three stacks (s3, rds, lambda) and initialize the RDS.

### Env variables
- AWS_FOLDER, which is the path to the folder in the repo where you can find the SAM templates
- AWS_STACK_PREFIX, will be the prefix for all the three stacks. By default is the branch name and is retrieved in the second step
- AWS_REGION_VALUE is the region where to deploy the resources
- AWS_ACCESS_KEY_ID_VALUE and AWS_SECRET_ACCESS_KEY_VALUE are credentials of the IAM AWS user used to login on AWS Cli
- AWS_DEPLOY_BUCKET, is the bucket where to upload the packages created by AWS Cli
- PGUSER and PGPASSWORD are the credentials of the Master User defined for the RDS Instance


### Third step summary
- Checks whether the env variables are initialized
- Checks whether there exists even just one of the three stacks (failing if so)
- Deploys the S3 stack and apply the policies
- Deploys the RDS stack. Creates the Master User and a database instance called as the PGUSER
- Retrieves RDS endpoint and port and S3 bucket name then deploys the lambda stack and apply the triggers to S3
- Initializes the RDS instance with the an empty reach dump
- Deletes the packages as a clean up
