---
title: "Fullstack Cloudformation stack with rollback"
date: 2023-11-01T15:34:30-04:00
categories:
  - blog
tags:
  - Mongo Atlas
  - AWS
  - Compliance
---

# Fullstack Cloudformation stack with rollback

## Server

First lets add two lambda functions with a lambda execution role together with function URLs which will allow us to call them from the outside

```
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
        - arn:aws:iam::aws:policy/AWSLambdaExecute
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          Effect: Allow
          Principal:
            Service: lambda.amazonaws.com
          Action: sts:AssumeRole


  BlogsFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./blogs
      Handler: dist/lambda.handler
      MemorySize: 1024
      Role: !GetAtt LambdaExecutionRole.Arn
      Runtime: nodejs18.x
      Timeout: 30

  BlogsFunctionURL:
    Type: AWS::Lambda::Url
    Properties: 
      AuthType: NONE
      TargetFunctionArn: !Ref BlogsFunction

  ApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./api
      Handler: dist/lambda.handler
      MemorySize: 1024
      Role: !GetAtt LambdaExecutionRole.Arn
      Runtime: nodejs18.x
      Timeout: 30

  ApiFunctionURL:
    Type: AWS::Lambda::Url
    Properties: 
      AuthType: NONE
      TargetFunctionArn: !Ref ApiFunction

```

Now we could end here but we don't want to lock down the function urls in our frontend
function urls are randomly generated
we want to have all of our APIs grouped in a single dns
cloudfront also allows us to attach additional stuff like WAF, user info headers and logging

So we create a cloudfront distribution which and put the two functions behind it.
This gives us a randomly generated cloudfront distribution url, but if we are creating multiple environments we might want an easy way to tell them apart, so I added a dns record to the template which gets us a nice subdomain based on the current branch.

```
  CloudFrontDistribution:
    DependsOn:
      - BlogsFunctionURL
      - ApiFunctionURL
    Type: "AWS::CloudFront::Distribution"
    Properties:
      DistributionConfig:
        Aliases:
          - !Sub "api-${BRANCH}.domain.com"
        ViewerCertificate:
          AcmCertificateArn: arn:aws:acm:us-east-1:841805187071:certificate/0e0cc568-ea74-4df3-a4c1-e51e6b3c3877
          MinimumProtocolVersion: TLSv1.2_2019
          SslSupportMethod: sni-only
        DefaultCacheBehavior:
          TargetOriginId: ApiFunction
          ViewerProtocolPolicy: "redirect-to-https"
          CachePolicyId: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad
          OriginRequestPolicyId: b689b0a8-53d0-40ab-baf2-68738e2966ac
          AllowedMethods: [ 'GET', 'HEAD', 'OPTIONS', 'PUT', 'PATCH', 'POST', 'DELETE' ]
        CacheBehaviors:
          - AllowedMethods: [ DELETE, GET, HEAD, OPTIONS, PATCH, POST, PUT ]
            TargetOriginId: BlogsFunction
            PathPattern: /blogs
            ViewerProtocolPolicy: redirect-to-https
            CachePolicyId: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad
            OriginRequestPolicyId: b689b0a8-53d0-40ab-baf2-68738e2966ac
        Enabled: true
        HttpVersion: http2
        Origins:
          - Id: ApiFunction
            DomainName: !Select [2, !Split ["/", !GetAtt ApiFunctionURL.FunctionUrl]]
            CustomOriginConfig:
              HTTPSPort: 443
              OriginProtocolPolicy: https-only
          - Id: BlogsFunction
            DomainName: !Select [2, !Split ["/", !GetAtt BlogsFunctionURL.FunctionUrl]]
            CustomOriginConfig:
              HTTPSPort: 443
              OriginProtocolPolicy: https-only
        PriceClass: "PriceClass_All"

  Domain:
    DependsOn:
      - CloudFrontDistribution
    Type: "AWS::Route53::RecordSetGroup"
    Properties:
      HostedZoneId: Z249BWKE7HL1UY //default
      RecordSets:
        - Name: !Sub "api-${BRANCH}.domain.com"
          Type: A
          AliasTarget:
            HostedZoneId: Z2FDTNDATAQYW2  // domain.com zone
            DNSName: !GetAtt
              - CloudFrontDistribution
              - DomainName

```

## Frontend

Frontedn will be hosted on a S3 bucket behind a cloudfront distribution, in this case because of its caching features. similarily as in the APIs case we add an easy-to-read dns to our existing domain

