## Author [Stephen Kuenzli](https://www.effectiveiam.com/why-aws-iam-is-so-hard-to-use)

AWS Identity and Access Management system is a feat of modern engineering, correctly and quickly enforcing access controls at massive scale. The AWS security policy language is very powerful and flexible.   The access evaluation engine is ubiquitously integrated, highly available, performant, and quickly consistent &mdash; processing [hundreds of millions of access requests per second][1].  Engineers can use AWS security services to authenticate and authorize access to all infrastructure and even applications.   But interviews with many Cloud, DevOps, and Security engineers revealed problems with AWS IAM.

Engineers say it's hard to:
* Write policies that do what they intend
* Understand what all the policies actually do
* Validate access controls without breaking things

Even experts find it difficult to create policies that do what they _intend_.  My research found most practitioners feel like understanding who has access to their data is impossible.

The _**usability**_ of AWS' powerful security policy engine leaves much to be desired.  The problem is not that AWS IAM doesn't have enough features.  The opposite is closer to the truth.  There are many features, features interact in ways that are difficult to understand, and feedback on the correctness of a policy is slow and inadequate.  

This chapter explains why AWS security policy engineering is so hard, and offers solutions to those usability problems.  The 50+ engineers I interviewed really _want_ to create good policies.  They fear letting down customers when they don't.  However, because policy development and testing is difficult, engineers run out of time, energy, and patience to do it well without blocking delivery.

We'll examine the design of AWS IAM and uncover problems that make it difficult for engineers and teams to use it correctly.  

First, you'll learn IAM's complicated process for evaluating security policies to determine whether an API action should be allowed or denied.

Second, you'll learn how the flexibility of the AWS security policy language makes it difficult to identify what is in scope and how policies interact.

Then we'll examine AWS IAM's usability through the lens of design and safety engineering to identify the problems that make it difficult for engineers and teams to use it correctly. 

Finally, we'll discuss how to address these design problems with components that wrap IAM's power in a usable package.

## How AWS Security policies are evaluated
Let's uncover why evaluating the _effects_ of AWS security policies is hard.

This flowchart depicts AWS’ policy evaluation logic and is reproduced from the IAM docs:

