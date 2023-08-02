---
layout: post
title: Salesforce DevOps Strategy
description: Promoting Changes Safely and Confidently from Implementation to Production
image: assets/images/dev-ops-strategy.svg
nav-menu: true
---

Our proposal for a Salesforce DevOps Strategy involves two different aspects that work together to move metadata changes from the implentation stage to production easily and consistently: The **Environment Strategy** and the **CI/CD Strategy**.

The **Environment Strategy** defines what kind of environments we need in a healthy deployment pipeline and the **CI/CD Strategy** specifies how the changes will flow through the pipeline from one environment to the next one.

By defining what kind of environments we need and how we move changes from an environment to the next we create deployment pipelines that allow us to bring changes to production in safe, consistent and auditable way.

## Environment Strategy
The environment strategy talks about the types of environments we should/can have in a salesforce deployment pipeline according to what they’re used for. Each one of the orgs in a pipeline will fit one or more of the following roles:

+ **Implementation**: Changes are implemented in these environments, be them configuration or development changes.
+ **Merge**: Changes coming from different implementation environments on the same project are merged in these environments. We need to have at least one merge environment when we have more than one implementation environment in the pipeline.
+ **Testing**: We use these environments to run different types of validations. I.e. Deployability, Integration, QA, UAT or DR validations.
+ **Training**: We use these environments to have the users trained in the new functionalities the projects are implementing (or have implemented already).
+ **Pre-Production**: We use this environment to package functionalities approved during tests and run load, regression or performance tests. This environment makes sense when we deploy features when they’re ready and not when the whole project is implemented and tested.
+ **Production**: Real time operation happens in this environment.

When we configure our pipelines we need to have at least one of the following roles: **Implementation**, **merge** (whenever we have more than one implementation environment), **testing** and **production**. But we’ll need more of them depending on the needs of each company.

## CI/CD Strategy
This aspect of the overall Salesforce DevOps strategy defines when and how we move changes through the pipeline.

In this strategy we will use git repositories as the source of truth for the metadata we have in production and flowing through the pipeline.
And it uses our actions in git to automatically trigger jobs (not necessarily using GitHub Actions, eventhough is one of the posibilities) that will validate our changes and promote them through the pipeline.

### Git Branching
The same way we defined the environment roles in the pipeline, we'll now talk about git branches and their roles.
These git branches will help us keep track of the changes we're implementing, at what stage these changes are and defines the current state of production and other shared environments.

These are the branches we need to use in this strategy:
+ **main / master**: this is the branch that is the source of truth for the metadata in production.
+ **release**: this is the branch that bundles all the changes we'll promote to production (and maybe some testing environment) in the next release.
+ **project release braches**: if we are implementing a complex pipeline with several projects, we will use these branches to bundle the changes for that project only before moving them. The naming convention for these branches is to use the project id as the prefix for `-release`, i.e. if we have a project with the id `project1` we'll create a project release branch named: `project1-release`.
+ **environment branches**: these branches are the source of truth for the metadata in specific environments, i.e. SIT, DryRun, etc... It's useful when we want to test changes in a shared test environment before we bundle them in a release. These branches follows the following naming convention: environment id + `-environment`, i.e. we have a sandbox we're using for integration tests we call `sit` in which we don't want to bundle the changes yet, for that we'll create a branch called `sit-environment` as the source of truth for that environment.
+ **feature branches**: we will create these branches from the `release` or `project release` branches when we need to bring changes to the pipeline and we will merge them back to them when we finish that implementation (via a PR). The name doesn't need to follow any naming convention, but I'd suggest to use `feature/` + feature id. We just need to avoid naming them as an environment or a project release branch.

### Automating with Git
The pipelines we can build using this Salesforce DevOps Strategy automatically create packages with only changes, validate them and promote them when we bring them to the git repository.

The pipeline **automates validations** when we create pull requests and it **automates promotions** when we tag commits or merge pull requests.

The automation will find out where to validate or promote based on the destination branch of the pull request or the name of the tag we are creating, as follows:
+ **Creating a PR to a project release branch**: The pipeline will find out the blundle environment for that project and will validate the changes there. We can configure the pipeline to validate using a pool of environments instead.
+ **Merging a PR to a project release branch**: The pipeline will find out the bundle environment for the project and will promote the changes in that branch.
+ **Creating a PR to an environment branch**: The pipeline will validate the changes against the environment backed by that environment branch.
+ **Merging a PR to an environment branch**: The pipeline will promote the changes on the environment backed by that branch.
+ **Creating a PR to `release` branch**: The pipeline validates on the environment where we bundle the changes for production (we call it MERGE). Here we can configure the pipeline to validate against a pool of environments too.
+ **Merging a PR to `release` branch**: The pipeline promotes the changes to the bundle environment.
+ **Tagging a commit with a non-production tag (in `release`)**: A non-production tag is a git version tag prefixed with an environment id, i.e. sit-v1.2.3. When we create these tags in `release` branch, the pipeline will find out the environment we want to promote to and it will promote the changes there. In the example before, the non-production version tag `sit-v1.2.3` would trigger the pipeline to promote the change to the environment configured as SIT in the pipeline.
+ **Creating a PR to main or master**: The pipeline will validate the changes against production.
+ **Tagging a commit with a production tag (in `main`/`master`)**: A production version tag is a semver style git tag, i.e. `v1.2.3`. When we tag a commit in `main` or `master` with these kind of tags, it will promote the changes to production.
+ **Tagging a commit with a backpromote tag**: Similarly with the production and non-production tags, we can create a backpromote tag for an environment and the pipeline will look for that environment in its configuration, and back-promote the changes there. Backpromote tag naming convention start with `backpromote-`, then the environment id, and then we can add a `-` followed by anything to make it unique, i.e. the current date. An example is `backpromote-dev1-20230524`, this would trigger the pipeline to look for a configurated environment called `dev1` and then back-promote the changes there.
