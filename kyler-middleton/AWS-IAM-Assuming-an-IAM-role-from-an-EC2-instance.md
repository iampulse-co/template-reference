# AWS IAM: Assuming an IAM role from an EC2 instance

## Author: [Kyler Middleton](https://www.kylermiddleton.com/2019/08/aws-iam-assuming-iam-role-from-ec2.html)

> tl;dr: A batch script (code provided) to assume an IAM role from an ec2
> instance. Also provided is terraform code to build the IAM roles with proper
> linked permissions, which can be tricky. 

I'm working through an interesting problem - syncing Azure DevOps to AWS, and
making the connection functional, scalable, and simple. Sometimes, when
designing anything, a path is followed that doesn't pan out. This is one of
those paths, and I wanted to share some lessons learned and code that might
help you if this path is a winner for you.

Our security model for EC2 requires that a machine assume a higher IAM policy
when it is required, but the rest of the time it have much lower permissions.
That's a common use case, and a best practice.

Some applications support assuming a higher IAM role natively - I later
learned, after pursuing this, that terraform is one of those applications (more
details on that in a future blog). However, some applications can't, and
require you to do the heavy lifting yourself.

## IAM - a Sordid (and Ongoing) History

![IAM A What?](https://1.bp.blogspot.com/-_oP7Ur9iqmY/XWM1aPrgThI/AAAAAAAAxnY/E-_EXp8lQckmjsfhdkxq1sQcWgbtEe0OQCLcBGAs/s400/iam.png "IAM A What?")

IAM (Identity and Access Management) is complex beast that controls
authentication (who are you?) and authorization (what are you allowed to do?).
Because even simple complex can be made complex with enough work, IAM supports
recursive role assumptions, so a resource that starts with 1 set of credentials
can assume a different (or more expensive) set of credentials during certain
actions.

This has the benefit of being very flexible, and the detriment of allowing
deployments so complex it can require a serious amount of nancy drew-ing to
sort out what permissions something "really" has. 

This complexity has led to a series of high profile security vulnerabilities
introduced by a lack of understanding or a too-complex deployment in some of
what are generally thought to be the most security companies. The most recent
high profile one was Capital One's hack by an ex-AWS employee. The ec2 IAM
policies were written in such a way as to provide access to all s3 buckets, so
once a single ec2 instance was compromised, all data everywhere was
compromised. KrebsOnSecurity has a great [write-up of the
incident](https://krebsonsecurity.com/2019/08/what-we-can-learn-from-the-capital-one-hack/).

## Definitions - Policies, Roles, and Trust Relationships, Oh My

So clearly, lack of understanding here can be a vulnerability all in itself, so
let's break down what pieces comprise IAM. 

 - **Policies**: Policies are a list of permissions that can be granted. They are not allowed to be assigned to resources themselves (to my knowledge). Rather, they are assigned to one or more roles, and the roles are assigned to or assumed by resources.
 - **Roles**: An IAM role is a bucket of permissions. The permissions it contains are not "within" the role, but rather are described in the IAM policies that are assigned to the role. These roles can be assigned to a resource (think ec2 resources being assigned a single ec2 role) or assumed by a resource or process. 
 - **Instance Profile**: An IAM Instance Profile is a somewhat hidden feature of IAM roles. Instance Profiles are assigned 1:1 to an IAM Role, and when assigned, allow an ec2 instance to be assigned the role. To be even simpler: This stand-alone resource acts as a check box for an IAM role on whether it can be assumed by an ec2 instance or not. 
   - Interesting tip: I say this this resource type is somewhat hidden because when an IAM role is created in the GUI, an Instance Profile is automatically created and assigned. However, if you're building an IAM Role via command line or API call (thing Terraform or CloudFormation), this resource isn't automatically created, and instead acts as a "gotcha". 
 - Trust Relationship: An IAM Trust Relationship is a special policy attached to an IAM Role that controls who can assume the role. This is a key part of our IAM role assuming, and we'll walk through the different policies required on the implicit (assigned) IAM role for the ec2 instance vs the IAM role assumed by the instance.

## The Implicit IAM Role

We'll build several IAM roles, with associated policies and trust
relationships. First, let's build the Implicit IAM role. This role will be
assigned directly to the ec2 instance, and is static.

Note that this role has an embedded IAM policy - this is our trust policy that
permits the ec2 instance service to assume this role - this is required if any
ec2 instance will be assigned the role.  

```
resource "aws_iam_role" "ado_iam_implicit_role" {
  name = "AzureDevOpsImplicitIamRole"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

  lifecycle {
    prevent_destroy = true
  }
}
```

Next we'll create an IAM policy for this implicit role. The only permission we
want this policy to contain is the ability to use the STS service to assume a
specific IAM role. Otherwise, this ec2 instance should act as a normal virtual
machine, and not be able to edit or control the AWS environment around it.

```
# Create IAM policy to give implicit role permission to assume broad IAM Role
resource "aws_iam_policy" "ado_iam_role_permit_sts_assume" {
  name = "AzureDevOpsPolicyPermitStsAssume"

  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "${aws_iam_role.ado_iam_assumed_role.arn}"
    }
  ]
}
EOF

  lifecycle {
    prevent_destroy = true
  }
}
```

Then we link the two together - remember that roles and policies are not linked
by default, and have to be assigned together.

```
resource "aws_iam_role_policy_attachment" "ado_iam_attach_implicit_role_to_sts_assume_policy" {
  role       = "${aws_iam_role.ado_iam_implicit_role.name}"
  policy_arn = "${aws_iam_policy.ado_iam_role_permit_sts_assume.arn}"
  lifecycle {
    prevent_destroy = true
  }
}
```

And remember this implicit IAM role needs to be statically assigned to an ec2
instance, and that requires it to have an instance profile, so let's build that
and assign to the IAM role. 

```
resource "aws_iam_instance_profile" "ado_iam_role_implicit_instance_profile" {
  name = "AzureDevOpsImplicitIamRole"
  role = "${aws_iam_role.ado_iam_implicit_role.name}"
```

![Permissions
Policies](https://1.bp.blogspot.com/-sMtBfOMlRuU/XWGCkYp85eI/AAAAAAAAxmE/TEKIGSeDqdcMnaMEwG2TZjovDlGkqEP8QCLcBGAs/s1600/Screen%2BShot%2B2019-08-24%2Bat%2B12.30.22%2BPM.png
"Permissions Policies")

And here's the trust policy under the "trust relationships" tab. You should see
the ec2 service is trusted by this policy to be assumed.

![Trust
Relationships](https://1.bp.blogspot.com/-JJOXun9P_Ck/XWGEtmsLf1I/AAAAAAAAxmQ/Knh9XkjGrxYd7iMA8aySepUThJhmahEbQCLcBGAs/s400/Screen%2BShot%2B2019-08-24%2Bat%2B12.40.38%2BPM.png "Trust Relationships")

Now that we have an IAM role with a policy and a trust relationship to the ec2
service (and that gotcha of an instance profile), let's go assign it to an ec2
instance. I didn't include terraform code for this, so you'll build an ec2
instance by hand. Once ready, go into the instance settings, and click
"Attach/Replace IAM Role".

![Replace IAM
Role](https://1.bp.blogspot.com/-tYd45KAWYYc/XWGFcPPKESI/AAAAAAAAxmc/UyIftBjyQCgNEogpMYXfaBGaW0CZWpcawCLcBGAs/s640/Screen%2BShot%2B2019-08-24%2Bat%2B12.42.05%2BPM.png "Replace IAM Role")

Find the IAM role you want to associate with the ec2 instance (the implicit one
we just built). If you don't see it, try the refresh icon next to the list, or
go check to make sure the instance profile is built and associated with the IAM
role properly. 

![IAM
Role](https://1.bp.blogspot.com/-cVIZnu0gSH0/XWGFb-c2sJI/AAAAAAAAxmY/2gNF4LAv4-cgHNt4Xtie5Kd7rjAjpT-_ACLcBGAs/s640/Screen%2BShot%2B2019-08-24%2Bat%2B12.41.55%2BPM.png "IAM Role")

Great, now we have an IAM role, assigned to an ec2 instance, that permits it to
assume a higher permissions role. Which is all well and good, but we haven't
built that higher permissions role yet, so let's do that.

### More Permissions, Give Me More!

The whole point of this exercise is for the ec2 instance to be able to assume a
set of more expansive permissions when it needs it, so we need to build a
distinct IAM role to contain those permissions, a policy to describe what
permissions we want to grant, and a trust relationship that allows the implicit
(statically assigned) ec2 instance to assume the higher permissioned role.

First let's build the IAM role. The role parts are exactly the same, but notice
the embedded IAM policy (the trust relationship) is entirely different. Rather
than trusting the ec2 service to assume it, it's trusting the first IAM policy
only. This assures that only a single specific IAM role can assume this upper
IAM role. And the lower role is assigned only to a single ec2 (or more if you
want) instance, creating a limited chain of permissions that is very flexible
to assign.

```
resource "aws_iam_role" "ado_iam_assumed_role" {
  name = "AzureDevOpsAssumedIamRole"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "${aws_iam_role.ado_iam_implicit_role.arn}"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

  lifecycle {
    prevent_destroy = true
  }
}
```

Now, let's build an IAM policy of permissions for this expansive role. The
example here permits all actions to all services, which is NOT AT ALL a best
practice. If at all possible make sure to limit your expansive IAM policies to
much more specific actions to specific resources. The policy here should rarely
be used.

```
resource "aws_iam_policy" "ado_iam_assumed_policy" {
  name = "AzureDevOpsAssumedIamPolicy"

  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAllPermissions",
      "Effect": "Allow",
      "Action": [
        "*"
      ],
      "Resource": "*"
    }
  ]
}
EOF
  lifecycle {
    prevent_destroy = true
  }
}
```

And you can probably guess what comes next - we need to link the IAM role to
the IAM policy that we just built, which looks like this:

```
resource "aws_iam_role_policy_attachment" "ado_iam_attach_assumed_role_to_permissions_policy" {
  role       = "${aws_iam_role.ado_iam_assumed_role.name}"
  policy_arn = "${aws_iam_policy.ado_iam_assumed_policy.arn}"
  lifecycle {
    prevent_destroy = true
  }
}
```

When you look at the new role in the AWS console, it'll look like this:

![IAM Assumed
Role](https://1.bp.blogspot.com/-7CoX4-W0ALM/XWGIiev5AzI/AAAAAAAAxms/woPNF6TcWH8BlKHfAfOol2nz_tIKZzNnwCLcBGAs/s640/Screen%2BShot%2B2019-08-24%2Bat%2B12.56.09%2BPM.png "IAM Assumed Role")

The trust relationship tab will look like this:

![Edit Trust
Relationships](https://1.bp.blogspot.com/-_82wnjZS9Nk/XWGInWTd-wI/AAAAAAAAxmw/hMcobW0mGScqHrjXgLj6Zx6KUQlJG349QCLcBGAs/s640/Screen%2BShot%2B2019-08-24%2Bat%2B12.56.21%2BPM.png "Edit Trust Relationships")

### Let's Assume The Role, For Real Now

Now that everything is in place, we're ready to go onto the ec2 instance and
assume the role. This involves running a batch script, which will do several
things - clearing the variables in case of a last run hanging around, figuring
out the account ID by calling the AWS ec2 metadata service, figuring out the
instance ID, and setting the information to a text file where bash can call it
and set the global env variables.

```
# Clean variables
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN
unset ACCOUNT_ID

# Call for ACCOUNT_ID
curl -s http://169.254.169.254/latest/meta-data/identity-credentials/ec2/info | jq -r '.AccountId' | awk '{print "export", "ACCOUNT_ID="$0}' > variables

# Call for INSTANCE_ID
curl -s http://169.254.169.254/latest/meta-data/instance-id | awk '{print "export", "INSTANCE_ID="$0}' >> variables

# Read account ID variables into global
. ./variables
```

Then we start the cool stuff. AWS ec2 linux AMIs already contain the AWS CLI
toolset. If you don't have it, install it for this to work.

First we use the AWS CLI to assume our role, depending on both the dynamic info
we gathered earlier - the account number and the EC2 ID. These dynamic pieces
permit this same script to be run in any account, and to set an IAM session
name that is globally identifiable to this instance, for later CloudTrail-ing.

Then we use jq (javascript query) to export the pieces we need to a file, then
we call bash to read the file and set variables into the bash shell
environment. Then we cleanup by removing the STS creds from the disk.

```
# Assume role
aws sts assume-role --role-arn arn:aws:iam::$ACCOUNT_ID:role/AzureDevOpsAssumedIamRole --role-session-name "BuilderHostname=$INSTANCE_ID" > mysession.json

# Extract values from session, write to disk
jq -r '.Credentials.AccessKeyId' mysession.json | awk '{print "export", "AWS_ACCESS_KEY_ID="$0}' > variables
jq -r '.Credentials.SecretAccessKey' mysession.json | awk '{print "export", "AWS_SECRET_ACCESS_KEY="$0}' >> variables
jq -r '.Credentials.SessionToken' mysession.json | awk '{print "export", "AWS_SESSION_TOKEN="$0}' >> variables

# Read variables into global
. ./variables

# Delete variables file on disk
rm variables
```

Boom, your ec2 instance has now assumed a higher IAM role that the assigned
one, and can do all sorts of stuff.

### Wrap It All Up

The collected code for all these examples can be found here:
[https://github.com/KyMidd/AWS_EC2_IAM_Authentication](https://github.com/KyMidd/AWS_EC2_IAM_Authentication)

I'll continue to investigate how to use IAM roles in order to build a
comprehensive terraform and Azure DevOps CI/CD, so these types of posts will
continue. In the next one, I'll cover how Terraform can handle most of these
items itself, so the bash script is not needed.

However, I hope this script and the coverage of IAM helps you in your
non-Terraform requirements. Thanks all!

Good luck out there.
kyler
