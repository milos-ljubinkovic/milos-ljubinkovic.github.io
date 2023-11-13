---
title: "Self-Service testing environments"
date: 2023-11-01T15:34:30-04:00
categories:
  - blog
tags:
  - Mongo Atlas
  - AWS
  - Compliance
---

# Self-Service testing environments

## Purpose

Originally we started with two environments production and development, first built from master and second from our develop branches
Developers added code as it was finished to the develop environment qa personnel did their tests there and after passing tests and getting approval from the manager the changes were merged to the master branch

This caused issues as things sometimes got out of sync as one environment contained both code that the developers uploaded for their own client-server integration, code that was in testing and code that was out of testing and waiting for approval to be deployed.

We quickly added another staging environment and branch for code that was ready for final testing and all deployment was done from this branch after final testing passed

Eventually 2 issues appeared. One, sometimes out of the multiple changes on the staging environment only some were approved by the manager. This slowed down development as the environment had to be tested all over again after taking out the unapproved changes.
Second, as the team increased having multiple devs working on a single develop environment sometimes caused issues and downtime due to conflicts. 

Eventually we decided to start working with multiple short-lived environments which would be forked from master on demand. Every larger issue in a sprint would get their own environment created and smaller ones like bugfixes could be bundled up in a single one. After developers finished, the unique environment will get into the hands of the QA department and after it passed and final approvals were given the deployment would be done directly from the unique environment to master.

We wanted to give the ability to the developers to create and destroy environments as needed to make the whole procedure are agile as possible as I feared that any unnecessary procedure or bureaucracy would cause them to revert to the old system of using a single environment for everything.

One option was to create a brand new dashboard for this or go with a custom one. Luckily the combination of gitlab and cloudformation allowed us to use the already existing CICD pipelines with some smaller changes.



## Cloudformation

## Gitlab
