---
title: Simple Pipeline Configuration
layout: post
description:
image: ../assets/images/env-strategy-simple-pipeline.svg
nav-menu: false
show_tile: false
---
With the base configuration we can already start using a simple pipeline like this:
![Simple Pipeline](../assets/images/env-strategy-simple-pipeline.svg)

In this pipeline we'll implement our changes in the **Implementation** org, maybe more than one developer or admin will work on the same org. Then Will use git and Travis-CI to move the changes to Testing (which will act as a merge and testing environment for us), and once we've tested the changes, we'll promote the changes to production.

This is how we can start using this simple pipeline:
1. When we have finished a user story we will create a `feature` branch from `release` branch.
2. We will retrieve the changes we've made to that `feature` branch and push the changes.
3. We will create a pull request from the `feature` branch to `release`, this will trigger Travis-CI to:
   - **Validate** the changes against the org we've configured in MERGE_AUTH_URL environment variable in Travis-CI Settings.
   - Run **static code validation** on the changes: PMD.
   - Run **prettier** to check the changes follow our style.
4. Once the PR is reviewed and all the validations pass, we'll `merge the PR` and this will trigger Travis-CI to promote the changes to the org in MERGE_AUTH_URL, where we can run other types of tests like SIT, UAT or end-to-end tests.
5. We will repeat these steps 1 to 4 for all the user stories we want to promote to production.
6. When we have validated the functionality, we'll create a pull request between `release` and `main`, and with that Travis-CI will validate agains production, which is configured in PROD_AUTH_URL Travis-CI environment variable.
7. If the validation passes and after the PR is reviewed, we can merge it. This won't trigger anything in Travis-CI.
8. When we are ready to move the changes to production we will create a production version tag in `main` branch: i.e. `v1.2.0`. This will trigger Travis-Ci to:
   - **Create a GitHub Release** with a zip file that includes only the changes we are promoting to production.
   - **Promote** the changes to production.
9. Last is to `change the PROD_VERSION` environment variable in Travis-CI to the production tag we've used, i.e. `v1.2.0`. So for the next release Travis-CI will create the delta package by calculating the differences from that version.

With this we have moved a set of changes from the implementation org to our testing environment and finally to production.

We can disable the static code validation by adding a Travis-CI environmnent variable named `DISABLE_PMD` with value `true`. We can also disable prettier by adding another Travis-CI environment variable named `DISABLE_PRETTIER` with value `true`.

**[<< Back to Pipeline Examples](../03-travis-ci.html#pipeline-examples-implementation)**