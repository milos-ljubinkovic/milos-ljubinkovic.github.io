---
title: "Adding rollback to a Cloudformation stack"
date: 2023-11-01T15:34:30-04:00
categories:
  - blog
tags:
  - AWS
  - Cloudformation
  - Devops
---

## Changing the Deploy procedure

We first call package to upload all of the referenced local files to the DEPLOY_BUCKET and handle all of the yaml file transformations:

`aws cloudformation package --template-file ./cf.yml --s3-bucket $DEPLOY_BUCKET  --output-template-file packaged-sam.yaml`

Now you can run `aws cloudformation deploy` on this file with some additional parameters

## Rollback alarm

We create a canary synthetic tester with its own internal logic that's used to determine the health of the deployed environment. It needs an s3 bucket and permissions needed to write the logs to it.

```
  CanaryBucket:
    Type: "AWS::S3::Bucket"
    Properties:
      BucketName: !Sub "domain-${Name}-${Branch}-canary"

  CanaryRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          Effect: Allow
          Principal:
            Service: lambda.amazonaws.com
          Action: sts:AssumeRole
      Policies:
        - PolicyName: allowCanary
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
            - Effect: Allow
              Action:
              - s3:PutObject
              - s3:GetObject
              Resource:
              - !GetAtt CanaryBucket.Arn
            - Effect: Allow
              Action:
              - s3:GetBucketLocation
              Resource:
              - !GetAtt CanaryBucket.Arn
            - Effect: Allow
              Action:
              - logs:CreateLogStream
              - logs:PutLogEvents
              - logs:CreateLogGroup
              Resource:
              - "*"
            - Effect: Allow
              Action:
              - s3:ListAllMyBuckets
              - xray:PutTraceSegments
              Resource:
              - "*"
            - Effect: Allow
              Resource: "*"
              Action: cloudwatch:PutMetricData
              Condition:
                StringEquals:
                  cloudwatch:namespace: CloudWatchSynthetics

  SyntheticsCanary:
      Type: 'AWS::Synthetics::Canary'
      Properties:
          Name: !Join ['', ['canary_', !Ref ENVIRONMENT]]
          ExecutionRoleArn: !Ref CanaryRole
          Code: {Handler: pageLoadBlueprint.handler, Script: "const synthetics = require('Synthetics');\n\nconst apiCanaryBlueprint = async function () {\n    const validateSuccessful = async function (res) {\n        return new Promise((resolve, reject) => {\n            if (res.statusCode < 200 || res.statusCode > 299) {\n                throw new Error(res.statusCode + ' ' + res.statusMessage);\n            }\n            res.on('end', () => {\n                resolve();\n            });\n        });\n    };\n\n    let request = {\n        hostname: 'api-'+process.env.ENVIRONMENT+'.helbizscooters.com',\n        method: 'GET',\n        path: '/health',\n        port: '443',\n        protocol: 'https:',\n        body: '',\n        headers: {\n            'User-Agent': synthetics.getCanaryUserAgentString()\n        }\n    };\n    let config = {\n        includeRequestHeaders: false,\n        includeResponseHeaders: false,\n        includeRequestBody: false,\n        includeResponseBody: false,\n        continueOnHttpStepFailure: true\n    };\n\n    await synthetics.executeHttpStep('Verify api', request, validateSuccessful, config);\n    request.path = '/live/healthcheck';\n    await synthetics.executeHttpStep('Verify live', request, validateSuccessful, config);\n};\n\nexports.handler = async () => {\n    return await apiCanaryBlueprint();\n};"}
          ArtifactS3Location: s3://      canray bucket
          RuntimeVersion: syn-nodejs-puppeteer-3.9
          RunConfig: 

          Schedule: {Expression: 'rate(1 minute)', DurationInSeconds: 3600}
          RunConfig: {TimeoutInSeconds: 60}
          FailureRetentionPeriod: 30
          SuccessRetentionPeriod: 30
          StartCanaryAfterCreation: true

  CanaryAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: Canary health check
      AlarmName: !Join ['', ['canaryalarm', !Ref ENVIRONMENT]]
      ComparisonOperator: LessThanLowerOrGreaterThanUpperThreshold
      MetricName: SuccessPercent
      Namespace: CloudWatchSynthetics
      Statistic: Average
      Period: 120
      EvaluationPeriods: 1
      Threshold: 99
      Dimensions:
        - Name: CanaryName
          Value: !Join ['', ['canary_', !Ref ENVIRONMENT]]
      ComparisonOperator: LessThanThreshold

```