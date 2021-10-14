# AWS IAM Deep Dive. Chapter 2: Users, Groups, Roles, Policies

Let’s keep going, friends. The first concept we are going to look into is IAM Users.

## IAM Users

So what are IAM Users? According to [AWS documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction_identity-management.html#intro-identity-users):

> The “identity” aspect of AWS Identity and Access Management (IAM) helps you with the question “Who is that user?”, often referred to as authentication.

So IAM User has the same concept as any other “users” (in LDAP, DB, App, Facebook, Google, etc.).

IAM User itself is nothing, just an entry in the directory. It can not access AWS on its own, it doesn’t have any permissions (being more specific — it has implicit — “default” — DENY).

There are 2 ways IAM user authenticates against AWS:

 - API Access key/secret — those are used with awscli or AWS SDK on your own machine
 - Login/Password — that you can use only with the AWS Console

In addition to that, IAM User also has security credentials to be used with AWS CodeCommit (SSH/HTTPS). You can attach an IAM Policy or create a so-called Inline Policy — a policy which “lives” in that user and can only be used by that user.

The creation of IAM users is fairly obvious. Since this is a more advanced blog post series, I’m not going to post screenshots of “Next-Next-Finish” from the Console. Instead — awscli is my best friend.

```
aws iam create-user --user-name Ivan
```

If you want some logical hierarchy, you can use [paths](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_identifiers.html#identifiers-friendly-names). This will come in handy when you work with IAM [variables](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_variables.html).

```
aws iam create-user --user-name Ivan --path /developers/
```

The difference in the example above is ARN. If you create a user without specifying the path, your user ARN is `arn:aws:iam::accoint_id:user/Ivan`, while with the path it is `arn:aws:iam::account_id:user/developers/Ivan`.

Note that path elements are not the same as IAM Groups. You can not attach a policy to all users in a certain path.

### Inline policies

As mentioned above, Inline Policy is a policy attached directly to the user and is permitted to that exact user. This policy cannot be shared with other users, groups, or roles. I can’t see the real use case where you should use it (IMHO, it’s a bad practice), but if I promised to cover as much as I can, then I will. ;)

Picking the same example policy, you saw in the previous chapter:

```
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

Save it to the text file on your local disk, and run the following command to create an inline policy and attach it to that user:

```
aws iam put-user-policy --user-name Ivan --policy-name InlineIsBad --policy-document file://policy.json
```

`put-user-policy` is different from `attach-user-policy`. The first command will create and attach an inline policy, while the second attaches the existing one.

### IAM User Limits

There is not much we need to learn about IAM Users (in real-world production environments, you do not use them in favor of Identity Federation and IAM Roles), but there some noticeable limits you’d like to know (the whole list is available [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-limits.html), so please check, cause AWS is changing and at the moment of reading this article the information below might not be valid anymore).

 - You can not have more than 2 API key/secret pairs assigned to a user (let it be root or IAM)
 - IAM user can belong to not more than 10 groups at the same time
 - You can have a maximum of 5000 IAM Users within an account. Now you might think that’s enough, but keep in mind that you will have to use IAM users for Git auth on CodeCommit. That basically means that you can have max 5000 developers with Git credentials to work with CodeCommit in the same account.

## IAM Groups

Another entity in AWS IAM is an IAM Group. IAM Group is similar to LDAP groups. It is used to have a grouped and centralized authorization matrix for multiple IAM users (instead of attaching policies to each user directly).

IAM Groups cannot be used in the same manner as IAM users (you can not authenticate as a group), but they also have inline policies and managed policies attached to a group.

Creation is fairly simple:

```
aws iam create-group --group-name SuperStars
```

Again you can apply paths to it if you favor it. Creation of inline policies is similar in the case of groups too:

```
aws iam put-group-policy --group-name SuperStars --policy-document file://policy.json --policy-name InlinePolicyIsStillBad
```

To mention — policies attached to the Group do not limit but extend the policies attached to the IAM user. If the User has Full Access to S3 and the Group has Full Access to EC2, it will have Full Access to both EC2 and S3.

Still, not covering too much here cause Groups is boring (and so 2008), so let’s move to a more interesting topic — IAM Roles.

## IAM Roles

Now, if you work with AWS for more than 1 month and you ever needed to grant your EC2 instances with access to S3 or DynamoDB, you have likely read this kind of message on any documentation, blog, or training portal.

> DO NOT USE HARDCODED CREDENTIALS AND NEVER STORE API KEY AND SECRET ON THE EC2 INSTANCE. NEVER! BAD PRACTICE! BAD!

![HARDCODED KEYS IS BAD](https://miro.medium.com/max/1154/0*b1HqHXxStFaCNNBn.jpg "HARDCODED KEYS IS BAD AND YOU SHOULD FEEL BAD")

You don’t need to be a genius to know about the danger of hardcoded credentials (especially if they are not encrypted or hashed). The security issue with API keys is that they don’t require 2-step verification for authentication.

You can apply some extra layer of security to force users to use MFA and assume IAM role, which has the necessary set of policies for your engineers to work, but if an IAM user has some policies (either attached or inline) — the one who has the keys has the same powers.

To eliminate that risk and still provide the ability for your applications to communicate with other AWS services, IAM has a concept of [Role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html):

> An IAM role is an IAM identity that you can create in your account that has specific permissions. An IAM role is similar to an IAM user, in that it is an AWS identity with permission policies that determine what the identity can and cannot do in AWS. However, instead of being uniquely associated with one person, a role is intended to be assumable by anyone who needs it. Also, a role does not have standard long-term credentials such as a password or access keys associated with it. Instead, when you assume a role, it provides you with temporary security credentials for your role session.

To obtain IAM Role privileges, an entity (IAM user, Lambda Function, ECS Task, EC2 Instance, etc.) needs to assume that role.

Here AWS introduces another service called Security Token Service or STS.

STS is a service that provides you with short-term credentials. When you invoke STS API with a method, GetSessionToken your response will contain 3 credential items:

 - Session Key
 - Session Secret
 - Session Token

These credentials have TTL, which you define when invoking that method. When the duration is expired, you will have to “refresh” them and start your session over.

`GetSessionToken` can be used separately, but it is mostly used as a part of `AssumeRole`. But we’re getting a bit far of the scope of IAM Roles…

### IAM Role internals

IAM Role is created as the same object as others in IAM. However, it has a key difference, which is called AssumeRolePolicyDocument. This is a policy document that has only one action attached called `sts:AssumeRole`.

This document looks like this:

```
{
  "Version": "2012-10-17",
  "Sid": "AllowAssumeRole",
  "Statement": [ 
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Principal": YOUR_PRINCIPAL_GOES_HERE
    }
  ]
}
```

Although it looks like any other policy document, the difference here is that we don’t have a resource, which this action is applied to, but the Principal.

The [Principal](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_principal.html) is an entity which is allowed to perform AssumeRole action. It can be:

 - AWS Account (you place Account ID)
 - IAM user (a single IAM user ARN or a list of IAM User ARNs)
 - Service (Full name entry of that AWS Service like ec2.amazonaws.com or ecs-task.amazonaws.com and so on)
 - Federated users (specify Federation URL for Cognito, Google, etc)
 - Everyone (means that everyone can assume that role, specified by a wildcard “*”)

The AssumeRolePolicyDocument is required when you create a role. Once the document is saved as JSON file you can create a role.

```
aws iam create-role --role-name MyRole --assume-role-policy-document file://PolicyDocument.json
```

Role again can have an inline or attached policy, we already went through this.

Once the role is created, another entity can assume it, based on the document (in Console you can find it in the Trust Relationships).

If the Document allows EC2 instances to assume that role (specified as a Service, not as an AWS Principal with ARN), any EC2 instance can assume that role, but only EC2 instance.

One thing to mention: EC2 instance requires an IAM entry called “[Instance Profile](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html)” — this is just an entry which is linked to the IAM role. You can create an Instance Profile and link it to the Role with the following commands:

```
aws iam create-instance-profile --instance-profile-name MyProfile
aws iam add-role-to-instance-profile --instance-profile-name MyProfile --role-name MyRole
```

Note that Instance Profiles are not shown on the Console. You will have to use *awscli* or SDK to list them.

Instance Profile is only applied to EC2 instances. Other AWS Services, like Lambda, CloudWatch, etc will need just a Role with necessary policies, but make sure you’ve set a correct Trust Relationship.

The best practice here is to follow the “[Principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege)”. That principle means that any entity must have *that* exact amount of permission to perform its function, not more.

If you specify AWS Service (making an AWS Service Linked Role) that means any resource within that service will be able to assume that role. To increase the level of paranoia, make sure you monitor which resources or entities assume that role.

## IAM Policies

As mentioned in the first chapter the Policy is a set of permissions which you apply to a user, group or role.

The policy contains statements. Each statement contains:

1. Effect — Allow or Deny
2. Action — API calls
3. Resource — on which resource this statement has an effect
4. Condition — in which circumstances this statement is applied

Let’s take a look at our example IAM Policy which allow Full Access to S3:

```
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

