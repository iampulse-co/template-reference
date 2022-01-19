# Determining AWS IAM Policies According To Terraform And AWS CLI  

![Architecture](https://meirg.co.il/wp-content/uploads/2021/04/determining-aws-iam-policies-according-to-terraform-and-aws-cli-cover.png "Architecture")

I find myself mentioning the term [Principle Of Least
Privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege) often,
so I thought, "Let's write a **practical** blog post of how to implement this
principle in the CI/CD realm".

In this blog post, I'll describe the process of granting the least privileges
required to execute `aws s3 ls` and `terraform apply` by a CI/CD runner.

## HIPAA  

In case you're from health tech, this process might help you with qualifying
some of the [HIPAA
compliance](https://www.hhs.gov/hipaa/for-professionals/security/laws-regulations/index.html)
requirements, see [HIPAA's Minimum Necessary
Requirement](https://www.hhs.gov/hipaa/for-professionals/privacy/guidance/minimum-necessary-requirement/index.html).

## Scenario  

Imagine this, you've created an IAM user `cicd-user`, with
[AdministratorAcess](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html)
and generated the access keys `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`
for that user. You've created that user so that your CI/CD service, whatever it
is, [GitHub Actions](https://github.com/features/actions),
[drone.io](https://www.drone.io/), [Jenkins](https://www.jenkins.io/), etc.,
will be able to apply changes in your AWS account.

This is a common scenario that usually happens in small startup companies,
where the product and sales are far more important than meeting regulations and
securing the product. But why do people do that? Because every time a CI/CD
attempts to do something, figuring out which policies are required for the job
is a nightmare.

> "Let's provide the CI/CD service an admin permission, we'll deal with that
> later, we must focus on the product."

![Avenged Sevenfold - Nightmare](https://i.ytimg.com/vi/VzkBv1-Y-TE/maxresdefault.jpg)

**TIP**: Listen to [Avenged Sevenfold -
Nightmare](https://www.youtube.com/watch?v=94bGzWyHbu0&ab_channel=AvengedSevenfold)
while reading this blog-post.

## The Nightmare  

A typical "Nightmare Situation" where you need to create an IAM policy for a
CI/CD service; here goes.

 1. Create an IAM user with no permissions and generate access keys for the CI/CD service.
 2. IAM user (CI/CD job or you) invokes some `aws` or `terraform` command.
 3. If the operation fails due to authorization (403) issues, then you'll get
an error message that states which permission(s), usually it's singular, is
required.
 4. Add the required permission(s) to the user's IAM policy.
 5. Start again from step 2 - invoke `aws` or `terraform` ...

**NOTE**: Sometimes, you'll get `Forbidden status code: 403`, which is quite
useless for finding out the required IAM permissions.

## There's A Tool For That

Here comes [iamlive](https://github.com/iann0036/iamlive) by [Ian
Mckay](https://github.com/iann0036).

**Generate an IAM policy from AWS calls using client-side monitoring (CSM) or embedded proxy**

![iamlive](https://raw.githubusercontent.com/iann0036/iamlive/assets/iamlive.gif)

I'll focus on the [Proxy Mode](https://github.com/iann0036/iamlive#proxy-mode),
which supports running `iamlive` as a Docker container. No worries, we'll get to
that.

## Proxy Mode? What Do you Mean?

The way `iamlive` works is pretty simple. `iamlive` runs in the background in
proxy mode and serves `0.0.0.0:10080`, which allows access from "any IP", but
only we have access to this process, so we're good. In a separated terminal,
you set `HTTP_PROXY`, `HTTPS_PROXY`, and `AWS_CA_BUNDLE` according to iamlive.

## Running iamlive in Docker
### Build The Image Locally

For full visibility, since we're talking about credentials, let's build the
Docker image locally. Copy-paste the following `Dockerfile`:

**Dockerfile**
```
ARG GO_VERSION=1.16.3
ARG REPO_NAME=""
ARG APP_NAME="iamlive"
ARG APP_PATH="/go/src/iamlive"

# Dev
FROM golang:${GO_VERSION}-alpine AS dev
RUN apk add --update git
ARG APP_NAME
ARG APP_PATH
ENV APP_NAME="${APP_NAME}" \
    APP_PATH="${APP_PATH}" \
    GOOS="linux"
WORKDIR "${APP_PATH}"
COPY . "${APP_PATH}"
ENTRYPOINT ["sh"]

# Build
FROM dev as build
RUN go install
ENTRYPOINT [ "sh" ]

# App
FROM alpine:3.12 AS app
RUN apk --update upgrade && \
    apk add --update ca-certificates && \
    update-ca-certificates
WORKDIR "/app/"
COPY --from=build "/go/bin/iamlive" ./iamlive
RUN addgroup -S "appgroup" && adduser -S "appuser" -G "appgroup" && \
    chown "appuser:appgroup" "./iamlive"

USER "appuser"
EXPOSE 10080
ENTRYPOINT ["./iamlive"]
CMD ""
```

We need the source code of [iamlive](https://github.com/iann0036/iamlive) to
build the Docker image, so let's `git clone` it and then build the Docker
image.  Though before building the image, let's add a
[.dockerignore](https://docs.docker.com/engine/reference/builder/#dockerignore-file)
file to copy only the required files to build the `iamlive` application.

**.dockerignore**
```
**
!LICENSE
!service/
!vendor/
!.gon-*.json
!map.json
!iam_definition.json
!*.go
!*.mod
!*.sum
```

Now let's get the source code and build the Docker image.

```
# Get source code
git clone https://github.com/iann0036/iamlive.git
cd iamlive

# Build the Docker image from source code and tag it
docker build -t iamlive-test  .
```

### Run iamlive-test Docker Container

We need to proxy all of `aws` and `terraform` requests via `iamlive-test`, and then
iamlive will be able to generate the relevant IAM permissions according to the
invoked request. Pretty awesome, right?

**Zooming in on some of the arguments**

 1. `-p 80:10080` and `-p 443:10080` - Maps the ports 80 and 443 in the Host to the Container port 10080
 2. `--bind-addr 0.0.0.0:10080` - iamlive listens on port 10080, from any IP address
 3. `--force-wildcard-resource` - Makes it easier to iterate over missing permissions
 4. `--output-file "/app/iamlive.log"` - Save the generated permissions to a file upon kill -HUP 1.

```
docker run \
  -p 80:10080 \
  -p 443:10080 \
  --name iamlive-test \
  -it iamlive-test \
  --mode proxy \
  --bind-addr 0.0.0.0:10080 \
  --force-wildcard-resource \
  --output-file "/app/iamlive.log"
# Runs in the background ...
# Average Memory Usage: 88MB
```

## Using The Proxy

First, I recommend that you create a fresh new IAM user with **no permissions at
all**, let's name that user `dummy-user`. Doing so will ease getting the minimum
required permissions (**all of them**).

The fact that the `iamlive-test` container is running means nothing to `aws` and
`terraform`. To configure both CLIs to use this proxy server, **open a new terminal
window** and execute the below commands.

```
export AWS_ACCESS_KEY_ID="AKIA_DUMMY_USER_ACCESS_KEY_ID"
export AWS_SECRET_ACCESS_KEY="DUMMY_USER_SECRET_ACCESS_KEY"

export HTTP_PROXY=http://127.0.0.1:80 \
       HTTPS_PROXY=http://127.0.0.1:443 \
       AWS_CA_BUNDLE="${HOME}/.iamlive/ca.pem"
```

Say what? From where did this
[AWS_CA_BUNDLE](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html)
come from? Well, this environment variable instructs tools that use the [AWS
SDK](https://aws.amazon.com/tools/) to trust the provided [Certificate
Authority Certificate](https://en.wikipedia.org/wiki/Certificate_authority), in
our case, it's `ca.pem`.

The `ca.pem` file is generated by `iamlive` for each execution of the `iamlive-test`
container. We need to copy the `ca.pem` file from the container to our machine
(Host).

```
docker cp iamlive-test:/home/appuser/.iamlive/ ~/
```

The environment variables [HTTP_PROXY and
HTTPS_PROXY](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-proxy.html)
are telling AWS's SDK to forward traffic via a proxy server (iamlive-test
container).

## Generating IAM Policies With Relevant Permissions

So far, we've got two terminal windows. In the first terminal, we have the
`iamlive-test` container running, and it probably looks like it's doing nothing,
but it's running, trust me. In the second terminal, we exported the relevant
environment variables and copied `ca.pem` from the `iamlive-test` docker container
to our machine (Host).

## Executing A Command With AWS CLI

In the second terminal, we'll execute commands with `aws` and `terraform`. After
the command execution is completed, we'll inspect the logs of the first
terminal, which runs the `iamlive-test` container.

```
aws s3 ls

# Output
# An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied
```

Really? All I need is the `ListBuckets` permission? The real permission is called
`s3:ListAllMyBuckets`, and I know that because the logs of `iamlive-test` look like
this.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListAllMyBuckets"
            ],
            "Resource": "*"
        }
    ]
}
```

## Executing A Command With Terraform CLI

Before we proceed, it's important to mention that `terraform init` cannot be
proxied via `iamlive-test` since it attempts to access
[registry.terraform.io](https://meirg.co.il/2021/04/23/determining-aws-iam-policies-according-to-terraform-and-aws-cli/registry.terraform.io),
and it's not covered by `iamlive`. So first, unset the proxy settings, and then
execute `terraform init`.

This is what it looks like when you attempt to execute `terraform init` with the
proxy settings (environment variables) on.

```
terraform init

 Error: Failed to query available provider packages

 Could not retrieve the list of available versions for provider
 hashicorp/aws: could not connect to registry.terraform.io: Failed to
 request discovery document: Get
 "https://registry.terraform.io/.well-known/terraform.json": x509:
 certificate signed by unknown authority
```

For testing purposes, create a new directory and add the following `main.tf`
file.

```hcl
variable "region" {
  type = string
  default = "eu-west-1"
}

provider "aws" {
  region = var.region
}

resource "aws_s3_bucket" "app" {}
```

Unset proxy related environment variables and then execute terraform init

```
unset HTTP_PROXY HTTPS_PROXY AWS_CA_BUNDLE

mkdir terraform-iamlive
cd terraform-iamlive
vim main.tf # copy-paste the above main.tf file

terraform init
# Terraform has been successfully initialized!
```

Finally, we can execute `terraform apply` and see which permissions are required
for the task.

```
export AWS_ACCESS_KEY_ID="AKIA_DUMMY_USER_ACCESS_KEY_ID"
export AWS_SECRET_ACCESS_KEY="DUMMY_USER_SECRET_ACCESS_KEY"

export HTTP_PROXY=http://127.0.0.1:80 \
       HTTPS_PROXY=http://127.0.0.1:443 \
       AWS_CA_BUNDLE="${HOME}/.iamlive/ca.pem"

# In terraform-iamlive dir
terraform apply

# Output
# Error: error reading S3 Bucket (terraform-20210422212704452600000001): Forbidden: Forbidden
# │       status code: 403, request id: A25SVTBABN0B3DSH, host id: /c1b5TsnsBE23AaDDHJQ34yLAYdrR7y3kvu2lqEX7VvstffawROKWwcPYfxNjleeluZPg9nucKY=
```

No idea what that means; let's check `iamlive-test` container logs to see if
`iamlive` knows which permissions are required.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sts:GetCallerIdentity",
                "ec2:DescribeAccountAttributes",
                "s3:ListBucket"
            ],
            "Resource": "*"
        }
    ]
}
```

Sometimes, you won't get the full list of required permissions. To overcome
that, add the given IAM policy and invoke `terraform apply` again to see which
permissions are missing.

Also, you might want to limit "`*`" to specific resources or patterns, but still,
it's better than the current nightmare.

Here's the output **after** adding the above IAM policy to my "dummy-user".

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sts:GetCallerIdentity",
                "ec2:DescribeAccountAttributes",
                "s3:ListBucket",
                "s3:GetBucketAcl"
            ],
            "Resource": "*"
        }
    ]
}
```

A new permission was added - `s3GetBucketAcl`. We need to iterate this process a
few times, but each time it's a simple copy-paste of the generated permissions
to the existing IAM policy in AWS Console.

This is the final result, after eight (8) iterations. If you're about to deploy
a large stack with multiple resources, you'll have to iterate more than a few
times.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sts:GetCallerIdentity",
                "ec2:DescribeAccountAttributes",
                "s3:ListBucket",
                "s3:GetBucketAcl",
                "s3:GetBucketCORS",
                "s3:GetBucketWebsite",
                "s3:GetBucketVersioning",
                "s3:GetAccelerateConfiguration",
                "s3:GetBucketRequestPayment",
                "s3:GetBucketLogging",
                "s3:GetLifecycleConfiguration",
                "s3:GetReplicationConfiguration",
                "s3:GetEncryptionConfiguration",
                "s3:GetBucketObjectLockConfiguration",
                "s3:GetBucketTagging",
                "s3:CreateBucket"
            ],
            "Resource": "*"
        }
    ]
}
```

**IMPORTANT**: Remember, limit "`*`" to specific resources or patterns.

**NOTE**: After adding the policy to an IAM user, it took a few retries to get the
updated IAM policy from `iamlive-test`. Just keep on adding permissions until a
successful attempt.

## Stop And Start iamlive-test

The beauty of omitting the flag `--rm` in the `docker run` command is that
`iamlive-test` will **not be removed** when it stops. This is important! We want to
keep the same `ca.pem` in the next execution of `iamlive-test` instead of
re-copying `ca.pem` from `iamlive-test` to our machine (Host).

```
# Hit CTRL+C To stop the container

docker start -i iamlive-test
# Keep it running in the background
```

## Get The Lastest Generated IAM Policy

We can send a SIGHUP signal to the `iamlive-test` container, which instructs
`iamlive` to dump its latest output to the file `iamlive.log`. I piped the output
through [jq](https://stedolan.github.io/jq/) to beautify it.

```
docker exec iamlive-test kill -HUP 1 && \
docker exec iamlive-test cat /app/iamlive.log | jq
```

![s3:ListAllMyBuckets](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7yu04pj6d6er9nk60n32.png "s3:ListAllMyBuckets")

## Stop Using The Proxy

To avoid using a proxy server for [AWS
SDK](https://docs.aws.amazon.com/sdk-for-go/v1/developer-guide/configuring-sdk.html#:~:text=Configuring%20a%20Proxy)
operations, unset the relevant environment variables, or restart your terminal
window.

```
unset HTTP_PROXY HTTPS_PROXY AWS_CA_BUNDLE
```

## Alternatives

### CloudTrail

I've tried getting the required permissions by investigating [AWS
CloudTrail](https://aws.amazon.com/cloudtrail/) logs, and again, it was a
nightmare. I just wanted a simple way, with the least overhead, to generate
permissions easily for my CI/CD services.

### IAM Policy Simulator

In case you don't know, AWS provides the **free** service [IAM Policy
Simulator](https://policysim.aws.amazon.com/home/index.jsp), which is great for
testing and debugging IAM policies in your AWS account. Then again, different
tools for different purposes.

### IAM Access Analyzer

Fresh from the oven, [AWS extended the capabilities of IAM Access
Analyzer](https://aws.amazon.com/about-aws/whats-new/2021/04/review-last-accessed-information-identify-unused-ec2-iam-lambda-permissions-tighten-access-iam-roles/),
posted on Apr 19, 2021. Though its capabilities are still limited and do not
cover all AWS services. It's still nice to see that AWS is improving the
capabilities to ease [writing least privilege IAM
policies](https://aws.amazon.com/blogs/security/techniques-for-writing-least-privilege-iam-policies/).

## Final Thoughts

I'll probably write some Bash script that invokes `terraform apply` and when the
error code equals `403`, adds the output of `iamlive.log` to the relevant IAM
policy in AWS. That might make it friendlier than it is now; it's annoying to
copy-paste.

Got a better way to achieve the same thing? Have some thoughts about this
process? Let's discuss it; feel free to comment below!
