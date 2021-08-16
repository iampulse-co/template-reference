# AWS IAM In Layman's Terms: Identities

Defining IAM Identities. What are IAM Users, Groups, and Roles.

## Author: [Shuo Yang](https://syang.substack.com/p/aws-iam-in-a-laymans-words-1-identities)

 - [X] AWS
 - [X] Serverless

## Why IAM?

Engineers need to know security in the serverless paradigm development era.
Security is boring, but I believe that security knowledge is necessary to
engineers when we develop services in the serverless paradigm. Here is why.

We probably heard of Jeff Bezos’ “two pizza team rule”:

> We try to create teams that are no larger than can be fed by two pizzas

And that means the security design needs to be well-solved inside that team
(the job title 'DevSecOps').

For teams shifting toward serverless paradigm, a lot of security design
responsibility, if not all, will and should be taken by the serverless
service development team. But Why?

The reason is simple.

Because the infrastructure definition has already become part of serverless
development team’s code-base ("infrastructure as code").

Other than network security, **resource access control** is another security
topic we have to take care of to build a (micro) service or a product.

Why do we need to understand AWS IAM a little bit? **AWS made it clear that AWS IAM
is the foundation of solving the access control problem.**

> AWS Identity and Access Management (IAM) is a web service that helps you
securely control access to AWS resources. 

“Why does it matter to application developers? Isn’t all these security things
the business of our security team, who manage my access to our AWS account?”

Not necessarily. All of these could be and will be part of your code base if you
adopt the serverless paradigm.

For example, developers may want to understand AWS Cognito for authentication /
authorization mechanism. When we use Cognito Identity Pool, the authenticated
users will assume a role defined in the IAM under the hood.

## IAM Identities

![AWS IAM Workflow](https://cdn.substack.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fbucketeer-e05bbc84-baa3-437e-9518-adb32be77984.s3.amazonaws.com%2Fpublic%2Fimages%2F71c3c49f-50ad-4692-89e0-b08600434ce9_800x709.png "AWS IAM Workflow")

The above diagram is a very busy one, but we will focus on the top left corner
of it. This portion of the IAM defines the “who”.

## Users

An IAM user is "an entity that you create in AWS to represent the person", it
has the similar concept as "users" in an LDAP directory;

 - An IAM user has key_pair(s) associated to it (see the key associated with them
in the diagram below).

 - Though you could, typically we don’t assign IAM policy directly to a user.
Instead, we assign IAM policy to groups.

## Groups

"An IAM Group is a collection of IAM users", it has the similar concept as
"group" in an LDAP directory.

 - An IAM group can group a list of users together, so that user management
becomes manageable and maintainable (see the user grouping relationship in the
diagram below).

 - We typically specify the IAM policy at the group level.

## Roles

An IAM role is similar to an IAM user. However, instead of being uniquely
associated with one person, a role is intended to be _assumable_ by anyone who
needs it.

 - A role does not have standard long-term credentials such as a password or
access keys associated with it (see the diagram: there is not any key
associated with a role).

 - When you assume a role, it provides you with temporary security credentials for
your role session, though AWS STS (Security Token Service)

 - We typically define and associate the IAM policy to a role.

![Roles](https://cdn.substack.com/image/fetch/f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fbucketeer-e05bbc84-baa3-437e-9518-adb32be77984.s3.amazonaws.com%2Fpublic%2Fimages%2F4905f68a-3e2c-4ac4-8044-c68f6e3252b8_799x510.png "Roles")

## Summary
AWS IAM is the foundation of AWS resource access control. Services like Cognito
Identity Pool and Secure Token Service (STS) are built to facilitate the
application layer access control logics around it.

In this post we only touched upon the identity side of the equation. More to
come on other interesting details such as IAM policy evaluation.
