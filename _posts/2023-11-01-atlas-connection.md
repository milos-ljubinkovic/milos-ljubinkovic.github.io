---
title: "Forget Mongo Passwords: AWS IAM Atlas Authentication"
date: 2023-11-01T15:34:30-04:00
categories:
    - blog
tags:
    - Mongo Atlas
    - Devops
    - AWS
    - Compliance
toc: true
toc_label: "Contents"
toc_icon: "book-open"
---

![image](/assets/images/monbgoiam.png)

## Why

You don't want to deal with passwords. They are hard to keep track of, they get everywhere and on their own they provide zero accountability.

Inside of the AWS ecosystem we use IAM, we assign permissions to roles and roles to actors and this fixes a lot of those problems. Now, Mongo Atlas relatively recently added an option to use IAM when authenticating to your database which is documented [here](https://www.mongodb.com/docs/atlas/security/passwordless-authentication/), but I didn't find a full guide on how (or why) to use only IAM connections.

Some problems you fix by ditching passwords completely:

1. Completely removes one vector of attack (if you aren't network limiting your database connections, which you should but that's a different post)

2. In the case of a security breach or a theft you don't need to rotate or reset passwords

3. You don't have to keep track of who has access to what, everything is visible in IAM dashboard

4. In the case of a security audit (ex ISO/IEC 27001) you have an immutable log of who had which permissions when and even who had the permission to give access

Now, you can create a solution that achieves all of this using Secrets Manager, some additional network rules with flow logs, mongo access logs etc. But the solution in this post has the fewest moving parts, it's easier to maintain and can be easily extended through IAM integrations with other tools (like SSOs).

## IAM Connection in Atlas

First, create a **"MongoAccessRole"** IAM role without any permissions in AWS IAM. This will be the our dedicated Mongo connection role. Allow the services you want to connect to the database to assume this role.

In this case I am trying to connect a server that's deployed on an AWS Lambda and I want to allow the members of the IAM Group DBAdmins to connect (do not recommended for production db).

So I edit the Trust Relationship of the Mongo role to allow the lambda's execution role and users in general to assume it.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::{your aws account id}:role/service-role/{Lambda execution role}"
            },
            "Action": "sts:AssumeRole"
        },
        {
            "Sid": "Statement2",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::{your aws account id}:root"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

Then we create a Permission policy **"AssumeMongoConnectionRole"**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::{your AWS acc number}:role/MongoConnectionRole"
        }
    ]
}
```

And we attach it to the Lambda execution role and to the user group DatabaseAdmins.

Now all lambdas with that specific execution role and all users in that user group will be able to connect to the database.

## Settings in Mongo Atlas

Now let's go to Mongo Atlas and create a new Database access user on your cluster:

![image](/assets/images/mongo1.png)

Pick AWS IAM as the auth method and IAM Role as IAM Type. Then copy the ARN of the **MongoAccessRole** role we created in the previous step.

Additional documentation on this [link](https://www.mongodb.com/docs/atlas/security/passwordless-authentication/)

## Server Connection Example

Going over to the connection

On the server side we first assume the Mongo connection role, extract credentials from it and create a new connection string using them like this: [^1]

```typescript
  import { STS } from '@aws-sdk/client-sts';
   // ... //
  let role = await new AWS.STS({region: 'eu-west-1',}).assumeRole({
    RoleArn: process.env.MONGO_ROLE!,
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

In this code snippet the environment variables **MONGO_HOST**, **MONGO_DB** and **MONGO_ROLE** are used respectively for the Mongo cluster domain, database name and the previously created **"MongoAccessRole"** role.

## Compass Connection Example

If we are a developer in the DatabaseAdmins IAM user group and we want to use [Mongo Compass](https://www.mongodb.com/products/tools/compass) to connect to our database we would have to first get the credentials from the previous step and then enter them into the Compass connection fields.

Now this is troublesome and I fear that passwords would creep back in if we mandated something like that. So let us automate that part as well. We can take the credentials same as above, save them into a file in the proper format [^2] and the start Compass from there.

```typescript
    import { exec } from 'node:child_process';
    // ... //
    let temp = {
      type: 'Compass Connections',
      version: {
        $numberInt: '1',
      },
      connections: [
        {
          id: 'iam',
          connectionOptions: {
            connectionString: dbURL.href,
          },
        },
      ],
    };

    await writeFileSync('./mongo-connection.json', JSON.stringify(temp));
    exec('"/Applications/MongoDB Compass.app/Contents/MacOS/MongoDB Compass" --file=./mongo-connection.json iam');

```
* The path to your executable file might vary


## Monitoring

While we made sure through the trust policy that only the lambda role can assume the mongo role we can also add some monitoring just to make sure.

We can use AWS CloudTrail for this, as it allows us to log all AssumeRole and similar events. Just create a new trail [here](https://eu-west-1.console.aws.amazon.com/cloudtrail/home?region=eu-west-1#/create) and make sure you enable CloudWatch logs for better readability through the CW tools instead of reading raw files on S3.[^3]

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

[^1]: Mongo Atlas AWS Auth connection string: [https://www.mongodb.com/docs/kafka-connector/current/security-and-authentication/mongodb-aws-auth/](https://www.mongodb.com/docs/kafka-connector/current/security-and-authentication/mongodb-aws-auth/)


[^2]: Start Compass from the Command Line: [https://www.mongodb.com/docs/compass/current/connect/connect-from-the-command-line/#configuration-file-connection-specification](https://www.mongodb.com/docs/compass/current/connect/connect-from-the-command-line/#configuration-file-connection-specification)


[^3]:
    Cloudtrail User Guide: [https://docs.aws.amazon.com/IAM/latest/UserGuide/cloudtrail-integration.html#cloudtrail-integration_examples-sts-api
    ](https://docs.aws.amazon.com/IAM/latest/UserGuide/cloudtrail-integration.html#cloudtrail-integration_examples-sts-api)