[//]: # (Do not change the following lines as it may break the rendering of your objects)
```pulse-diagram
{"_key":"52edf2be-7132-432f-b1dc-bcece8b0d6dc"}
```

**Figure 3.1 AWS Security Policy Evaluation Logic**

Each of the five types of AWS security policy are integrated into the access decision making process. This is not simple to understand or evaluate.

Did you notice there are two paths for allowing access when a service supports resource policies? Look for paths to the green end state.

Both a Resource policy attached to a bucket or an Identity policy attached to an IAM principal may grant access to an S3 bucket. If either the Resource or Identity policy allows access to the bucket, the IAM principal is granted access.

More than 20 AWS services support resource policies, primarily those where it is useful to share a resource across accounts.

AWS IAM evaluates _all_ the policies in scope for the account, principal, session, and resource.  Any of those policies might `Deny` access and two kinds of policy may `Allow` access.

To understand effective access, engineers must evaluate them too.

Merely gathering a principal’s in-scope Identity policies is complicated.

Engineers need to account for _all_ Identity policies managing an IAM principal's access.  This might include:
* AWS Managed Policy attached directly to the IAM role, user, or a group the user belongs to
* Customer Managed Policy attached directly to the IAM role, user, or a group the user belongs to
* Inline policy attached directly to an IAM role or user, or a group the user belongs to

IAM supports attaching up to 10 policies to a role or group by default; AWS support can raise that limit to 20.  An IAM user can be a member of up to 10 IAM groups.  See [IAM Limits][2] for full details.

Depending on where you define policies, engineers may have to account for many policies, defined in many places.

And while convenient to grant a lot of access quickly, AWS Managed Policies:
* Do not limit resource scope.  Actions in the policy apply to all resources in the account.  So when the `ReadOnlyAccess` policy is applied to an IAM principal, it covers all resources and data in the account.  Every S3 bucket, DynamoDB table, EBS volume, etc.
* Change over time.  For example, the `ReadOnlyAccess` policy updates to grant read access to new services soon after launch.  Even service and job-specific policies see updates.  (💡 follow the MAMIP [twitter account][3] by Victor Grenu for updates)

⚠️ This policy evaluation logic also doesn’t account for service-specific access control systems such as S3’s Object ACLs. Of course it is still the engineer’s responsibility to understand how these work together.


## Identify what's in scope
A security policy can bring a lot or a little into scope.  

Chapter 2 described the form and operation of AWS security policy and statement elements.  The `Principal`, `Action`, and `Resource` elements may include a very wide or very narrow scope, particularly when using wildcards.

Here are some examples :

<table>
<thead>
<tr>
  <th>Element</th>
  <th>Narrow</th>
  <th>Medium</th>
  <th>Wide</th>
</tr>
</thead>
<tbody>

<tr>
  <td>Principal</td>
  <td>Fully-qualified IAM role ARN:
  <br/><br/>
  <code>arn:aws:iam::111: ↲</code><br/><code>&nbsp;&nbsp;role/app</code></td>
  <td>N/A</td>
  <td>Another AWS Account: <code>123456789012</code> or All AWS accounts <code>*</code></td>
</tr>

<tr>
  <td>Action</td>
  <td>Specific actions:
  <br/><br/>
  <code>["s3:GetObject",</code><br/>
  <code>"s3:GetObjectAcl", "s3:PutObject"]</code></td>
  <td>All S3 actions starting with <code>GetObject</code>:
  <br/><br/><code>"s3:GetObject*"</code></td>
  <td>Full access to S3:
  <br/><br/>
  <code>s3:*</code>
  <br/><br/>
  or full access to AWS APIs:&nbsp;<code>*:*</code></td>
</tr>

<tr>
  <td>Resource</td>
  <td>Specific bucket and object:
  <br/><br/>
  <code>["arn:aws:s3:::app-data",
        "arn:aws:s3:::app-data/file-1"]</code></td>
  <td>All objects in a dedicated bucket:<br/><br/><code>["arn:aws:s3:::app-data/*"]</code></td>
  <td>All objects in a shared bucket: <code>["arn:aws:s3:::shared-data/*"</code>
  <br/><br/>
  or
  <br/><br/>
  all Buckets: &nbsp;<code>*</code>
  <br/><br/>
  (this is what AWS Managed Policies do)</td>
</tr>

<tr>
  <td>Condition - <code style="font-size: 11px;">aws:PrincipalArn</code></td>
  <td><code>arn:aws:iam::111: ↲</code><br/><code>&nbsp;&nbsp;role/app</code><br/><br/>(Same as Principal)</td>
  <td>All IAM 'app' roles with a known variation:
  <br/><br/>
  <code>arn:aws:iam::111:role/app-*</code>
  <br/><br/>
  or
  <br/><br/>
  <code>arn:aws:iam::111:role/env-prod/app-????????</code>
  </td>
  <td>Another AWS Account: <code>123456789012</code> or All AWS accounts <code>*</code><br/><br/>(Same as Principal)</td>
</tr>

</tbody>
</table>

**Table 3.1 How wildcards affect scope**

So engineers now have a huge information gathering and evaluation task to perform.  They must correctly:

* retrieve from zero to _tens_ of identity policies that may be directly attached as a managed or inline policy, or indirectly attached via a group, again managed or inline
* retrieve resource policies for relevant resources involved in the request path, e.g. S3 bucket or KMS key
* parse the policies and build a mental model of what each statement brings in and out of scope
* calculate the effects of the statement, properly accounting for Effect precedence and negation

But engineers usually do this in their heads with less than perfect information about the policies in their system and how IAM works.  Mistakes will happen.

Engineers will get the `Allow` path to work.  No team is going to give up until the application or person's principal's can execute requests successfully.  But most teams will move on long before achieving the least privilege they intended.

[//]: # (Do not change the following lines as it may break the rendering of your objects)
```pulse-diagram
{"_key":"1853915a-6f19-4a93-9a6f-452f385a0bb5"}
```

**Figure 3.2 Implementing Least Privilege to Resources**

Because for engineers to grant _only_ the access they intended to principals and resources, two non-obvious things should be included in their security policies:

1. Identity policies attached to principals should scope resource access to implement least privilege for the _principal_
2. Resource policies should allow intended principals and deny everyone else to implement least privilege for the _resource_

Don't let anyone minimize this task.

Writing minimal access policies is impossible for humans working with real systems without serious help.

## Problems with the usability of AWS IAM
The biggest problem with AWS security is that most people cannot confidently configure security policies using information they have in their head or at hand.  They need to look things up constantly, and they're still unsure.  While configuring security controls is an 'everyday' experience, engineers cannot configure policies quickly and correctly enough to achieve their goals.

This is _not_ the engineers' fault.

Engineers are using a system which has not been designed for usability.  

The AWS security policy system requires users to know and understand a tremendous amount in order to use it effectively.  Engineers must understand a lot about how AWS security policies are evaluated, how the policy language works, the features of the policy language for a given service, and the state of the system in order to create a mental model and finally create policies.

This is a serious usability problem because there is a large difference between what most engineers have and what they need to secure resources effectively.  Engineers must expend a lot of effort to configure AWS IAM correctly.  Usability expert Don Norman describes this as two "Gulfs"[^1] in [The Design of Everyday Things][4]:

![Gulfs of Execution & Evaluation for AWS Security](https://effectiveiam.com/EIAM-norman-gulfs-aws-security.png)
[//]: # (Do not change the following lines as it may break the rendering of your objects)
```pulse-diagram
{"_key":"cb41bad7-6665-46ac-8106-0ab8a07b4560"}
```

**Figure 3.2 The Gulfs of Execution and Evaluation for AWS Security**

Engineers must cross two gulfs:
1. The Gulf of Execution, where they try to figure out how to use IAM
2. The Gulf of Evaluation, where they try to figure out what state IAM is in and whether their actions got them to their goal

Engineers have difficulty answering Norman's critical usability questions for AWS Security:
* How do I work this? 5 types of security policy and many attachment points interact within a complex control flow. 🤔
* What can I do? The basic security policy language features are described on a single page, but engineers must understand volumes spread across specific AWS service docs.  The [AWS IAM user guide][5] is over 800 pages long! 🤯
* What happened? AWS provides no comprehensive way to understand the net effects of changes to principal and resource access. CloudTrail events provide delayed, sometimes incomplete feedback on why access was denied. 😕
* Is this what I wanted?  Difficult to tell if a given set of policies allows or denies access as intended. Critically, there is no comprehensive mechanism to identify excess access. 😖

AWS' flexibility requires engineers to gather and manage large amounts of security information in their heads.  Norman calls this "knowledge in the head."  AWS generally does not codify security best practices directly in the service, which would be "knowledge in the world."  Engineers must work hard to build this mental model and it takes time, so it will be different from colleagues' models.

In AWS' shared responsibility model, the responsibility for designing a set of access controls falls to the customer.  AWS provides the security primitives, customers provide almost everything else.

AWS partially recognizes the burden of acquiring all this knowledge.  AWS has tried to encode _some_ security knowledge "in the world."  Encoding knowledge in the world is "how designers can provide the critical information that allows people to know what to do, even when experiencing an unfamiliar device or situation." [^2]

Some examples of AWS encoding knowledge in the world are:
* Managed policies for particular jobs: Data Scientist,  Power User, etc
* Control Tower and Landing Zones' Service Control Policies
* Config Rules
* Beanstalk
* "Level 2" and "Level 3" constructs in AWS CDK

Given that AWS' is famously flexible, backwards compatible, and built by a collection of hundreds of independent teams, it's unlikely we'll see a cohesive set of usable security tools and libraries emerge from AWS which you can adopt into new or existing applications.  It's even more unlikely those security tools would work in other Clouds which have similar problems.

But third parties can help.  Internal platform teams, communities, and vendors can create opinionated, usable solutions that simplify security for their niche.  As Norman says[^3]:

> Because behavior can be guided by the combination of internal and external knowledge and constraints, people can minimize the amount of material they must learn, as well as the completeness, precision, accuracy, or depth of the learning. They also can deliberately organize the environment to support behavior. This is how nonreaders can hide their inability, even in situations where their job requires reading skills.

Third-party toolmakers need to help AWS users create better safety outcomes through a combination of approaches.

First, help users apply the knowledge they have to the unfamiliar AWS security system.  Enable application engineers and others versed in the domain to describe what access _should_ be.

Second, constrain configuration choices to a small set of safe options designed to work together.

These approaches have several benefits:
1. minimizes the AWS security knowledge the majority of engineers must learn to a manageable corpus
2. improves communication with a higher level language, patterns, and reference architectures for designing and implementing safer systems
3. builds design, implementation, and operational familiarity with a common set of reusable components

**We must help engineers use the information they _have_** to create great security outcomes.

[^1]:	Norman, Donald A.. The Design of Everyday Things (p. 38). Basic Books. Kindle Edition.

[^2]:	Norman, Donald A.. The Design of Everyday Things (p. 123). Basic Books. Kindle Edition.

[^3]:	Norman, Donald A.. The Design of Everyday Things (p. 76). Basic Books. Kindle Edition.

[1]:	https://aws.amazon.com/blogs/aws/happy-10th-birthday-aws-identity-and-access-management/
[2]:	https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html
[3]:	https://twitter.com/mamip_aws
[4]:	https://www.basicbooks.com/titles/don-norman/the-design-of-everyday-things/9780465050659/
[^5]:	https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html

Cover photo by [Michał Parzuchowski](https://unsplash.com/photos/geNNFqfvw48) on [Unsplash](https://unsplash.com/).
