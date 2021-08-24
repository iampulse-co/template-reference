# Control The Blast Radius Of Your Lambda Functions With An IAM Permissions Boundary

## Author: [Paul Swail](https://serverlessfirst.com/lambda-blast-radius-iam-permission-boundary/)

A great benefit of building Lambda-based applications is that the security best
practice of least privilege can be applied at a very granular level—the
individual Lambda function. This is done by creating a separate IAM role for
each function which grants the function just the permissions required to make
the AWS API calls that it needs to perform its task and nothing more. If a
single Lambda function is compromised, then the blast radius only goes as far
as the very limited (often single) API calls that this function’s role has
permissions to make.

Contrast this to an EC2-based application, where a single `EC2Application` IAM
role needs to define all the permissions that any component of the application
running on the instance needs access to. If one component gets compromised,
then the blast radius encompasses every single AWS resource that the
application role has access to.

![Blast Radius](https://serverlessfirst.com/img/blog-images/iam-blast-radius-ec2-vs-lambda.png "Blast Radius")

Serverless deployment frameworks encourage this best-practice by automatically
generating IAM roles for each function defined in the file. AWS SAM enables
this by default (see
[here](https://aws.amazon.com/premiumsupport/knowledge-center/lambda-sam-template-permissions/))
and the popular
[serverless-iam-roles-per-function](https://github.com/functionalone/serverless-iam-roles-per-function)
plugin does this for the Serverless Framework. In both cases, the developer
just needs to add a few lines to their YAML function definition with the IAM
policy statements to be included in the function’s role.

## Problems with this approach

While developer-specified per-function IAM roles are great, there are two
problems with relying solely on them to secure your Lambda application.

### Problem 1: IAM is hard and application developers aren’t IAM experts 🤷

Good developers care about the security of the features they’re building but
they also care about shipping them on time. This will often involve trade-offs
and IAM is notoriously difficult to get right. Finding the correct permissions
required for a particular API call is not an easy task amongst the expansive
AWS documentation and a time-pressured developer may be tempted to grant
unnecessarily broad permissions in order to get something working.

Similarly, less experienced developers may stumble upon internet articles or
Stack Overflow answers which suggest giving “Resource: *” permissions to get
past a specific error and take this at face value, without realising the
security impact of doing so.

[Getting IAM
right](https://www.effectiveiam.com/ch3-why-aws-iam-is-so-difficult-to-use) is
hard and these are examples of unnecessary privilege escalation which can
result in a security hole within your AWS account if left unbounded.

### Problem 2: Traditional organisational policy may disallow IAM role creation by application teams 🚫

Some larger organisations with a dedicated in-house cloud
platform/security/DevSecOps team—in particular those with limited experience of
operating fully serverless, Lambda-based applications—have a general policy of
not allowing application development teams to create their own IAM roles and
users. Given problem 1, this stance is understandable as the platform team will
be more skilled with IAM than the application development team and maintaining
security is their highest priority. They are the gatekeepers of any IAM roles
that get created and application teams need to submit requests for creation of
roles through them. [IAM roles for CI/CD
deployment](https://serverlessfirst.com/create-iam-deployer-roles-serverless-app/)
which the platform team typically install may not be given the permission to
create IAM roles for the application.

However, the big problem with this stance is that it doesn’t scale well for a
fast-moving serverless application development team. Having to go through a
cross-team human review of a new IAM role every time a new single-purpose
Lambda function is created (often several times a week) will get old very fast.

It is much less of an issue for traditional EC2-based application dev teams who
generally only need one application role created at the outset of development
and maybe the occasional statement addition every month or so.

A strict adherence to this policy by the platform team would require a single
IAM role to be shared across all Lambda functions in order to allow development
team to continue to move swiftly and independently. But then we have to
compromise on security as we lose all the benefits of the small blast radius
that Lambda gives us. We’re back to the EC2-sized blast radius. 🙁

## Permissions boundary to the rescue

An **IAM permissions boundary** allows us to get the best of both worlds:

 - Application team retains ownership of granular permissions in per-function roles and can ship independently 👍
 - Platform team can continue to enforce a maximum blast radius (equal to the
   `EC2Application` role) on the application, regardless of how developers specify their function policies 👍

A definition from the [AWS docs](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html):

> A permissions boundary is an advanced feature for using a managed policy to
> set the maximum permissions that an identity-based policy can grant to an IAM
> entity. An entity’s permissions boundary allows it to perform only the
> actions that are allowed by both its identity-based policies and its
> permissions boundaries.

Let’s look at how we can use a permission boundary to secure a Lambda
application. We’ll split it into two sections: 1) steps which the cloud
platform team (or whoever is responsible for [setting up your AWS landing
zone](https://serverlessfirst.com/services/launchpad/)) will implement; and 2)
steps which the application development team will implement.

### Cloud platform team implementation steps

 1. Create a custom IAM Managed Policy named ApplicationRuntimeBoundary in each
account environment where the application will run. This policy will define the
superset of runtime permissions that all Lambda functions in the application
will need. This policy statement would be provided up front to the platform
team by the application team. This will likely take a non-trivial amount of
time to prepare, but will be a one-off effort.

 2. When defining the [deploy-time IAM
role](https://serverlessfirst.com/create-iam-deployer-roles-serverless-app/)
`CloudFormationExecutionRole` (a role assumed by CloudFormation in the CI/CD
pipeline), add a `Condition` that specifies that CreateRole (and associated role
management actions) can only be performed on roles which have the
`ApplicationRuntimeBoundary` boundary attached to it.

```
# `CloudFormationExecutionRole` policies
Policies:
  # Only allow this deploy-time role to create/manage roles with the `${AppId}` prefix AND which have the permissions boundary managed policy attached
  - PolicyName: DeployBoundedIAMRoles
    PolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Action:
            - iam:CreateRole
            - iam:UpdateRole
            - iam:DeleteRole
            - iam:GetRole
            - iam:AttachRolePolicy
            - iam:DetachRolePolicy
            - iam:DeleteRolePolicy
            - iam:PutRolePolicy
            - iam:CreateServiceLinkedRole
            - iam:DeleteServiceLinkedRole
            - iam:PutRolePermissionsBoundary
            - iam:TagRole
            - iam:UntagRole
          Resource:
            - !Sub 'arn:aws:iam::${AWS::AccountId}:role/${AppId}-*'
          Condition:
            StringEquals:
              'iam:PermissionsBoundary': !Ref ApplicationRuntimeBoundary
```

### Application team implementation steps

Once the `ApplicationRuntimeBoundary` managed policy is in place, there is one
step remaining for the application team to complete. They need to update their
deployment framework source code where their functions are defined to instruct
the framework to attach this policy as a permission boundary to all IAM roles
it creates for each function.

With AWS SAM, this can be achieved by setting the `PermissionsBoundary` attribute
in the SAM template. See example [here](https://aws.amazon.com/premiumsupport/knowledge-center/lambda-sam-template-permissions/).

With the Serverless Framework, you can use the `PermissionsBoundary` attribute
of the [iam-roles-per-function
plugin](https://github.com/functionalone/serverless-iam-roles-per-function#permissionsboundary),
like so:

```
# serverless.yml
custom:
  serverless-iam-roles-per-function:
    iamGlobalPermissionsBoundary: !Sub 'arn:aws:iam::${AWS::AccountId}:policy/ApplicationRuntimeBoundary'
```

The diagram below shows how a boundary permission prevents a Lambda
function—whose IAM role policy has an extremely loose permission set granting
it full access to SSM Parameter Store—from accessing any parameter keys that do
not match the specified resource prefix path, and even then, it will still only
have read access to them.

![Permissions Boundary](https://serverlessfirst.com/img/blog-images/lambda-permission-boundary.png)

## Conclusion

Lambda-based cloud applications are by default more secure than traditional
EC2-based workloads. But the security models of both architectures differ a
fair bit and so both application developers and cloud ops engineers need to
adapt their existing practices and policies to keep up. Developers need to
learn how best to do least-privilege IAM and ops engineers learning about the
security benefits that serverless apps enable.

Use of an IAM permissions boundary for Lambda functions is an example of a
win-win for both parties. [This tweet from Andy
Carter](https://twitter.com/Hope4sun/status/1428706440605274118) illustrates
the significant benefits to cross-team productivity he has experienced with
allowing the application team to manage their own IAM roles:

> Yep it’s a great use case and in one customers case will reduce over 500+
> tickets a month (multiply out the time/costs on doing the work  as delays to
> the devs waiting on the change, it’s a big saving all round) hoping to
> automate the policy created into the pipeline for review