So what’s here? The policy has an Allow effect on all S3 API calls and on (not by) any resource.

That means that entity which is these privileges can do anything with S3. It can create or delete a bucket, put and get an object, set encryption, set versioning or Bucket Policy and so on.

We set no Conditions here, so there are no limits on that policy. Once attached — nothing can’t stop me from doing stuff (Well, there are things like Bucket policies (S3 specific limitations), Permission boundaries or Service Control Policies — but we’ll cover them in the later chapters).

Since there is a wildcard on Resource, that allows us to work with any bucket within the account.

Let’s say we want to allow a user to work with a specific bucket mybucket. To do so, I need to specify a bucket in the Resource section.

```
{
  "Version": "2012-10-17",
  "Sid": "AllowCreateBuckets",
  "Statement": [ 
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::mybucket"
    }
  ]
}
```

But specifying the bucket I limit API calls to the bucket (and only to the bucket).

Now let’s think together. Let’s say my bucket has some object in it. When I want to invoke GetObject to what this called is done?

The answer is — to the object. And since I limited access to calls specifically to the bucket I will get 403 Access Denied. So proper Resource section should look like that:

```
{
  "Version": "2012-10-17",
  "Sid": "AllowCreateBuckets",
  "Statement": [ 
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::mybucket",
        "arn:aws:s3:::mybucket/*"
      ]
    }
  ]
}
```