```
  S3Bucket:
    Type: "AWS::S3::Bucket"
    Properties:
      BucketName: !Sub "domain-${Name}-${Branch}"
      WebsiteConfiguration:
        ErrorDocument: "index.html"
        IndexDocument: "index.html"

  ReadPolicy:
    Type: "AWS::S3::BucketPolicy"
    Properties:
      Bucket: !Ref S3Bucket
      PolicyDocument:
        Statement:
          - Action: "s3:GetObject"
            Effect: Allow
            Resource: !Sub "arn:aws:s3:::${S3Bucket}/*"
            Principal:
              CanonicalUser: !GetAtt CloudFrontOriginAccessIdentity.S3CanonicalUserId

  CloudFrontOriginAccessIdentity:
    Type: "AWS::CloudFront::CloudFrontOriginAccessIdentity"
    Properties:
      CloudFrontOriginAccessIdentityConfig:
        Comment: !Ref S3Bucket

  S3CloudFront:
    DependsOn:
      - S3Bucket
    Type: "AWS::CloudFront::Distribution"
    Properties:
      DistributionConfig:
        Aliases:
          - !Sub "${Name}-${Branch}.domain.com"
        ViewerCertificate:
          AcmCertificateArn: arn:aws:acm:us-east-1:841805187071:certificate/0e0cc568-ea74-4df3-a4c1-e51e6b3c3877
          MinimumProtocolVersion: TLSv1.2_2019
          SslSupportMethod: sni-only
        CustomErrorResponses:
          - ErrorCode: 403 # not found
            ResponseCode: 200
            ErrorCachingMinTTL: 10
            ResponsePagePath: "/index.html"
          - ErrorCode: 404 # not found
            ResponseCode: 200
            ErrorCachingMinTTL: 10
            ResponsePagePath: "/index.html"
        DefaultCacheBehavior:
          CachePolicyId: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad
          TargetOriginId: s3origin
          ViewerProtocolPolicy: "redirect-to-https"
        # This DefaultRootObject configuration is not enough.
        # DefaultRootObject: "/index.html"
        Enabled: true
        HttpVersion: http2
        Origins:
          - DomainName: !Sub "domain-${Name}-${Branch}.s3.${AWS::Region}.amazonaws.com"
            Id: s3origin
            S3OriginConfig:
              OriginAccessIdentity: !Sub "origin-access-identity/cloudfront/${CloudFrontOriginAccessIdentity}"
        PriceClass: "PriceClass_All"

  FrontDNS:
    DependsOn:
      - S3CloudFront
    Type: "AWS::Route53::RecordSetGroup"
    Properties:
      HostedZoneId: Z249BWKE7HL1UY
      RecordSets:
        - Name: !Sub "${Name}-${Branch}.domain.com"
          Type: A
          AliasTarget:
            HostedZoneId: Z2FDTNDATAQYW2
            DNSName: !GetAtt
              - S3CloudFront
              - DomainName
```


## Deploy procedure

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
          Name: !Join ['', ['canary_', !Ref BRANCH]]
          ExecutionRoleArn: !Ref CanaryRole
          Code: {Handler: pageLoadBlueprint.handler, Script: "const synthetics = require('Synthetics');\n\nconst apiCanaryBlueprint = async function () {\n    const validateSuccessful = async function (res) {\n        return new Promise((resolve, reject) => {\n            if (res.statusCode < 200 || res.statusCode > 299) {\n                throw new Error(res.statusCode + ' ' + res.statusMessage);\n            }\n            res.on('end', () => {\n                resolve();\n            });\n        });\n    };\n\n    let request = {\n        hostname: 'api-'+process.env.BRANCH+'.helbizscooters.com',\n        method: 'GET',\n        path: '/health',\n        port: '443',\n        protocol: 'https:',\n        body: '',\n        headers: {\n            'User-Agent': synthetics.getCanaryUserAgentString()\n        }\n    };\n    let config = {\n        includeRequestHeaders: false,\n        includeResponseHeaders: false,\n        includeRequestBody: false,\n        includeResponseBody: false,\n        continueOnHttpStepFailure: true\n    };\n\n    await synthetics.executeHttpStep('Verify api', request, validateSuccessful, config);\n    request.path = '/live/healthcheck';\n    await synthetics.executeHttpStep('Verify live', request, validateSuccessful, config);\n};\n\nexports.handler = async () => {\n    return await apiCanaryBlueprint();\n};"}
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
      AlarmName: !Join ['', ['canaryalarm', !Ref BRANCH]]
      ComparisonOperator: LessThanLowerOrGreaterThanUpperThreshold
      MetricName: SuccessPercent
      Namespace: CloudWatchSynthetics
      Statistic: Average
      Period: 120
      EvaluationPeriods: 1
      Threshold: 99
      Dimensions:
        - Name: CanaryName
          Value: !Join ['', ['canary_', !Ref BRANCH]]
      ComparisonOperator: LessThanThreshold

```