# `create-aws-codedeploy-deployment` GitHub Action

This action creates [AWS CodeDeploy](https://aws.amazon.com/codedeploy/) deployments from your GitHub Actions workflow. Deployment Group and Deployment configuration itself are derived from an additional configuration section in `.appspec.yml`.

_Note:_ This README assumes you are familiar with the [basic AWS CodeDeploy concepts](https://docs.aws.amazon.com/codedeploy/latest/userguide/primary-components.html).
 
## Design Goals

While this Action tries to mostly get out of our way, it makes a few basic assumptions:

* For your GitHub repository, there is a corresponding CodeDeploy Application already set up.
* Git branches (and so, GitHub Pull Requests) will be mapped to CodeDeploy Deployment Groups. The action will create these, or update existing ones.
* Ultimately, a CodeDeploy Deployment is created with a [reference to the current commit in your GitHub repository](https://docs.aws.amazon.com/codedeploy/latest/userguide/integrations-partners-github.html).

The necessary configuration will be parsed from an additional `branch_config` key inside the `appspec.yml` file – which is the core config file for AWS CodeDeploy that you will need to keep in your repository anyway.

This makes it possible to create a matching configuration once, and then run deployments in different environments automatically. For example, updating a production system for commits or merges on `master`, and independent staging environments for every open Pull Request branch.

## Example Use Case

Please consider the following example Actions workflow that illustrates how this action can be used.

```yaml
# .github/workflows/deployment.yml

on:
    push:
        branches:
            - master
    pull_request:

jobs:
    deploy:
        runs-on: ubuntu-latest
        steps:
            -   uses: aws-actions/configure-aws-credentials@v1
                with:
                    aws-access-key-id: ${{ secrets.ACCESS_KEY_ID }}
                    aws-secret-access-key: ${{ secrets.SECRET_ACCESS_KEY }}
                    aws-region: eu-central-1
            -   uses: actions/checkout@v2
            -   id: deploy
                uses: webfactory/create-aws-codedeploy-deployment@v0.1.0
            -   uses: peter-evans/commit-comment@v1
                with:
                    token: ${{ secrets.GITHUB_TOKEN }}
                    body: |
                        @${{ github.actor }} this was deployed as [${{ steps.deploy.outputs.deploymentId }}](https://console.aws.amazon.com/codesuite/codedeploy/deployments/${{ steps.deploy.outputs.deploymentId }}?region=eu-central-1) to group `${{ steps.deploy.outputs.deploymentGroupName }}`.
```

First, this configures AWS Credentials in the GitHub Action runner. The [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials) action is used for that, and credentials are kept in [GitHub Actions Secrets](https://help.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets).

Second, the current repository is checked out because we at least need to access the `appspec.yml` file.

Third, this action is run. It does not need any additional configuration in the workflow file, but we'll look at the `appspec.yml` file in a second.
   
Last, another action is used to show how output generated by this action can be used. In this example, it will leave a GitHub comment on the current commit, @notifying the commit author that a deployment was made, and point to the AWS Management Console where details for the deployment can be found.

Due to the first few lines in this example, the action will be run for commits pushed to the `master` branch and for Pull Requests
being opened or pushed to. With that in mind, let's look at the [`appspec.yml` configuration file](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file.html).

```yaml
# .appspec.yml

# ... here be your existing CodeDeploy configuration.

# This section controls the action:
branch_config:
    wip\/.*: ~

    master:
        deploymentGroupName: production
        deploymentGroupConfig:
            serviceRoleArn: arn:aws:iam::1234567890:role/CodeDeployProductionRole
            ec2TagFilters:
                - { Type: KEY_AND_VALUE, Key: node_class, Value: production }
        deploymentConfig:
            autoRollbackConfiguration:
                enabled: true

    '.*':
        deploymentGroupName: $BRANCH.staging.acme.tld
        deploymentGroupConfig:
            serviceRoleArn: arn:aws:iam::1234567890:role/CodeDeployStagingRole
            ec2TagFilters:
                - { Type: KEY_AND_VALUE, Key: hostname, Value: phobos.stage }
        deploymentConfig:
            autoRollbackConfiguration:
                enabled: false
```
  
The purpose of the `branch_config` section is to tell the action how to configure CodeDeploy Deployment Groups and Deployments, based on the
branch name the action is run on.

The subkeys are evaluated as regular expressions in the order listed, and the first matching one is used.

The first entry makes the action skip the deployment (do nothing at all) when the current branch is called something like `wip/add-feature-x`. You can use this, for example, if you have a convention for branches that are not ready for deployment yet, or if branches are created automatically by other tooling and you don't want to deploy them automatically.

Commits on the `master` branch are to be deployed in a Deployment Group called `production`. All other commits will create or update a Deployment Group named `$BRANCH.staging.acme.tld`, where `$BRANCH` will be replaced with a DNS-safe name derived from the current branch. Basically, a branch called `feat/123/new_gimmick` will use `feat-123-new-gimmick` for `$BRANCH`. Since the Deployment Group Name is available in the `$DEPLOYMENT_GROUP_NAME` environment variable inside your CodeDeploy Lifecycle Scripts, you can use that to create "staging" environments with a single, generic configuration statement.

The `deploymentGroupConfig` and `deploymentConfig` keys in each of the two cases contain configuration that is passed as-is to the 
[`CreateDeploymentGroup`](https://docs.aws.amazon.com/codedeploy/latest/APIReference/API_CreateDeploymentGroup.html) or 
[`UpdateDeploymentGroup`](https://docs.aws.amazon.com/codedeploy/latest/APIReference/API_UpdateDeploymentGroup.html) API calls (for
`deploymentGroupConfig`), and to [`CreateDeployment`](https://docs.aws.amazon.com/codedeploy/latest/APIReference/API_CreateDeployment.html) for
`deploymentConfig`. That way, you should be able to configure about every CodeDeploy setting. Note that the `ec2TagFilters` will be used to control
to which EC2 instances (in the case of instance-based deployments) the deployment will be directed.

The only addition made will be that the `revision` parameter for `CreateDeployment` will be set to point to the current commit (the one the action is running for) in the current repository.

## Usage

0. The basic CodeDeploy setup, including the creation of Service Roles, IAM credentials with sufficient permissions and installation of the
 CodeDeploy Agent on your target hosts is outside the scope of this action. Follow [the documentation](https://docs.aws.amazon.com/codedeploy/latest/userguide/getting-started-codedeploy.html).
1. [Create a CodeDeploy Application](https://docs.aws.amazon.com/codedeploy/latest/userguide/applications-create.html) that corresponds to your
 repository. By default, this action will assume your application is named by the "short" repository name (so, `myapp` for a `myorg/myapp` GitHub
  repository), but you can also pass the application name as an input to the action.
2. Connect your CodeDeploy Application with your repository following [these instructions](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployments-create-cli-github.html).
3. Configure the [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials) action in your workflow and
 provide the necessary IAM credentials as secrets. 
4. Add the `branch_config` section to your `appspec.yml` file to map branches to Deployment Groups and their configuration. In the above example, the  
`master` and `.*` sub-sections show the minimal configuration required.
5. Add `uses: webfactory/create-aws-codedeploy-deployment@v0.1.0` as a step to your workflow file. If you want to use the action's outputs, you
 will also need to provide an `id` for the step.
 
## Action Input and Output Parameters

### Input

* `application`: The name of the CodeDeploy Application to work with. Defaults to the "short" repo name.

### Outputs

* `deploymentId`: AWS CodeDeployment Deployment-ID of the deployment created
* `deploymentGroupName`: AWS CodeDeployment Deployment Group name used
* `deploymentGroupCreated`: True, if a new deployment group was created. False, if an existing group was updated.

## Hacking

As a note to my future self, in order to work on this repo:

* Clone it
* Run `npm install` to fetch dependencies
* _hack hack hack_
* Run `npm run build` to update `dist/*`, which holds the files actually run
* Read https://help.github.com/en/articles/creating-a-javascript-action if unsure.
* Maybe update the README example when publishing a new version.

## Credits, Copyright and License

This action was written by webfactory GmbH, Bonn, Germany. We're a software development
agency with a focus on PHP (mostly [Symfony](http://github.com/symfony/symfony)). We're big
fans of automation, DevOps, CI and CD, and we're happily using the AWS platform for more than 10 years now.

If you're a developer looking for new challenges, we'd like to hear from you! 

- <https://www.webfactory.de>
- <https://twitter.com/webfactory>

Copyright 2020 webfactory GmbH, Bonn. Code released under [the MIT license](LICENSE).
