---
title: "Fullstack Cloudformation stack with rollback"
date: 2023-11-01T15:34:30-04:00
categories:
  - blog
tags:
  - AWS
  - Cloudformation
  - Devops
---

# Fullstack Cloudformation stack with rollback

## Intro

In this post we'll go through creating a reliable and easily scalable full-stack solution on AWS with an safe and easy deploy procedure. For hosting the backend server we'll use AWS Lambda. This simplifies the deploy procedure by a lot and gives us a bunch of tools to use in the future in regards to scaling. 

Frontend will be hosted as a simple static site on S3 sitting behind a Cloudfront distribution.

The final stack definition file with some extras can be found at https://github.com/milos-ljubinkovic/full-stack

## Cloudformation

Instead of using console or just the aws-cli to create this stack we'll use [AWS Cloudformation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) templates. Defining all of the AWS resources in a single JSON or YAML template will make the resulting stack much easier to replicate and maintain than using custom scripts or other solutions.

The [Template Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html) part of the documentation contains explanations for all fields and objects in the template that we'll write.

Most of the resources can be indefinitely recreated in a single AWS account, and if you deploy this template multiple times those resources will just automatically get unique names and ids with the name of the specific stack as a prefix. Others like S3 buckets and DNS entries must have globally unique names or other values. In those cases we'll insert the name of the deployed stack using the `${AWS::StackName}` variable.

TODO: Migrate to CDK

## Server

First lets add two lambda functions with a lambda execution role together with function URLs which will allow us to call them from the outside

CodeUri parameter is the local path to the folder containing the code of the specific function.

FunctionURLs are just boilerplate plugins to the lambda functions that'll allow us to trigger the functions from outside of the AWS system using standard HTTP.

The Role object is also boilerplate at this point, but in the future we would expand this role with other policies if we need the lambdas to access other AWS services or resources.


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

Now we could end here but this results in a pair of randomly generated immutable URLs, which is something we don't want in our production code. We'll now put the function URLs behind a Cloudfront distribution in order to get some more flexibility within our stack and to help with the future maintainability of the system. 

Cloudfront also gives us some cool stuff as an extra. Stuff like:
- Ability to attach [WAF](https://docs.aws.amazon.com/waf/latest/developerguide/what-is-aws-waf.html) to help with security and spam prevention
- [Cloudfront headers](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/adding-cloudfront-headers.html) for easier user handling and content personalization
- Logging and monitoring at the networking layer
- etc

I also added a [Route53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html) DNS alias to the distribution, this isn't necessary as Cloudfront covers most of our routing needs, but it gives us a readable and easier to remember URL. Which is a must if you are working with multiple deployed environments or just with other people and you want to make sure you are talking about the same deployment.


```
  CloudFrontDistribution:
    DependsOn:
      - BlogsFunctionURL
      - ApiFunctionURL
    Type: "AWS::CloudFront::Distribution"
    Properties:
      DistributionConfig:
        Aliases:
          - !Sub "api-${ENVIRONMENT}.domain.com"
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
        - Name: !Sub "api-${ENVIRONMENT}.domain.com"
          Type: A
          AliasTarget:
            HostedZoneId: Z2FDTNDATAQYW2  // domain.com zone
            DNSName: !GetAtt
              - CloudFrontDistribution
              - DomainName

```

## Frontend

Frontend will be hosted on a S3 bucket behind a different Cloudfront distribution. 

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
