<img src="https://iampulse-public.s3.us-west-1.amazonaws.com/iampulse-logo.png" width="200" />

# Introduction
This repo is reference documentation for articles to be published on [iampulse.com](https://www.iampulse.com)

IAM Pulse is a technical content community designed for cloud practitioners to share, consume, and discuss IAM configuration best practices across the major cloud providers. Why focus on IAM? Because it's the single service that touches everything – people, resources, workloads, and data – and it's the hardest to get right. 

Our community platform is centered around semi-structured technical content, which we refer to as an **Article**. We want to give our authors the flexibility to create content in their own style with their own voice, while also making each contribution reference-able and usable by others.

Members can self publish on the site, or if you prefer, follow the instructions below to craft an **Article** and share it directly with our team. Early authors & members benefit from getting showcased early... and a nice little swag bag :)

## Instructions
Fork this repo, choose one of the examples provided, or just follow along in your own Markdown editor to craft your own Article. Our publishing platform supports a series of select attributes to tag each article appropriately, and includes custom markdown blocks for code snippets and images. 

### Adding code blocks & images
As you're writing your Article, if you add a code block, use the standard 3-ticks, and stack the code with reference docs together. Because we're dealing with infrastructure & policy code that will naturally be custom to the environment, it's encouraged to document which variables need to be replaced and/or which elements need to be customized.


```
Code:
{Your Code Here: make it easy to see what's a variable, like uppercase values.}
```
```
Docs:
{Code Docs Here: make note of any variables to replace or elements to customize.}
```

When including images, use standard markdown and include a caption.

![enter image caption here](https://i.picsum.photos/id/864/200/200.jpg?hmac=enPW23d2MpTvv2RfL7CtuO_cKSvCg4DGCYtNPc4-48M)

### Selecting attributes
Include the following section at the top of your **Article**, and check off which of the following apply to your content.

**Cloud Provider(s)**
 - [ ] AWS
 - [ ] GCP
 - [ ] Azure
 - [ ] Other

**Code Framework(s)**
 - [ ] CloudFormation
 - [ ] Terraform
 - [ ] Pulumi
 - [ ] Ansible
 - [ ] Puppet
 - [ ] Chef
 - [ ] SaltStack
 - [ ] Other

**Identity Provider(s)**
 - [ ] Okta
 - [ ] Active Directory
 - [ ] LDAP
 - [ ] Other

**Compliance Guideline(s)**
 - [ ] AWS CIS
 - [ ] CSA
 - [ ] SOC 2
 - [ ] PCI-DSS
 - [ ] NIST
 - [ ] FedRAMP
 - [ ] Other

**Technical Theme(s)**
 - [ ] Zero Trust
 - [ ] Serverless
 - [ ] Kubernetes

## Your Article

### Choosing the type of Article to write
Before you begin writing your Article it is important to define the nature of the content and the intened audience. We've taken inspiration from the [Diataxis Framework](https://diataxis.fr/), and modeled the content types as the following:
* [Tutorials](examples/TUTORIAL.md)
* [How-To Guides](examples/HOWTO.md)
* [Reference Architectures](examples/REFERENCE.md)
* [Topic Explainers](examples/EXPLAINER.md)
* [Tips & Tricks](examples/TIP.md)

Ask yourself what it is you're trying to accomplish with this content.
* Are you trying to teach a specific skill? (Tutorial)
* Do you want to guide someone to solve or avoid a problem? (How-To Guide)
* Are you attempting to describe an infra environment, piece of code or API use case? (Reference Architecture)
* Are you writing an article about the history and evolution of the industry? (Topic Explanation)
* Did you uncover something in your work or research that was an "aha" moment? (Tip or Trick)
