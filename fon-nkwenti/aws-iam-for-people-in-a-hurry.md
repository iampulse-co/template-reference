# AWS IAM for People in a Hurry

## Author [Fon Nkwenti](https://fonnkwenti.hashnode.dev/aws-iam-for-people-in-a-hurry)

How to quickly set up a user account using the AWS Console

![People in a hurry](https://fonnkwenti.hashnode.dev/_next/image?url=https%3A%2F%2Fcdn.hashnode.com%2Fres%2Fhashnode%2Fimage%2Fupload%2Fv1629488054123%2Fen8BF7jMR.png%3Fw%3D1600%26h%3D840%26fit%3Dcrop%26crop%3Dentropy%26auto%3Dcompress%2Cformat%26format%3Dwebp&w=3840&q=75 "People in a Hurry")

## Introduction

IAM, which stands for Identity Access Manager, is an AWS service that allows
you to manage access to your compute, storage, database and application
services on AWS' Cloud. This is done by creating users, groups and roles with
the desired permissions to allow or deny access to your AWS resources. IAM is a
global service and is available free of charge.


## What we'll cover

 - What you can do with IAM
 - How you can set up a user with permissions
 - Limitations/caveats

## What can you do with the service?

You can specify permissions to control which users can access specific
services, the kind of actions they can perform and which resources are
available, ranging from VMS, DB instances and even the ability to filter DB
query results. You can determine which users have MFA access to specific Amazon
EC2 resources and perform specific actions on those resources, such as
restricting who can lunch an Amazon EC2 instance. In combination with
CloudTrail, you can keep track of all of the API calls made by the IAM users.

You can create users and assign them passwords and secret access keys.

You can create groups with similar access patterns, for example, the developer
team group. Each developer account would be assigned to the group and inherit
the same permissions set at the group level. You can integrate your existing
enterprise identity system, such as Microsoft active directory. This is done by
using standards-based federation technologies like SAML. It eliminates the need
for additional sets of credentials to manage your AWS resources.

You can use roles to grant other people permissions to resources in your AWS
account without sharing your password or secret access keys.

## How does a typical setup look like?

Let us go through a few steps to set up an administrator account that you would
use instead of your root account to manage your AWS compute, database, storage
and application services. To make things smooth, the administrator account will
have administrator privileges.

 1. First of all, you need to sign up for an AWS account. You can refer to How to set up a Free Tier AWS account to get you up and running.
 2. Search and click on IAM in the search bar on the AWS console to avoid scrolling through all the AWS services.

![AWS Management Console](https://cdn.hashnode.com/res/hashnode/image/upload/v1629488261290/fiHY98p9A.png?auto=compress,format&format=webp "IAM Dashboard")

 3. Click on Users on the left menu, then click on add user.

![IAM Dashboard](https://cdn.hashnode.com/res/hashnode/image/upload/v1629488362089/Zw0LzVV6E.png?auto=compress,format&format=webp "IAM Dashboard")

 4. Click on Add user

![Add IAM User](https://cdn.hashnode.com/res/hashnode/image/upload/v1629488384491/b_2ZGcV-C.png?auto=compress,format&format=webp "Add IAM User")

 5. Provide a name for the user and check AWS Management Console access.

![Set User Details](https://cdn.hashnode.com/res/hashnode/image/upload/v1629488408050/ri4kqD3pg.png?auto=compress,format&format=webp "Set User Details")

 6. Autogenerate password for the user and continue to permissions.

![Select AWS Access Type](https://cdn.hashnode.com/res/hashnode/image/upload/v1629488436192/OsulIvvNh.png?auto=compress,format&format=webp "Select AWS Access Type")

 7. Click on Attach existing policies directly and check the AdministratorAccess Policy.

![Set IAM Permissions](https://cdn.hashnode.com/res/hashnode/image/upload/v1629488472357/BYupCmnT4.png?auto=compress,format&format=webp "Set IAM Permissions")

This step is optional, but you can add an appropriate tag for the user.

![Add User Tags](https://cdn.hashnode.com/res/hashnode/image/upload/v1629488486147/L-jsn2Cbmg.png?auto=compress,format&format=webp "Add User Tags")

Review the configurations and click on **Create user**.

![Add User Review](https://cdn.hashnode.com/res/hashnode/image/upload/v1629488512182/opkNkWknP.png?auto=compress,format&format=webp "Add User Review")

![Permissions Summary](https://cdn.hashnode.com/res/hashnode/image/upload/v1629488538968/oFAovVMqe.png?auto=compress,format&format=webp "Permissions Summary")

 8. Copy the sign-in link and the password which you would use to log in. You may also have the information sent to the user's email or download the .csv file with the information.

![Add User - Success](https://cdn.hashnode.com/res/hashnode/image/upload/v1629488563915/iVI1WaMEl.png?auto=compress,format&format=webp "Add User Success")

The contents of the .csv file are;

![CSV file contents](https://cdn.hashnode.com/res/hashnode/image/upload/v1629488788794/LGvD7y5hB.png?auto=compress,format&format=webp "CSV file contents")

 9. On the sign-in page, enter the username and auto-generated password.

![AWS IAM Sign In](https://cdn.hashnode.com/res/hashnode/image/upload/v1629488608769/geIkeEICh.png?auto=compress,format&format=webp "AWS IAM Sign In")

 10. The user would be prompted to create and confirm a new password.

![Change IAM Account Password](https://cdn.hashnode.com/res/hashnode/image/upload/v1629488623527/IKLGxibH7.png?auto=compress,format&format=webp "Change IAM Account Password")

Once that is completed, the user would land on the console. Pay attention to the top right corner, which indicates which account is logged into the console.

![AWS Management Console](https://cdn.hashnode.com/res/hashnode/image/upload/v1629488641347/efk0ulMCo.png?auto=compress,format&format=webp "AWS Management Console")

## Limitations/Caveats

You are limited to 1000 IAM roles, but this can be increased with a support
request to AWS alongside your use case. AWS account ID aliases must be unique
across AWS products in your account. A user can be assigned a maximum of 2
access keys.

## Conclusion

I know you are in a hurry so we must leave it at this for now. As usual, you
can find more information by clicking on the links in the resources section
below. Feel free to follow up with me in the comments section or on
[Twitter](https://twitter.com/fon_nkwenti). Hope this has been very informative
to you. Have a good one!
