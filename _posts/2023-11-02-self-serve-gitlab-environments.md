---
title: "Self-Service testing environment(s)"
date: 2023-11-01T15:34:30-04:00
categories:
  - blog
tags:
  - Mongo Atlas
  - AWS
  - Compliance
---

## What and Why

Originally we started with two environments production and development, first built from master and second from our develop branches
Developers added code as it was finished to the develop environment qa personnel did their tests there and after passing tests and getting approval from the manager the changes were merged to the master branch

This caused issues as things sometimes got out of sync as one environment contained both code that the developers uploaded for their own client-server integration, code that was in testing and code that was out of testing and waiting for approval to be deployed.

We quickly added another staging environment and branch for code that was ready for final testing and all deployment was done from this branch after final testing passed

Eventually 2 issues appeared. One, sometimes out of the multiple changes on the staging environment only some were approved by the manager. This slowed down development as the environment had to be tested all over again after taking out the unapproved changes.
Second, as the team increased having multiple devs working on a single development environment sometimes caused issues and downtime due to conflicts. 

Eventually we decided to start working with multiple short-lived environments which would be forked from master on demand. Every larger issue in a sprint would get their own environment created and smaller ones like bugfixes could be bundled up in a single one. After developers finished, the unique environment will get into the hands of the QA department and after it passed and final approvals were given the deployment would be done directly from the unique environment to master.

We wanted to give the ability to the developers to create and destroy environments as needed to make the whole procedure are agile as possible as I feared that any unnecessary procedure or bureaucracy would cause them to revert to the old system of using a single environment for everything.

One option was to create a brand new dashboard for this or go with a custom one. Luckily the combination of gitlab and cloudformation allowed us to use the already existing CI/CD pipelines with some smaller changes.


## Cloudformation

We start with a basi

## Gitlab

The resulting pipeline looks something like this. 

We have:

1. placeholder **prebuild** and
2. **build** steps with separate build logic
3. a manually triggered **cleanup** step which deletes the cloudformation stack, can't be executed on the master branch

3. manually triggered **create-stack** step which creates the stack, it's also prevented from being executed on the master branch

4. regular step **update-cf** which is allowed to fail, and will fail if the stack doesn't exist



![image](/assets/images/pipeline.png)


```yaml
image: node:18-alpine #alpine:latest

workflow:
    rules:
        - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
          when: never
        - if: "$CI_COMMIT_BRANCH"

stages:
    - prebuild
    - build
    - deploy
    - cleanup


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

create-stack:
    stage: deploy
    image: registry.gitlab.com/gitlab-org/cloud-deploy/aws-base:latest
    allow_failure: true
    script:
        - aws cloudformation package --template-file ./cf.yml --s3-bucket $DEPLOY_BUCKET  --output-template-file packaged-sam.yaml
        - aws s3 cp packaged-sam.yaml s3://$DEPLOY_BUCKET/packaged-sam-$CI_COMMIT_REF_SLUG.yaml
        - aws cloudformation create-stack  --stack-name main-stack-$CI_COMMIT_REF_SLUG --template-url https://$DEPLOY_BUCKET.s3.$AWS_DEFAULT_REGION.amazonaws.com/packaged-sam-$CI_COMMIT_REF_SLUG.yaml --capabilities CAPABILITY_NAMED_IAM CAPABILITY_IAM CAPABILITY_AUTO_EXPAND --parameters ParameterKey=BRANCH,ParameterValue=$CI_COMMIT_REF_SLUG ParameterKey=ENV,ParameterValue=$SECRETS_FILE
        - aws cloudformation wait stack-create-complete --stack-name main-stack-$CI_COMMIT_REF_SLUG 
        - aws cloudformation --region $AWS_DEFAULT_REGION describe-stacks --stack-name main-stack-$CI_COMMIT_REF_SLUG > stack.json
    artifacts:
      untracked: false
      when: on_success
      expire_in: "30 days"
      paths:
        - stack.json
    environment:
        name: $CI_COMMIT_REF_SLUG
        url: https://api-$CI_COMMIT_REF_SLUG.$DOMAIN/
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
        - aws cloudformation get-template-summary --stack-name main-stack-$CI_COMMIT_REF_SLUG > /dev/null
        - aws cloudformation package --template-file ./cloudformation.yml --s3-bucket $DEPLOY_BUCKET  --output-template-file packaged-sam.yaml
        - aws s3 cp packaged-sam.yaml s3://$DEPLOY_BUCKET/packaged-sam-$CI_COMMIT_REF_SLUG.yaml
        - aws cloudformation update-stack --template-url https://$DEPLOY_BUCKET.s3.$AWS_DEFAULT_REGION.amazonaws.com/packaged-sam-$CI_COMMIT_REF_SLUG.yaml --stack-name main-stack-$CI_COMMIT_REF_SLUG --capabilities CAPABILITY_NAMED_IAM CAPABILITY_IAM CAPABILITY_AUTO_EXPAND --parameters ParameterKey=BRANCH,UsePreviousValue=true ParameterKey=ENV,UsePreviousValue=true
        - aws cloudformation wait stack-update-complete --stack-name main-stack-$CI_COMMIT_REF_SLUG 
        - aws cloudformation --region $AWS_DEFAULT_REGION describe-stacks --stack-name main-stack-$CI_COMMIT_REF_SLUG > stack.json
    environment:
        name: $CI_COMMIT_REF_SLUG
        url: https://api-$CI_COMMIT_REF_SLUG.$DOMAIN/

    rules:
        - if: '$CI_COMMIT_BRANCH !=  "master"'
        - if: '$CI_COMMIT_BRANCH == "master"'
          when: manual

delete-stack:
    stage: cleanup
    image: registry.gitlab.com/gitlab-org/cloud-deploy/aws-base:latest
    needs: []
    script:
        - aws s3 rm s3://helbiz-skyview-${CI_COMMIT_REF_SLUG} --recursive
        - aws s3api delete-bucket --bucket "helbiz-skyview-${CI_COMMIT_REF_SLUG}"
        - aws cloudformation delete-stack --stack-name main-stack-$CI_COMMIT_REF_SLUG
        - aws cloudformation wait stack-delete-complete --stack-name main-stack-$CI_COMMIT_REF_SLUG 
    rules:
        - if: '$CI_COMMIT_BRANCH != "master"'
          when: manual
        - if: '$CI_COMMIT_BRANCH == "master"'
          when: never

```