Looks better, right? But there is one more thing I’m worried about — I don’t want to allow this policy to delete objects from the bucket. How do I do that again?

We already know that explicit Deny (a Deny effect stated in the policy) always overrides the Allow. So we need to add extra lines in the Statement block:

```
{
  "Version": "2012-10-17",
  "Sid": "AllowCreateBuckets",
  "Statement": [ 
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::mybucket",
        "arn:aws:s3:::mybucket/*"
      ]
    },
    {
      "Effect": "Deny",
      "Action": "s3:Delete*",
      "Resource": [
        "arn:aws:s3:::mybucket/*",
        "arn:aws:s3:::mybucket"
      ]
    }
  ]
}
```

Now I’ve denied all delete calls to the bucket and its objects.

Actions section is not limited by one specific service. We can have multiple permissions with Deny or Allow effect:

```
{
  "Effect": "Allow",
  "Action": [
    "s3:*",
    "ec2:*",
    "iam:*"
    # And so on    
  ]
}
```

Or specifying permissions one by one:

```
{
  "Effect": "Allow",
  "Action": [
    "s3:ListBucket",
    "s3:PutObject",
    "s3:GetObject",
    "dynamodb:GetItem",
    "dynamodb:PutItem",
    # Still I can combine them with wildcards
    "iam:*"
  ]
}
```

And I will have to specify resources if I don’t give too many powers to that policy. If I apply multiple Actions within one Statement I will have to put multiple various resources.

```
{
  "Version": "2012-10-17",
  "Sid": "AllowCreateBuckets",
  "Statement": [ 
    {
    "Effect": "Allow",
    "Action": [
      "s3:ListBucket",
      "s3:PutObject",
      "s3:GetObject",
      "dynamodb:GetItem",
      "dynamodb:PutItem"
    ]
    "Resource": [
        "arn:aws:s3:::mybucket",
        "arn:aws:s3:::mybucket/*",
        "arn:aws:dynamodb:my_region:acct_id:table/mytable"
    ]
  }
  ]
}
```

Looks a bit dirty, right? When you have to work with multiple services or resources, it is better to have multiple Statement elements. Like this:

```
{
  "Version": "2012-10-17",
  "Sid": "AllowCreateBuckets",
  "Statement": [ 
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::mybucket",
        "arn:aws:s3:::mybucket/*"
      ]
  },
  {
    "Effect": "Allow",
    "Action": [
      "dynamodb:GetItem",
      "dynamodb:PutItem"
    ],   
    "Resource": "arn:aws:dynamodb:my_region:acct_id:table/mytable"
  }
  ]
}
```

Depending on the authorization matrix you want to implement in your company or project there are various ways to manage permissions.

You can either put them all in the single policy and attach that policy to the role. Or you can create different policies (each for some specific functionality) and attach them to the roles you need. The latter method is better, cause if you have to apps which have the same set of permissions (let’s say 2 applications which need to work with the same DynamoDB table) it makes sense to create separate IAM policy and attach it to the multiple roles.

### In conclusion

In this chapter, I explained IAM Users, Groups, Roles and Policies. I also briefly covered STS and how it interacts with IAM.

Got something on these topics I forgot to mention? Leave a response and I will update this article.

In the next chapter I am going to dive into the following :

 - Credential evaluation
 - Conditions
 - Principals
 - IAM policy variables

![We Need To Go Deeper](https://miro.medium.com/max/1300/0*ZCMqhOt4UaQbj4ag.jpg "We Need To Go Deeper")
