# How I Made Sense of the Master User in the Amazon ElasticSearch Service  

## Author [Yann Stoneman](https://ystoneman.medium.com/a85b8518b1b6)

If you are reading this post, it’s probably because you are knees deep into figuring out how to enable fine-grained access control. And that means you are delegating most of your identity and access management to the controls of the Open Distro for ElasticSearch (soon to be OpenSearch) instead of AWS IAM. This is explained in the below screenshot from the wizard for creating an ES domain:

![Fine-grained access control](https://miro.medium.com/max/1400/1*KcLfkUIvuhPHrRMAO728dw.png "Fine-grained access control")

The master user in ES can be either a username and password or an AWS IAM principal (a role or a user). Both just function to get you in the door of ES as a master user. It’s like the username and password combo for the master user of an RDS MySQL database.

### So the policies on the IAM role or user don’t even matter?

Right. So you **might as well give zero permissions** to the IAM role or IAM user, and, as long as the machine or person can authenticate to that role or user, then they will have the powers of “master user” in ES.

The AWS IAM role or IAM user serve purely for authentication—the policies on that role or user have no bearing on the authorization of the ES master user. Those are handled via the controls provided within ES itself.

### I’ve never needed to create an AWS IAM user without permissions. Show me what you mean!

Right. This made me scratch my head too. But if you do a GitHub search for code of how an IAM-based master user is implemented, you’ll get templates where the actual user or role does not have any permissions.

For example, I grabbed [this](https://github.com/aws/aws-cdk/blob/2337b5d965028ba06d6ff72f991c0b8e46433a8f/packages/%40aws-cdk/aws-appsync/test/integ.graphql-elasticsearch.expected.json#L3-L16) from the official [aws/aws-cdk](https://github.com/aws/aws-cdk) repo on GitHub:

```
Resources:
  User1:
    Type: AWS::IAM::User
  Domain1:
    Type: AWS::Elasticsearch::Domain
    Properties:
      AdvancedSecurityOptions:
        Enabled: true
        InternalUserDatabaseEnabled: false
        MasterUserOptions:
          MasterUserARN: !GetAtt User1.Arn
```

The IAM *user* in that template has *no permissions*. It’s just specified as the master user in the ES domain definition.

And as another example, taken from an [unofficial repo](https://github.com/mats16/amazon-elasticsearch-cognito-auth/blob/8a8d1ade7da6e1fb120de478099215dd28a71a84/template.yaml) of a [Solution Architect](https://github.com/mats16) at AWS, this time with an AWS IAM role as the master user:

```
Resources:
  Elasticsearch:
    Type: AWS::Elasticsearch::Domain
    Properties:
      AdvancedSecurityOptions:
        Enabled: true
        InternalUserDatabaseEnabled: false
        MasterUserOptions: 
          MasterUserARN: !GetAtt KibanaMasterRole.Arn
      # ...
      # hiding other parts to emphasize master user
      # ...
  KibanaMasterRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Action: "sts:AssumeRoleWithWebIdentity"
            Principal:
              Federated: cognito-identity.amazonaws.com
            Condition:
              StringEquals:
                "cognito-identity.amazonaws.com:aud": !Ref KibanaIdentityPool
              ForAnyValue:StringLike:
                "cognito-identity.amazonaws.com:amr": authenticated
  KibanaMasterGroup:
    Type: AWS::Cognito::UserPoolGroup
    Condition: FineGrainedAccessControlEnabled
    Properties: 
      Description: Master users of Kibana
      GroupName: Master
      Precedence: 0
      RoleArn: !GetAtt KibanaMasterRole.Arn
      UserPoolId: !Ref KibanaUserPool
```

Again, the template has *no permissions for the master user, this time defined as an IAM role*.

### So does the master user just have complete permissions that don’t need to be explicitly defined within AWS IAM?

Yes, the master user is the master of AWS ES, regardless of what the IAM role or user is.

### How do I test that the permissionless IAM user can really access ES?

To check this out quickly on a test ES domain in a test AWS account, *disregarding all security best-practices* to isolate testing of the master user, I:

 - launched an ES domain, set as public facing (non-VPC-based)
 - created an IAM user called User1 with *no policies*
 - designated master user as User1
 - and used the default open resource-based policy (aka, the domain access policy), in which nothing is limited:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": "es:*",
      "Resource": "arn:aws:es:us-east-1:12345678901:domain/test/*"
    }
  ]
}
```

![Access policy](https://miro.medium.com/max/1400/1*d2jfQ6-X3dZBQ-rEODppCA.png "Access policy")

Once the domain was provisioned, I plopped the endpoint in the browser to verify that it was publicly accessible, but in need of authentication:

![Authentication Failed](https://miro.medium.com/max/1400/1*Gt4CeUjL4tEaJqNvzmyDOA.png "Authentication Failed")

Then I grabbed a Python code snippet from the Amazon documentation on [“Signing HTTP requests to Amazon ElasticSearch Service.”](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-request-signing.html#es-request-signing-python) This helped me to see in action how the master user’s IAM user credentials are used to authenticate a request:

```
# Author: https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-request-signing.html#es-request-signing-python

from elasticsearch import Elasticsearch, RequestsHttpConnection
from requests_aws4auth import AWS4Auth
import boto3

host = 'search-testing-mvevysr2i4aen6pqfq5ympghsy.us-east-1.es.amazonaws.com' # For example, my-test-domain.us-east-1.es.amazonaws.com
region = 'us-east-1' # e.g. us-west-1

service = 'es'
credentials = boto3.Session().get_credentials()
awsauth = AWS4Auth(credentials.access_key, credentials.secret_key, region, service, session_token=credentials.token)

es = Elasticsearch(
    hosts = [{'host': host, 'port': 443}],
    http_auth = awsauth,
    use_ssl = True,
    verify_certs = True,
    connection_class = RequestsHttpConnection
)

document = {
    "title": "Moneyball",
    "director": "Bennett Miller",
    "year": "2011"
}

es.index(index="movies", doc_type="_doc", id="5", body=document)

print(es.get(index="movies", doc_type="_doc", id="5"))
```

When we run that code locally with a random IAM user’s credentials like those of my machine’s default user called OtherUser, we get an error saying,

```
AuthorizationException(403, 'security_exception', 'no permissions for [indices:data/write/index] and User [name=arn:aws:iam::12345678901:user/OtherUser, backend_roles=[], requestedTenant=null]')
```

When I updated the code to have boto3 grab my named profile `User1` instead, it worked.

With `credentials = boto3.Session(profile_name='User1').get_credentials()` I got back the desired response:

```
{
   "_index":"movies",
   "_type":"_doc",
   "_id":"5",
   "_version":1,
   "_seq_no":0,
   "_primary_term":1,
   "found":true,
   "_source":{
      "title":"Moneyball",
      "director":"Bennett Miller",
      "year":"2011"
   }
}
```

### And how do you integrate with an external identity provider like Okta?

Same way as with any AWS IAM role, you just add an assume role policy to the role:

```
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Principal":{
            "Federated":"<COPY & PASTE SAML ARN VALUE HERE>"
         },
         "Action":"sts:AssumeRoleWithSAML",
         "Condition":{
            "StringEquals":{
               "SAML:aud":"https://signin.aws.amazon.com/saml"
            }
         }
      }
   ]
}
```

More details in [the Okta Setup SSO article](https://saml-doc.okta.com/SAML_Docs/How-to-Configure-SAML-2.0-for-Amazon-Web-Service).

So that’s how I made sense of the master user in the AWS ElasticSearch Service.
