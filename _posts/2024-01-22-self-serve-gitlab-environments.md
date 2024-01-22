---
title: "Self-Service testing environment(s)"
date: 2024-01-22T15:34:30-04:00
categories:
  - blog
tags:
  - Gitlab
  - CICD
  - AWS
  - Cloudformation
toc: true
toc_label: "Contents"
toc_icon: "book-open"
---

<style>
    .page__content {
        text-align: justify;
    }
</style>


![image](/assets/images/cart.png)

## What and Why

We originally started with two environments: production and development. Production was built from master and development from our develop branches.
Developers added code as it was finished to the develop environment, qa personnel did their tests there and after their approval and after getting "business approval" from the manager, the changes were merged to the master branch and automatically deployed.

This worked for a while but as the company grew it caused issues. Things sometimes got out of sync as the development environment contained both the code that the developers uploaded for their own client-server integration, the code that was in testing and the code that was out of testing and waiting for approval to be deployed.

We quickly added another staging environment/branch pair for code that was ready for final testing and all deployment was done from this branch after final testing passed

Eventually 2 issues appeared. One, sometimes out of the multiple changes on the staging environment only some were approved by the manager. This slowed down development as the environment had to be tested all over again after taking out the unapproved changes.
Second, as the team increased having multiple devs working on a single development environment sometimes caused issues and downtime due to conflicts. 

Eventually we decided to start working with multiple short-lived environments which would be forked from master on demand. Every larger issue in a sprint would get their own environment created and smaller ones like bugfixes could be bundled up in a single one. After developers finished, the unique environment will get into the hands of the QA department and after it passed and final approvals were given the deployment would be done directly from the unique environment to master.

We wanted to give the ability to the developers to create and destroy environments as needed to make the whole procedure are agile as possible as I feared that any unnecessary procedure or bureaucracy would cause them to revert to the old system of using a single environment for everything.

One option was to create a brand new dashboard for this or go with a custom one. Luckily the combination of gitlab and cloudformation allowed us to use the already existing CI/CD pipelines with some smaller changes.


## Cloudformation

We start with a basi

## Gitlab

The resulting pipeline looks something like this. 


![image](/assets/images/pipeline.png)



The **"prebuild"** and **"build"** steps get executed for all branches. This will make sure every commit in the repo gets a flag that it can at least build and pass unit tests. 


Then the pipeline continues with the deploy stage and the **"update"** step, which will be skipped if the environment doesn't exist. To create a deployed environment for this branch just simply click on play button next to the **"create-stack"** step.

After the environment is created every following commit to this branch will deploy automatically to it through the **"update"** step.


To delete the stack simply start the manual step **"delete-stack"** and all traces of it will be deleted. Both delete and create steps have been disabled on our default branch which deploys the production environment for obvious reasons.


## CICD Pipeline

### Placeholders and boilerplate

First we define the default docker image used in the pipeline, make sure the pipeline triggers on commits and define the 3 stages. The first two stages are just placeholders to later be filled with actual build and test commands.




```yaml
image: registry.gitlab.com/gitlab-org/cloud-deploy/aws-base:latest

workflow:
    rules:
        - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
          when: never
        - if: "$CI_COMMIT_BRANCH"

stages:
    - prebuild
    - build
    - deploy


prebuild:
    stage: prebuild
    script:
        - echo secrets > .env
    artifacts:
        paths:
            - .env

build:
    stage: build
    script:
        - mkdir dist
        - echo scripts > ./dist/app.js
    artifacts:
        paths:
            - dist/**

```

### Create and Update stacks

We wish to create 2 steps, one manual which will create a new cloudformation stack and one automatic which will check if the stack with that name already exists and update it if it does or exit if it doesn't. 

We can perform the check for the existence of the stack with the `aws cloudformation get-template-summary NAME` command, which will exit with a failure code if a stack with that name doesn't exits and make the step `allow_failure: true`. This isn't optimal as the step will be rendered with a yellow ! in gitlab's UI in this case, possibly causing some confusion. But we will keep it like this for now because gitlab doesn't allow for an easy way for a job to affect the flow of a pipeline.

