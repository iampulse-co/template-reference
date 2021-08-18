# CloudFormation Example for an IAM User with Rotating Credentials

## Author: [Thomas Step](https://thomasstep.com/blog/cloudformation-example-for-an-iam-user-with-rotating-credentials)

 - [x] AWS
 - [x] CloudFormation

I had a need for an IAM User not too long ago and wanted to create a
CloudFormation template instead of going through the console. I do not create
IAM entities too often, so I figured that this would be a good time to cement
my knowledge into a template. I wanted the user to have CLI access for some
automation, which meant that I needed to also create an access key. While I was
looking through the documentation for access keys I noticed an interesting
field: `Serial`.

`Serial` is a field specific to CloudFormation that accepts an integer. If that
integer is increased, the access key is rotated. This is a cool feature that I
knew I wanted to test out.

After creating an IAM User, I wanted to create an access key for that user
based on a `Serial`, and after that access key was created, I wanted to store
the credentials in a secret. Whenever the credentials needed to be rotated, it
should be as simple as incrementing the `Serial` and grabbing the new
credentials from the secret. Here is what I came up with.

```yaml
Parameters:
  Serial:
    Type: Number
    Description: Increment this to rotate credentials

Resources:
  IamUser:
    Type: AWS::IAM::User
    Properties: 
      ManagedPolicyArns: 
        - arn:aws:iam::aws:policy/AdministratorAccess
  AccessKey:
    Type: AWS::IAM::AccessKey
    Properties: 
      Serial: !Ref Serial
      Status: Active
      UserName: !Ref IamUser
  AccessKeySecret:
    Type: AWS::SecretsManager::Secret
    Properties: 
      Description: !Sub "These are the credentials for the IAM User ${IamUser}"
      SecretString: !Join
        - ""
        - - '{"AccessKeyId":"'
          - !Ref AccessKey
          - '","SecretAccessKey":"'
          - !GetAtt AccessKey.SecretAccessKey
          - '"}'
```

This template is also available in my [aws-cloudformation-reference
repository](https://github.com/thomasstep/aws-cloudformation-reference/blob/21543313e993c2183c95846255a1fcfa350f9870/iam/iam-user.yml).
I also [made a video of me creating the
template](https://www.youtube.com/watch?v=hcEWgwP2xzk) in case the process of
building and deploying something like this from scratch is of interest.
