# AWS IAM:PassRole explained

Understanding the iam:PassRole permission is key to not only getting your
applications working in AWS, but also doing it securely.

## Author: [Rowan Udell](https://blog.rowanudell.com/iam-passrole-explained/)

![Hard Hats](https://blog.rowanudell.com/content/images/size/w1600/2020/10/helmets.png "Hard Hats")

Photo by [Silvia Brazzoduro](https://unsplash.com/@scdisilviabrazzoduro?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

A common point of confusion when getting started with AWS IAM, and when trying to implement "least privileges" on IAM is the message "is not authorized to perform: iam:PassRole on resource". Usually this refers to "User" or "CloudFormation" as the culprit. While the error tells you that iam:PassRole is involved, it doesn't really hint at the how or why of the problem at hand. Hopefully this explanation and diagrams can help you.

There's no doubt AWS IAM is great at its job. Unfortunately you sometime get exposed to some complicated situations right out of the gate when you start using it, which doesn't make it easy to learn. Understanding the iam:PassRole permission is key to not only getting your applications working in AWS, but also doing it securely.

## IAM PassRole

Confusion with IAM PassRole is not that unusual, and a quick [search on SO](https://stackoverflow.com/search?q=iam+passrole) will show you many other people suffering the same problem. For better or worse, most of the explanations are copied from the [official documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_passrole.html) or [this old blog post about EC2](https://aws.amazon.com/blogs/security/granting-permission-to-launch-ec2-instances-with-iam-roles-passrole-permission/), and neither of those do a great job of explaining why you need to jump through the IAM PassRole hoop to get things working

The problem is, **IAM PassRole is not an action**. Even though you can see it sitting there in the Action list of your policies, ~~it's a lie~~ it's not an action.

Almost all the other actions you will see in all your policies map nicely to actions you will take on AWS e.g. `ec2:CreateInstances` to create some instances, `lambda:InvokeFunction` to trigger a function, etc. `iam:PassRole` is not one of those, which causes confusion when the error message demands that you authorise it. At least now the [Actions, resources, and conditions page for IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/list_identityandaccessmanagement.html) lists it with [permission only] next to it, but that's a relatively recent improvement and a subtle one at that.

## But Why PassRole, Why?

The reason `iam:PassRole` is required is not to get your scenario working, it's to prevent a specific, but very realistic, escalation of privilege attack based on the [confused deputy problem](https://en.wikipedia.org/wiki/Confused_deputy_problem). Unfortunately there isn't a lot of reference to this in the starting documentation or many of the AWS blog posts out there, so you're left to figure this one out on your own.

If you're just bashing around AWS with `AdministratorAccess` attached to your role like most people are in their development environment, you won't notice any of this since you've implicit granting of `iam:PassRole` to * resources/roles.

When you go to do this in an environment where you don't have full access is usually when you experience it. The PassRole permission is an important layer of security to protect your AWS environment from unintended and unwanted activities and escalation of privileges.

## The Happy Path


Giving AWS services authorisation to do things in your account is a completely normal thing to do. While this might sound a little scary at first, it's actually a good thing! Configuring AWS services to do things for you means less work that you have to do.

In this case, a user (who has `AdministratorAccess0` attaches a role to their instance so that it can be used to make calls to the AWS APIs.

![The Happy Path](https://blog.rowanudell.com/content/images/2020/10/iam-passrole-explained-happy-path.png "The Happy Path")

## The Sad Path

Where this goes wrong, is when the AWS service is told to do something that you probably didn't intend.  Without any additional layer of permissions, it can be ordered to do something it can do, but **should not do**.

In this scenario EC2 has been configured by a user (who only has the made-up `LimitedAccess` policy, which includes the ability to create EC2 instances BUT is not as powerful as `AdministratorAccess`) to give the instance it creates a role higher level of access to your environment than the original requestor had (in this case, the highly desirable `AdministratorAccess`).

![Without Passrole](https://blog.rowanudell.com/content/images/2020/10/iam-passrole-explained-sad-path.png "Without Passrole")

## The Solution

The PassRole permission (not action, even though it's in the Action block!) is the additional layer of checking required to secure this.

By giving a role or user the iam:PassRole permission, you are is saying "this entity (principal) is allowed to assign AWS roles to resources and services in this account".

You can limit which roles a user or service can pass to others by specifying the role ARN(s) in the Resource field of the policy that grants them iam:PassRole:


```
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:::123456789012:role/LimitedAccess"
}
```

This diagram shows the previous scenario, but with the policy statement above included to provide an additional layer of checking with PassRole, and limit it to the role(s) specified the Resource block.

![The Solution](https://blog.rowanudell.com/content/images/2020/10/iam-passrole-explained-the-solution.png "The Solution")

### Specific AWS Services

You can also limit roles to be used by specific AWS services as another level of security you can apply, which is always a good idea. [This page in the IAM User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_iam-passrole-service.html) has an example of the policy you should use to limit `iam:PassRole` to a specific AWS service, but keep in mind it's allowing access to ALL your roles in its current form. Unfortunately at the time of writing, not all services you'd expect are supported e.g. Lambda (for normal functions). Check out [the list of services that support service-linked roles in the documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-services-that-work-with-iam.html).

Hopefully this helps you understand the "_is not authorized to perform: iam:PassRole on resource_" error message. If you're still unsure, [get in touch with me](mailto:rowan@rowanudell.com) and let me know so I can improve on this explanation.