Now there are 2 different ways we can do this. We can use the `cloudformation deploy` command for both, or use a combination of `cloudformation create-stack` and `cloudformation update-stack`. The `deploy` command is simpler and easier to use but lacks some of the more granular options in the `update/create` CLI commands like rollbacks and stack policies. For now lets go with just deploy, but in a later blog I'll demonstrate how to implement automatic stack rollback on server errors using the `update-stack` command.

We will name our stack: `stack-$CI_COMMIT_REF_SLUG` which will resolve in gitlab runner to the slug of the current branch branch if the trigger for the pipeline is a commit. And we will setup rules so that `create` can't trigger on the master branch and requires a manual step on other branches, and that updating the master branch stack requires a manual confirmation and is automatic on all other branches.


```yaml

create-stack:
    stage: deploy
    image: registry.gitlab.com/gitlab-org/cloud-deploy/aws-base:latest
    allow_failure: true
    script:
        - aws cloudformation package --template-file ./cf.yml --s3-bucket $DEPLOY_BUCKET  --output-template-file packaged-sam.yaml
       - aws cloudformation deploy --template-file packaged-sam.yaml  --stack-name stack-$CI_COMMIT_REF_SLUG --capabilities CAPABILITY_NAMED_IAM CAPABILITY_IAM CAPABILITY_AUTO_EXPAND --parameter-override BRANCH=$CI_COMMIT_REF_SLUG --s3-bucket deploy-bucket-841805187071
    rules:
        - if: '$CI_COMMIT_BRANCH != "master"'
          when: manual
        - if: '$CI_COMMIT_BRANCH == "master"'
          when: never

update-cf:
    stage: deploy
    image: registry.gitlab.com/gitlab-org/cloud-deploy/aws-base:latest
    allow_failure: true
    script:
        - aws cloudformation get-template-summary --stack-name stack-$CI_COMMIT_REF_SLUG > /dev/null
        - aws cloudformation package --template-file ./cf.yml --s3-bucket $DEPLOY_BUCKET  --output-template-file packaged-sam.yaml
        - aws cloudformation deploy --template-file packaged-sam.yaml  --stack-name stack-$CI_COMMIT_REF_SLUG --capabilities CAPABILITY_NAMED_IAM CAPABILITY_IAM CAPABILITY_AUTO_EXPAND --parameter-override BRANCH=$CI_COMMIT_REF_SLUG --s3-bucket $DEPLOY_BUCKET
    rules:
        - if: '$CI_COMMIT_BRANCH !=  "master"'
        - if: '$CI_COMMIT_BRANCH == "master"'
          when: manual
```


### Delete

And in the end we add the commands for deletion of the entire stack. Make sure you disable this step from being executed on you master branch and make it a manual step on all other branches.

There are some resources like S3 buckets that can cause some issues with stack deletion. The files on s3 fall out of the scope of the stack so they don't get deleted and they also prevent the bucket from being deleted causing the whole delete operation to fail. 

There area a couple of solutions to this problem. The cleanest one is to just empty and delete any buckets you created in this step. For example like this:

```sh
aws s3 rm s3://BUCKET --recursive
aws s3api delete-bucket --bucket "BUCKET"
```

Or as a safe solution you can put a flag in the Cloudformation template to retain the bucket even after you delete the stack like this:[1]

```yaml
Resources:
  myS3Bucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
```

```yaml
delete-stack:
    stage: .post
    image: registry.gitlab.com/gitlab-org/cloud-deploy/aws-base:latest
    needs: []
    script:
        - aws cloudformation delete-stack --stack-name stack-$CI_COMMIT_REF_SLUG
        - aws cloudformation wait stack-delete-complete --stack-name stack-$CI_COMMIT_REF_SLUG 
    rules:
        - if: '$CI_COMMIT_BRANCH != "master"'
          when: manual
        - if: '$CI_COMMIT_BRANCH == "master"'
          when: never
```



^[1] AWS Deletion policy https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html