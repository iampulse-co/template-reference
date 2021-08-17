# AWS IAM Deep Dive. Chapter 1: Essentials

![The Key](https://miro.medium.com/max/1400/0*tSJCOuBaULmAQIgQ.png "The Key")

> So I’ve been looking over internets if there are any blog posts or series
> related to the internals of AWS IAM. Quite a surprise: there are none except
> several videos and hundreds of courses on portals like A Cloud Guru, Linux
> Academy, Udemy, etc.
> Although I do believe that there is a lot of effort done to deliver the
> knowledge and you should be paid for it, I also believe that knowledge should
> be shared (if possible). Hence I’m starting a blog series on AWS IAM —
> Identity and Access Management service.

## Prerequisites

While I’m going to cover most of IAM (literally taking it all out from AWS
documentation and providing it in an easy-to-read format) I assume you have
some working knowledge and understanding what IAM is and how it is used in AWS.

And of course, I assume you are coming from an IT background. ;)

---

## IAM structure

Before we can investigate the internals of IAM, let's memorize what is AWS.

AWS is a cloud provider offering a broad variety of services (at the moment of
this writing more than 160) in different areas: networking, compute, analytics,
databases, storage and so on.

AWS is an API based provider which means each service has its own API, which
you can invoke in order to perform some action or to retrieve some data, like
create a virtual machine, take a snapshot of the EBS volume, upload or retrieve
an object from S3 or an item from DynamoDB. Based on that different services
interact with each other: RDS uses S3 as a storage backend for DB snapshots,
Cloudwatch uses SNS for notifications, Cloudformation uses Lambda for Custom
Resources, etc, etc, etc.

To manage access permissions to those services AWS has created IAM. IAM is a
global service, which means it’s covering all regions within an account.

### IAM internals

IAM as a service is responsible for 2 processes:

 - Authentication — are you the one you claim to be?
 - Authorization — are you allowed to do what you want to do?

IAM has a lot of internal items:

 - Users — entities who have console login and password, API key/secrets and other security credentials.
 - Groups — similar to LDAP groups IAM Groups are used to manage permissions for multiple entities as a single entity.
 - Roles — while IAM group can delegate permissions to users, IAM Roles can manage permissions to any entities like Lambda functions, EC2 instances, etc.
 - Policies — set of permissions which can be assigned to a user, group or role.

And many other things like server certificates, federations, etc.

### How IAM works

Let’s take a look at the common AWS case. We have a user Lucy, who’s willing to
create an S3 bucket. Lucy has an API key/secret stored in ~/.aws/credentials
and uses awscli to create a bucket. From her side this process is simple:

```
aws s3 mb s3://lucy_bucket
```

What happens on the background is:

 1. awscli will check if there are credentials present (env vars, credential file, etc).
 2. awscli will make a call with provided credentials.
 3. AWS IAM will check if Lucy is authenticated (e.g. check if API key/secret combination is correct and exists in IAM database).
 4. AWS IAM will evaluate (will go over policy evaluation in the next chapters) if Lucy is authorized to perform s3:CreateBucket.
 5. AWS S3 API will receive a call to create a new bucket and return a response.

If all these parts work as intended Lucy will get a response 200 OK, but if not
she’ll get an error message (could a 5XX, but most likely 4XX)

### IAM Policy Essentials

We already know that there are various resource types in IAM. One of the core
components of IAM is Policy.

**IAM Policy contains a list of actions which are permitted to an entity on some specific resource or a service.**

Let’s look at this example:

```json
{
  "Version": "2012-10-17",
  "Sid": "AllowCreateBuckets",
  "Statement": [ 
    {
    "Effect": "Allow",
    "Action": "s3:CreateBucket",
    "Resource": "*"
    }
  ]
}
```

So what do we have here? We have a policy which allows the entity to perform an
API Call action “CreateBucket” on service AWS S3. We see that this call can be
made to any resource within that exact AWS account.

We know that there is already some boundary set. You cannot create a bucket on
EC2, because EC2 API doesn’t have such API Action. We also know that the entity
can create a bucket in any region (cause we simply do not limit it).

Since IAM policies support regexp, we can make “YOLO” policies, like this:

```json
{
  "Version": "2012-10-17",
  "Sid": "AllowCreateBuckets",
  "Statement": [ 
    {
    "Effect": "Allow",
    "Action": "s3:*",
    "Resource": "*"
    }
  ]
}
```

This will allow us to make any S3 API Action across the account.

Now the question is: let’s say I want to permit a user to do everything except
one (or two) calls. How I can achieve it without writing hundreds of actions in
the statement? The answer is “Deny”.

```
{
  "Version": "2012-10-17",
  "Sid": "AllowEverythingButNotCreateAndDeleteBuckets",
  "Statement": [ 
    {
    "Effect": "Allow",
    "Action": "s3:*",
    "Resource": "*"
    },
    {
    "Effect": "Deny",
    "Action": [
      "s3:CreateBucket",
      "s3:DeleteBucket"
    ],
    "Resource": "*"
    }
  ]
}
```

There is one core concept you **must** remember (and it will come in handy
during the exam): Deny always overrides an Allow. If you have “Allow” in the
IAM policy but Deny on the resource policy (or Service Control Policy) — that
Deny takes over. Always. No exceptions.

You can use "Allow"/"Deny" in conjunction with Conditions (we’ll get to them
too). For example, you can deny sts:AssumeRole if the user hasn’t logged in
with MFA, or SourceIpAddress is different.

Now that’s it for this chapter. If you have any questions or remarks feel free
to drop a comment.

There is a **lot to cover** in this series. Like really a lot! In the next
chapter, I am going to cover IAM Users, Groups, Roles and Policies. Stay tuned!
