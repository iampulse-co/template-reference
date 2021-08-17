# IAM Policies VS S3 Policies VS S3 Bucket ACLs – What should be used?

## Author: [Mahesh Mogal](https://analyticshut.com/iam-policies-vs-s3-policies-vs-bucket-acls/)

We can use IAM policies to manage access for different users for the S3 bucket.
Obviously life is not that simple. With S3 we have Bucket policies and Bucket
Access Control Lists ( hereafter referred to as ACLs) which also can be used to
manage access to S3 buckets. Let us understand the difference between IAM
policies VS S3 Policies and S3 ACLs and when should you use what.

## IAM Policies

IAM polices are used to specify which actions are allowed or denied on AWS
services/resources for particular user. for example, user Tom can read files
from “Production” bucket but can write files in “Dev” bucket where as user
Jerry can have admin access to S3.

We can attach IAM policies to users, groups or roles. These users or roles then
can perform AWS operations depending on permission granted to them by AWS
policy.

Below is sample policy that allows read access to s3 bucket
“test-sample-bucket”. Any user or group or role which has below policy attached
will be able to read data from this bucket.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion"
            ],
            "Resource": "arn:aws:s3:::test-sample-bucket"
        }
    ]
}
```

## S3 Bucket Policies

With S3 bucket policy, you can specify which actions are allowed or denied on
that bucket for some user. The user in context of S3 bucket policies is called
principal. Principle can be IAM user or AWS root account. That means you can
grant access to another AWS account than in which your AWS S3 bucket is
created.

S3 bucket policies can be attached to only S3 buckets. You can use them with
any other AWS user or service. Below is a sample S3 bucket policy that grants
root user of AWS account with ID 112233445566 and the user named Tom full
access to S3 bucket.

```
{
  "Id": "Policy1586088133829",
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1586088060284",
      "Action": "s3:*",
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::test-sample-bucket",
      "Principal": {
        "AWS": [
          "arn:aws:iam::112233445566:root",
          "arn:aws:iam::112233445566:user/Tom"
        ]
      }
    }
  ]
}
```

Note “Principal” statement in S3 bucket policy. Principal has a list of users
who have access to this bucket. In the case of IAM policies “Principal” is not
necessary as this is derived from user or group or role to which IAM policy is
assigned.

> In the case of IAM policies, mentioning “Principal” is not necessary as this
is derived from the user or group or role to which IAM policy is assigned. -
AWS Facts

If you want to try and play around to create S3 bucket policies then AWS has
provided [policy
generator](https://awspolicygen.s3.amazonaws.com/policygen.html). You can try
out creating policies for different scenarios.

## S3 Bucket ACL

S3 ACLs is the old way of managing access to buckets. AWS recommends the use of
IAM or Bucket policies. But there are still use cases where ACLs give
flexibility over policies. That is one of the reasons ACLs are still not
deprecated or going to be deprecated any time soon.

The biggest advantage of using ACL is that you can control the access level of
not only buckets but also of an object using it. Whereas IAM or Bucket Policies
can only be attached to buckets but not to objects in the bucket, Bucket ACLs
can be assigned to buckets as well as objects in it.

This means you have a lot of flexibility and control over your S3 resources.
You can make some object public in a private bucket or vice versa without any
issue.

ACLs can have one of following type of value.

 - “private”
 - “public-read”
 - “public-read-write”
 - “authenticated-read”

## So What Should I Use?

Now that we have understood the basics of IAM Policy, Bucket Policy, and Bucket
ACLs, We can decide in which scenario we should use which type of access
control. Below is a table that should help you decide what you should use in
your case.

| Criteria      | IAM Policies  | Bucket Policies  | Bucket ACLs |
| ------------- | ------------- | ---------------- | ----------- |
| AWS recommendation | AWS recommends using IAM policies where you can.	| You can use bucket policies as its simper way compared to IAM policies. | Bucket ACLs is older way of managing access to S3 buckets and generally are not recommended |
| Environment | IAM allows you to centrally manage all permissions to AWS which is easier | S3 policies are limited to S3 environment only |  S3 ACLs are limited to S3 environment only |
| Use Cases | IAM Policies can specify permission rules to other AWS Services/resources	| Can only be used with S3 | Can only be used with S3 |
| Ease of Use and Maintenance | IAM policies are easy to maintain when you have a large number of users and you frequently need to make changes to permission levels of users (i.e. user promoted or left organization)	| S3 policies are easy to create but become difficult to maintain when you lot of users or if you want to make a change to the access level of some user. You might need to update bucket policies for all buckets if some user need more access to S3 buckets | Same as S3 bucket Policies, ACLs are difficult to maintain |
| Object Level Control | IAM policies can only be attached to the root level of the bucket and cannot control object-level permissions. | Using ACL is that you can control the access level of not only buckets but also of an object using it. | If you need to manage object level permissions in S3, then you need to use Bukcet ACLs. |

## Conclusion

I hope you have learned the difference between IAM policies, S3 policies, and
S3 ACLs. It is your decision to use one of them depending on your use case. If
you have any questions, let me know. See you in the next blog. Cheers.
