---
title: "Making a fully compliant Mongo Atlas connection"
date: 2023-11-01T15:34:30-04:00
categories:
    - blog
tags:
    - Mongo Atlas
    - AWS
    - Compliance
---

![image](/assets/images/monbgoiam.png)
# Making a compliant Mongo Atlas database connection
## Why

Let's say we want to connect to a Mongo Atlas cluster somewhere from our AWS account. The default auth type is username/password but this isn't all that suitable for a massive production system. Even if we limit network access to our VPC, use AWS Secrets Manager for storing passwords and our servers retrieve them at runtime, we have no possible way to verify that they aren't also stored and used from somewhere else in the company. Any auditor working on getting you a ISO/IEC 27001 or similar certification would show you this deficiency in the system and would demand additional proof that unauthorized personnel don't have unrestricted access to the database.

For this purpose we could maybe combine VPC flow logs and Mongo Database access logs, but there is a simpler solution with less overhead, that is compatible with any type of network topography or setting, using AWS IAM auth with Mongo Atlas.

## IAM Connection in Atlas

First, create a role without any permissions in AWS IAM. This will be the our dedicated Mongo connection role. Now allow the services you want to connect to the database to assume this role. In this case I am trying to connect a server that's deployed on an AWS Lambda, so I edit the Trust Relationship of the Mongo role to allow the lambda's execution role to assume it.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::{your aws account id}:role/service-role/{role name}"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

## Settings in Mongo Atlas

Now let's go to Mongo Atlas and create a new Database access user on your cluster:

![image](/assets/images/mongo1.png)

Pick AWS IAM as the auth method and IAM Role as IAM Type. Then copy the ARN of the role we created in the previous step.

Additional documentation on this [link](https://www.mongodb.com/docs/atlas/security/passwordless-authentication/)

## Connection example

Going over to the connection
On the server side we first assume the Mongo connection role, extract credentials from it and create a new connection string using them like this: [^1]

```typescript
  let role = await  new STS({region: 'eu-west-1',}).assumeRole({
    RoleArn: process.env.MONGO_ROLE!
    RoleSessionName: 'connection-server-' + Date.now(),
  });

  const dbURL = new URL(`mongodb+srv://${process.env.MONGO_HOST}/${process.env.MONGO_DB}`);
  dbURL.username = role?.Credentials?.AccessKeyId;
  dbURL.password = role?.Credentials?.SecretAccessKey;
  dbURL.searchParams.append('authSource', '$external');
  dbURL.searchParams.append('authMechanism', 'MONGODB-AWS');
  dbURL.searchParams.append('authMechanismProperties', `AWS_SESSION_TOKEN:${role?.Credentials?.SessionToken}`);
  process.env.MONGO_STR = dbURL.href;
  let dbClient = new MongoClient(process.env.MONGO_STR, {});
  await dbClient.connect();
  let DB = dbClient.db();
```

In this code snippet the environment variables MONGO_HOST, MONGO_DB and MONGO_ROLE are used respectively for the Mongo cluster domain, database name and the previously created Mongo connection role.

## Monitoring

While we made sure through the trust policy that only the lambda role can assume the mongo role we can also add some monitoring just to make sure.

We can use AWS CloudTrail for this, as it allows us to log all AssumeRole and similar events. Just create a new trail [here](https://eu-west-1.console.aws.amazon.com/cloudtrail/home?region=eu-west-1#/create) and make sure you enable CloudWatch logs for better readability through the CW tools instead of reading raw files on S3.[^2]

After creating the trail and several minutes of operation we can check and see if it's working through the CloudWatch Logs Insights feature. Use the following query on the CloudWatch log group you created together with the trail. Just plug into the query the name of the mongo connection role:

```
fields @timestamp,
userIdentity.principalId' as identity, sourceIPAddress, userIdentity.sessionContext.sessionIssuer.arn' as role
| sort @timestamp desc
| filter requestParameters.roleArn = '{role ARN}' and userIdentity.type = 'AssumedRole'

```

You will get a table of all of the different identities which tried to assume the connection role and their IP addresses.

If necessary you can setup a CloudWatch alarm on this log group which would notify you if somehow someone else assumes the connection role.

## Links and Documentation

[^1]: Mongo Atlas AWS Auth connection string: https://www.mongodb.com/docs/kafka-connector/current/security-and-authentication/mongodb-aws-auth/
[^2]: Cloudtrail User Guide: https://docs.aws.amazon.com/IAM/latest/UserGuide/cloudtrail-integration.html#cloudtrail-integration_examples-sts-api
