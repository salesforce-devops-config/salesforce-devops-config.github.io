---
layout: post
title: Travis-CI
description: Travis-CI DevOps Configuration for Salesforce Pipelines
image: assets/images/travis-ci.svg
nav-menu: true
---
If you are evaluating tools to start implementing Salesforce DevOps, Travis-CI should be one of the tools you should look into.

We can use this [Travis-CI configuration](https://github.com/salesforce-devops-config/travis-ci){:target="_blank"} and GitHub to implement [pipelines](02-pipelines.html){:target="_blank"} using [this Salesforce DevOps Strategy](01-devops-strategy.html){:target="_blank"}.

## Preparing Your Repository with this Travis-CI Configuration
First you need to clone the [Travis-CI configuration repository](https://github.com/salesforce-devops-config/travis-ci){:target="_blank"} in a separate directory (not in your salesforce repository).

Then you need to copy the content of `config` directory of the Travis-CI repo inside the `config` directory of your repository, if you don't have a `config` directory in your repository, then just copy the whole `config` directory to the root of your directory (the root directory of your project is the directory that contains the sfdx-project.json file).

After that, if there's no `script` directory in your project, copy the whole `script` directory from the Travis-CI repo to the root of your repository, if you already have this `script` directory, copy the content of the `script` directory from the Travis-CI repository inside the `script` directory of your project.

Then copy the directories `rulesets` and `tests` and the files `.travis.yml`, `.pretierrc.yaml`, `package.json` and `sfdx-project.string-replacement.example.json` to the root of your project.

After doing this, go to your Travis-CI dashboard and find your salesforce's project repository, and in settings make sure to disable building for branches and pull requests, as we don't want Travis being triggered yet.

Once Travis-CI is disabled for your salesforce repository you can stage, commit and push the new Travis-CI configuration and supporting scripts to your repository.

Next step is to start implementing the pipeline, which you will do by creating branches and tags in git and adding environment variables in Travis-CI according to the needs of your pipeline.

When the pipeline is implemented, you can go back to your Travis-CI project's settings and re-enable building for branches and pull requests.

## Implementing Pipelines
We will implement our pipeline with Travis-CI and [this configuration](https://github.com/salesforce-devops-config/travis-ci){:target="_blank"} accordingly with our needs.

### Base Configuration
However there is some basic configuration points we need no matter what is the shape of the pipeline we will end up implementing:
+ We will need to **tag the baseline** of production in `main` or `master` branch. After we have added this Travis-CI config to our Salesforce repository we will pull the current state of Production to `main`, and then we will create and push a git tag to identify the base that the pipeline will use to calculate the changes. I.e. we can call it `prod-baseline` or `v0.0.0`.

+ We will need to **create a `release` branch** in our Salesforce repository. We will use this branch in which we will bundle all the changes that we will promote through the pipeline to Prod.

+ Create a **GitHub Personal Token** in github with `repo` and `read:org` permissions. This pipeline will use it to read the PR body and to create GitHub Release artifacts.

+ We will need to create the following **Travis-CI environment variables**:
  - **GH_TOKEN**: with the GitHub Personal Token we just created as value.
  - **PROD_VERSION**: with the baseline tag we just created (`prod-baseline` or `v0.0.0` or any other tag you've created) as value.
  - **MERGE_AUTH_URL**: With the SFDX Auth URL of the environment we will use to validate and promote the bundled changes when we create and merge a pull request to `release` branch as value. We can get this SFDX Auth URL by authorizing the salesforce org with `sf` (or `sfdx`) CLI and then getting its information in verbose mode: `sf org display --verbose -o <username or alias>` under the key `Sfdx Auth Url`.
  - **PROD_AUTH_URL**: With the SFDX Auth URL of our Prod environment.

Only with this mandatory configuration we already have implemented a simple pipeline in which we can safely move changes from one or more implementation environments to a bundle/testing environment and then to production.

### Other Configuration
These are the actions we can automate with this travis configuration out-of-the-box just by creating Travis-CI environment variables and git branches:
+ **Validate changes** when we create a pull request to release, project release, environment or main branches:
  + The pipeline by default will validate against the same environment to which it will promote these changes when the PR is merged.
  + But, it can be configured to validate against a pool of sandboxes to avoid having a bottle neck when there are several PRs trying to validate changes on the same sandbox.

+ **Define test levels** for each feature on change validation, including specific tests.

+ **Promote changes** when we merge a pull request or create a version tag (production or non-production version tag).
  + By default, the pipeline will run the default test level of the type of org.
  + But the pipeline can be configured to force running the same tests it run when it validated the changes on that environment.

+ **Validate and Promote un-bundled features** to the environments before the bundling stage by creating and merging pull requests to environment branches.

+ **Back-promote changes** by creating backpromote tags (or via pull requests).

## Pipeline Examples Implementation
Let's see how we can configure pipelines with different complexity levels.

I'm going to assume we already have a github repository with an SFDX project set up as explaind in the [Preparing Your Repository with this Travis-CI Configuration section](#preparing-your-repository-with-this-travis-ci-configuration) and configured as explained in the [Base Configuration section](#base-configuration).

### Simple Pipeline
First let's start with a very simple pipeline that hast an implementation environment, a merge/test environment and production, like this one:

![Simple Pipeline](assets/images/env-strategy-simple-pipeline.svg)

**[Learn More >>](travis-ci/simple-pipeline.html)**

### Simple Project
With small changes to the base Travis-CI configuration we can set up a slightly more complex configuration suited for small projects or in-house business as usual operations, like this one:

![Simple Project Pipeline](assets/images/env-strategy-simple-project.svg)

**[Learn More >>](travis-ci/simple-project.html)**

### Complex Pipeline
Combining the simple project configuration with a bit of extra git repository setup and a few more environment variables, we can create a much more complex pipeline, like the following one:
![Complex Pipeline](assets/images/env-strategy-complex-pipeline.svg)

**[Learn More >>](travis-ci/complex-project.html)**

## More Pipeline Configuration Points

### Allow Pull Requests to `main`
By default, this Travis configuration will only allow to create Pull Requests to `main` from either `release` or `cs-enhancements`, but we allow other branchs by adding the Travis-CI environment variable **TRAVIS_PULL_REQUEST_BRANCH**.

In **TRAVIS_PULL_REQUEST_BRANCH** we will add a comma separated list of branches that will be allowed to create a pull requests directly to `main`.

### Change the Baseline Tag for a Stream of Work
By default, this Travis-CI configuration creates the delta packages it validates and promotes using the tag configured the environemnt variable **PROD_VERSION**.

But we can change that, if we **create a baseline variable** for the environment we're validating on or promoting to. A baseline variable for an environment starts with the ID of the environment in Travis-CI config followed by `_BASELINE`.

As an example, when we validate or promote to `P1_AUTH_URL` and we want it to use a previous production version because the project is not ready to back-promote the changes. In this case the ID is `P1` and we can configure that version in `P1_BASELINE` and Travis-CI will use it to build the delta package.

### Increasing Travis-CI Wait for Timeouts
By default Travis-CI will use 30 minutes before it times out (using this configuration), but we can configure it with an environment variable.

We can set **TRAVIS_WAIT** environment variable with the number of minutes we need travis to wait. This will be applied to all the validations and promotions.

### Use Different Environments to Validate and Promote in a Pull Request
By default travis will use one of the `_AUTH_URL` variables to validate against or promote to. However there are some scenarios in which we may want to separate the validations and the promotions, and we can do dhat by adding `_AUTH_URL_CI` variables.

We can either choose to have a single CI environment to validate against or a pool of CI environments for each environment configured as `_AUTH_URL` variable.

For example, we have an environment in which we validate and promote when we create a pull request to `p1-release`, it is configured in the environment variable `P1_AUTH_URL`.
But we have a lot of validations happening from different PRs to `p1-release` that get blocked constantly because there's another validation going on, and we want to have Travis-CI validating in a pool of 4 CI sandboxes.

We will need to create 4 new variables: `P1_AUTH_URL_CI_1`, `P1_AUTH_URL_CI_2`, `P1_AUTH_URL_CI_3` and `P1_AUTH_URL_CI_4` anbd set the SFDX Auth URL of each one of the CI environments in them.

If we want just one CI environment, then we can just create teh environment variable `P1_AUTH_URL_CI` and set the SFDX Auth URL of that environment there.

### Default Test Level When Promoting
By default Travis will promote using the default test level for the org we are promoting to, that is it won’t run tests on promotion for sandboxes and it will run local tests for production orgs.

We can change it by adding a variable with the stream of work id  or the id of the id of the org Travis will deploy to, post-fixed with `_DEPLOY_DEFAULT_TEST_LEVEL` and give it the value of false. This way Travis will use the same test levels that we configure for the validations in the [next section](#how-to-configure-tests-for-each-feature).

#### Some examples:
We have a Stream of Work called with ID `P1`, which has a merge org configured with `P1_AUTH_URL` in Travis. If we want to run the same level of tests that we run when we validate against P1_AUTH_URL, we would add a variable called **P1_DEPLOY_DEFAULT_TEST_LEVEL** and give it false as value.

If we have a training org (configured in Travis as `TRAINING_AUTH_URL`) to which we promote changes using a training version tag (i.e. training-v1.2.0) and we want the promotion to run the same test level used to validate, we need to create a variable **TRAINING_DEPLOY_DEFAULT_TEST_LEVEL** with value false.

### Configuring Travis to Run Validations with Specific Test Levels
The sooner we run tests, the earlier we’ll find issues. We don’t want to promote an already tested version and find out it’s breaking tests.

Our out-of-the-box Travis configuration allow us to decide what test level and what tests to run for each feature by creating a `.json` file in the `tests` directory.

The content of this file is a JSON object with the following fields:
- **testLevel**: Corresponding to the salesforce test levels we can use with sfdx. Allowed Values:
  - RunAllTestsInOrg
  - RunLocalTests
  - RunSpecifiedTests
  - NoTestRun
- **test**: This is only mandatory when we use testLevel `RunSpecifiedTests` and it is a list of tests classes we want to run, in the form of a JSON array of strings.

#### How to Configure Tests for each Feature
For each feature we want to change change the test level we will create a .json file in `tests` directory with a unique name in the feature branch, i.e. `tests/feature-xxxx.json` for the Feature XXXX.

The name we choose is irrelevant for Travis, but it needs to be a .json file. However it’s beneficial for the team if the name is meaningful, so if possible, use the feature id, that helps identify what feature required what test level.

It is important to keep the .json file name unique, because when the feature branch is merged to the release branch we will avoid merge conflicts.

When we validate or promote from a release branch, with bundled features that included different test levels, Travis will consolidate the test levels like this:

1. If there’s any feature that needs RunAllTestsInOrg, this is the test level used.
2. If there’s no RunAllTestsInOrg, but there’s at least one RunLocalTests, Travis will use this test level for the bundle.
3. If there are no RunAllTestsInOrg or RunLocalTests, but there are RunSpecifiedTests, Travis will use this test level, and will run all the tests specified in the json files.
4. Finally, if there is no test levels, or we only have NoTestRun, Travis will use the default of the org we’re validating or promoting.

#### Running All Tests In Org for a Feature
If we need to run all tests for a feature, we will create a .json file in `tests` directory with a unique name, for example the feature id, i.e. `tests/feature-0001.json`, specifying the testLevel `RunAllTestsInOrg`.

Example:
```json
{
  "testLevel": "RunAllTestsInOrg"
}
```

#### Running Local Test for a Feature
When we need to run local tests for a feature, we will create a .json file in `tests` directory with a unique name, i.e.: `tests/feature-0002.json`, specifying the testLevel `RunLocalTests`.

Example:
```json
{
  "testLevel": "RunLocalTests"
}
```

#### Running Specific Tests for a Feature
More often we’ll want to specify what tests we want to run for our changes, to do it we will create a .json file in `tests` directory with a unique name, like `tests/feature-0003.json`, specifying the testLevel `RunSpecifiedTests` and adding the list of tests classes we want to run in the `tests` field.

Example:
```json
{
  "testLevel": "RunSpecifiedTests",
  "tests": [
    "Test01", "Test02", "Test03",
    "Test04", "Test05", "Test06"
  ]
}
```

#### No Test Run Validations or Promotions
In some cases we may not need to run tests for some feature, in that case we’ll create a .json file in `tests` directory with a unique name, i.e. `tests/feature-0004.json` specifying the testLevel `NoTestRun`.

Example:
```json
{
  "testLevel": "NoTestRun"
}
 ```