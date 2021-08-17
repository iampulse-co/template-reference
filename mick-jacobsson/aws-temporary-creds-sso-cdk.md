# AWS temporary creds with SSO and a CDK workaround

## Author: [Mick Jacobsson](https://www.talkncloud.com/aws-temporary-creds-sso-cdk/)

- [X] AWS

It's been a rainy past couple of days here so I'm catching up on some writing,
something that i've been meaning to talk about for sometime is how to manage
temporary credentials and access. The aws cli is a great tool, we all use it
and many of the AWS articles require setting up your credentials or access
keys. The main problem with AWS credentials is that you set them in IAM and
they live **forever**. It's like taking your password and putting it into a plain
text file, convenient but not a great idea.

So, what happens when you set a new account with AWS, start using an account
which probably has **full access** and then you save the access keys in your
local file on your computer? There are a couple of things to consider, the bad
guys want to steal those credentials to access your account, this might be to
run EC2 servers to [mine
crypto](https://www.zdnet.com/article/crypto-mining-worm-steal-aws-credentials/).
Someone might accidentally share credentials or check them into a code
repository, again, giving people access to your account.

## Great, now i'm worried, what can i do?

Well, this is actually a pretty simple fix and won't take too long with the use
of AWS single sign-on. Oh and it's free.

You can't start without organizations so let's jump in...

## AWS organizations

I'm not going to go into great detail about [AWS
organizations](https://aws.amazon.com/organizations/), just know that they are
needed to enable SSO. Orgs are basically a way to manage multiple AWS accounts,
people want many accounts for lots of different reasons. A common one is to
separate environments like development and production which also happens to
follow best practice, you can also start to setup policies across these
multiple accounts to better manage permissions and access. Use the AWS
management console and head over to AWS organizations and set up the basics.

## Fun begins with SSO

When you setup an AWS account you create users, these users are part of IAM
which is basically a standalone user directory in AWS. When you enable SSO it
comes with it's own directory, these users are totally separate from IAM users.
You can also choose to federate with your existing directory like active
directory, if you're familiar with SAML it's the same process you've used
before and if you need this you're probably a larger enterprise.

![Change Identity Source](https://www.talkncloud.com/content/images/2021/02/Screen-Shot-2021-02-20-at-8.34.38-pm.png "Change Identity Source")

We'll be using the SSO built-in user directory, it's straight forward to use
and great if you don't have a directory already and don't need AD/third-party.
You can change this later. One thing you'll want to work out first is which
region you want to configure SSO in, it's not global and you can only have one.
Things to consider might be where your users are located, if you need to have
users in a region for data protection or security requirements and if your
federating with something like AD where does that live. Adjust your region in
the management console (top-right) and navigate to SSO.

## Enabling SSO

When you reach the SSO service you'll have a simple option to enable SSO, this
will take a few seconds and when complete you've activated SSO for your
organization.

![Enable AWS SSO](https://www.talkncloud.com/content/images/2021/02/Screen-Shot-2021-02-20-at-5.46.12-pm.png "Enable AWS SSO")

*Note: If you've already enabled SSO and have changed regions it will let you
know that you'll need to remove the other one if you want to enable SSO for
this region.*

You should be greeted with something like this once completed:

![Welcome to AWS Single Sign-On](https://www.talkncloud.com/content/images/2021/02/Screen-Shot-2021-02-20-at-5.13.12-pm.png "Welcome to AWS Single Sign-On")

## Adding an SSO group

I'd recommend adding a group first before a user, this way you can simply add
the new users to the group. I did notice a small bug when you create a user
first without groups and then follow the user creation wizard to create a
group. It attempts to create more than one group. I've let AWS know about this
bug.

Adding a group is simply that, it's a logical group and nothing else, you have
a few options to create a group.

![Create Group](https://www.talkncloud.com/content/images/2021/02/Screen-Shot-2021-02-21-at-12.47.10-pm.png "Create Group")

After you've created a group you can add permission sets to a group. The way
you do this is a little confusing. Click on **AWS Accounts** in the SSO console,
you should all of your AWS accounts in your organizations, select the accounts
you want to work with and click **assign users**. Now you can select users or
groups, we want to use groups so select groups and select your newly created
group.

Now you can assign **permission sets**, this is empty be default, you'll get two
options:

 1. Use AWS pre-defined policies based on job functions
 2. Custom, create your own

I really like how AWS provide common job function policies, for most people
this will be fine. You can always add additional permissions later if you find
something missing.

![Create New Permission Set](https://www.talkncloud.com/content/images/2021/02/Screen-Shot-2021-02-20-at-5.15.23-pm.png "Create New Permission Set")

> As always, apply the principle of least privilege. Select the option that
> matches what you're trying to achieve without granting additional access that
> isn't required.

Finish off the wizard, now you have a group with permissions for the accounts
in SSO.

## Multi-factor authentication (MFA) is a good idea

I recommend using MFA, it's easy to use and provides additional protection to
protect your accounts. The cool thing about AWS is that they allow two levels
of MFA:

 1. Contextual
 2. Always

For option 1, you'll only be prompted if something has changed like your
network, location or other behaviour. For option 2, you'll be prompted for MFA
everytime.

You'll find SSO high level configuration options in the settings area.

![MFA Settings](https://www.talkncloud.com/content/images/2021/02/Screen-Shot-2021-02-20-at-5.23.41-pm.png "MFA Settings")

The other option here is the how you want to handle existing users, this
defaults to 'Allow them to sign in', but you can force your users to register,
block etc.

## Who doesn't like a custom sub domain?

This isn't really needed but I suggest changing this now so that when you go
through the login process and configure the cli you can bookmark etc with your
custom url.

This new subdomain will be your portal for users to use both the cli and the
web based management console. You can't change this once it's been configured
so you don't want to commit to this now, skip this step.

This can be found in the settings area, and it's a simple as deciding on what
name to chose...oh the possibilities...

## OK, now we can create a user

This should be nice and simple because we've done the hard work. From the SSO
console simply select add user and select the group we created earlier.

![Add User](https://www.talkncloud.com/content/images/2021/02/Screen-Shot-2021-02-21-at-12.44.36-pm.png "Add User")

The user will receive an email asking them to sign-in, register MFA.

![Register MFA Device](https://www.talkncloud.com/content/images/2021/02/Screen-Shot-2021-02-20-at-6.47.13-pm.png "Register MFA Device")

## AWS SSO and the AWS CLI

If you're an og and haven't been updating your AWS cli you'll need to download
the latest version. Version 2 is required for SSO, navigate to the AWS CLI page
to download and install the right one for you.

Here is the link for MacOS:

[https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-mac.html](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-mac.html)

## aws configure sso

Now, it's just a matter of configuring the cli:

```
aws configure sso
```

You'll be prompted for:

 - SSO start URL
   - Remember this was the custom url / sub domain from earlier.
 - The region
   - Enter the region SSO resides.

Your default web browser will automatically open and run through verification
and require a sign-in. If successful you should see the following:

![AWS Single Sign-On](https://www.talkncloud.com/content/images/2021/02/Screen-Shot-2021-02-20-at-5.30.51-pm.png "AWS Single Sign-On")

![Sign-On Success](https://www.talkncloud.com/content/images/2021/02/Screen-Shot-2021-02-20-at-7.06.10-pm.png "Sign-On Success")

If you get the following, it's more than likely a region issue because the
tokens are in another region. Go back and double check SSO and your cli are
using the same region:

![Invalid Grant](https://www.talkncloud.com/content/images/2021/02/Screen-Shot-2021-02-20-at-5.32.01-pm.png "Invalid Grant")

What happens here is that temporary credentials are issued to the cli tool that
last for 1 hour. This resides in the ~/.aws/sso/cache file for those wanting to
poke around. If you've been using the version 1 aws cli you might be familiar
with the ~/.aws/credentials file. If you have keys in that file, go ahead and
remove them as you should be using the SSO config from now on.

> If you are migrating to this way of logging in, don't forget to revoke your
> access keys from IAM as they are no longer needed.

Job done?

## Ah, yes, CDK is not SSO aware...yet

If you're a CDK head like myself you may have noticed that removing your
credentials from the AWS file and trying your usual tricks resulted in an
error. That is because it does not support SSO just yet, a more detail overview
of the situation can be found here:

[https://github.com/aws/aws-cdk/issues/5455](https://github.com/aws/aws-cdk/issues/5455)

It's pretty disappointing, I encourage you to jump on and upvote the feature
request so that the team can prioritize it.

Now, the github thread as a bunch of useful tools to workaround this problem
some are very feature rich and will do the job quite easily. I have used yawsso
which simply copies the configuration from the temporary file and updates the
credential file. It's easy to use.

[https://github.com/victorskl/yawsso](https://github.com/victorskl/yawsso)

## How does this work again

I'll recap the process of using the AWS cli so that it's easy to follow, you'll
need to just practice a few times to get the hang of it. All of these are from
the terminal.

```
aws sso login --profile my-profile
```

A browser window should pop up asking you to authenticate:

![Sign In](https://www.talkncloud.com/content/images/2021/02/Screen-Shot-2021-02-21-at-1.25.33-pm.png "Sign In")

**Note: this is the new user we setup earlier.**

![Successful Sign-In](https://www.talkncloud.com/content/images/2021/02/Screen-Shot-2021-02-20-at-7.06.10-pm-1.png "Successful Sign-In")

Now, your usual aws cli related commands will work as expected, to use version
1 style aws cli tools like CDK, simply run:

```
yawsso
```

This will update with the latest credential and you're back to normal with the
benefit of short lived credentials and better security.

## Final thoughts

So, that's a wrap on SSO and securing your keys with AWS SSO. It does include a
few extra services like organizations and SSO but the end result is well worth
it. The configuration of organizations and SSO takes about 30 minutes and
really sets up your accounts to built on as you pick up speed and more
knowledge around these new services.

The use of temporary credentials is a far better approach than long lived
credentials and hopefully seeing the job-function policies will give you some
more to take away around using read-only be default and potentially switching
into other roles and/or accounts if you need to make changes.

And again, all of this is free, not bad AWS, not bad.
