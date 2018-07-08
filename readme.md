# Cloudformation Extensions Macro (CFNX)
(Reserved alternate names for the project: CFNX, CFNExtensions, CFNXMacro, Generic Custom Resource, GCR, Custom Resource Helper, CRH)

### Gist

__GCR__ (or Generic Custom Resource) is not just a generic custom resource acting as a proxy. It is an extension to the already powerful AWS Cloudformation Service. It lets you __manage AWS resources missing pre defined resource types__ or __existing resources not created via cloudformation__, but also lets you __work with HTTP services__ without having to develop any code. If you like writing programs, it even lets you write pure __Python programs in the cloudformation template__ without having to worry about the whole exception handling, timeouts, retries, logging, signaling jargon. Moreover, it provides __new intrinsic functions__ which help in writing __programs__ in cloudformation tempalte which otherwise would need you to develop a new custom resource everytime. __GCR enables extended use of AWS cloudformation service__ and templates to work with scenarios where you:

* Want to __manage an existing AWS resource__ created outside cloudformation?
* Want to __manage an AWS resource not supported in cloudformation?__
* Want to __add Sleep/Delay__ for few seconds because you are seeing eventual consistency issues?
* Want to __manipulate data passed via parameters__ before passing it to other resources?
* Want to __pass complex data types as parameters__ but cloudformation supports only String, Integer, Boolean (other built-ins).
* Want to __ensure your application is healthy__ after updating a resource?
* Want to __share outputs across accounts/regions__ of the stack? or even __export big outputs__ which are currently limited in size?
* Want to __retrieve attributes__ of a resource because __Fn::GetAtt does not support the attribute__?
* Want to __read configuration or secrets from S3__ and pass it to your resource?
* Want to __run test cases for production application__ for new deployments via cloudformation stack?
* Want to do anything outside cloudformation but make it part of cloudformation stack execution

Documentation
---
- [FAQ](#FAQ)
- [Setup](#Setup)
  - [Initial](#InitialSetup)
  - [Deployments](#Deployments)

### FAQ
____________
##### Do I have to learn something very new to use this?
```
Not really. GCR is like any other pre-defined resource type. eg: AWS::EC2::Instance
All you do is provide properties for GCR resource.
```

##### What does it take for me to kick start?
```
A coffee and a 30 min read on below documentation should get you charged.
```

##### How do I efficiently get started?
```
Fastest way is to define your use case and find the right resource type.

A very succinct list of use cases that match the resource types are below:

1) Delay: Add sleep or delay

2) Resolver: manipulate input parameters, validate parameters, convert string to json
or from one datatype to other, add two parameters, ensure parameter values meet certain conditions,
generate random IDs for use with resource names, etc.

3) WaitForState: after creating a cloudformation resource you need to check it is setup correctly,
or consider the stack update successful if your application latency does not increase,
run some test cases after making a deployment and fail the update if tests fail etc.

4) GenericResource: Wan to manage an existing resource created outside cloudformation,
manage a resource not supported by cloudformation pre-defined resource types,
perform backups or post or pre actions during resource creation or updates etc.

5) Timer: You are testing a cloudformation stack and want to add Timeouts for Update actions etc.

Other combinations include:

*) Input Output Stores + any resource type above: read password from s3,
read attributes of a AWS resource, share data to other stacks
cross-region or cross-account etc.

*) Assume Roles + any resource type above: Want to run api calls
which require assuming different IAM roles for access,
eg: cross account access or restricted roles for deleting or updating resources etc.
```

##### I do not want to spend time reading the documentation. How can I quick start?
```
The best way is to 'Setup' the project (follow instructions below)
and use "bash crh_helper.sh generate-template" command to generate
a sample template for you and work from there.

If you have any queries before using the tool,
feel free to raise an issue.
```

##### Wait, Can I test GCR locally without running an actual CFN stack?
```
crh_helper.sh script is exactly for this purpose.
You can use 'bash crh_helper.sh runlocal' command and
pass your template and resource name to the command and
you will be able to test the template locally
```

##### Do I need heavy programming skills to use this project?
```
'NO'. GCR abstracts you from all the management jargon like sending signals,
exception handling, timeouts, retries, making api calls in the backend,
downloading or uploading stores, logging, re-invoking lambdas etc.
```

##### Can I write my own programs / logic inside cloudformation template?
```
'CRH::Python' intrinsic function and 'ScriptExecutor' ServiceType are designed
for intermediate to advanced use cases like this.

Both accept python code as arguments and accept return values from the code.

Samples are available for these use case. It is really fun to write
these small snippets and see how powerful they are to give you the desired results.
```

##### How do I view logs of my custom resource execution?
```
Use command line tool 'bash crh_helper.sh logs'
```

##### I am using WaitForState and GenericResource in GCR. How do I find which api calls I need to use for my resource?
```
Two options:

1) bash crh_helper.sh generate-template
<choose GenericResource or WaitForState>

The above command lets you choose the AWS service, API, Optional Arguments and gives you a workable template for you to fill values in and start testing / using the template.

2) GCR is completely based on python2.7 and boto3 library. So all the services and API calls are documented in http://boto3.readthedocs.io/en/latest/reference/services/
```

##### Do I incur addition AWS costs?
```
The solution deploys a lambda function in your account.
AWS provides free tier pricing for Lambda invocations per month.
For details of lambda pricing see Lambda Pricing in AWS documentation.

If you are using 'S3Outputs' and 'S3OutputStore' data will be written to S3
For details see AWS S3 Pricing.

Similarly if you are using 'S3InputStores' data will be read from S3
For details see AWS S3 Pricing.

GCR lambda function also writes logs to cloudwatch.
For details see AWS Cloudwatch Pricing.
```

##### What if I have other questions around using the tool or need a new feature?
```
Please raise an issue and we will address the queries
and new feature requests ASAP.
Alternatively, you are welcome to contribute to the project.
```

<a name="Setup"></a>
#### Setup

<a name="InitialSetup"></a>
##### Initial

The below setup will create a cloudformation stack and export named __CustomResourceHelper__ which is used in future templates. This export points to the __Alias__ of the lambda function created as part of the template.

__Security:__ The basic build template adds __AdministratorAccess__ policy to the lambda execution [role](https://docs.aws.amazon.com/lambda/latest/dg/intro-permission-model.html#lambda-intro-execution-role). Customize the template to remove this policy and add policies/rules that you desire or look at using [Assume Roles](#AssumeRoles) for refined access control. Do not remove other policies attached to the lambda function like AWSLambdaBasicExecutionRole, S3CrudPolicy (specific bucket), CloudFormationDescribeStacksPolicy as they are required for basic GCR functioning.

```
[projectdir] $ git clone <projecturl>
[projectdir] $ cd cfn-generic-customresource
[projectdir/cfn-generic-customresource] $ bash crh_helper.sh deploy-crh <s3bucketname> --region <regionname>
```

<a name="Deployments"></a>
##### Deployments

To pull the latest changes and deploy.

```
[projectdir/cfn-generic-customresource] $ git pull origin master
[projectdir/cfn-generic-customresource] $ bash crh_helper.sh deploy-crh <s3bucketname> --region <regionname>
```

<a name="HowGCRWorks"></a>
## How GCR Works

GCR is a custom resource in the backend, and hence fits into cloudformation template like any other pre-defined resource.

The main components required for GCR to work is the lambda function and the [resource properties](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/crpg-ref-requests.html).

When you setup this project, you will have a cloudformation stack which deploys the __GCR Lambda function__ and creates an export pointing to Lambda Arn/Alias with name __CustomResourceHelper__. We use this export name to point all our existing or new custom resources to using [__ServiceToken__](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cfn-customresource.html#w2ab2c21c10d182c13) property of the custom resource.

```
Resources:
  MyResource:
    Type: Custom::ResourceType
    Properties:
      ServiceToken: !ImportValue CustomResourceHelper  # --> All the magic happens here.
      GCRProperty: Go to Sleep                         # --> Define rest of the GCR properties
```

This lambda function is implemented using __Python 2.7__. This lambda function has all the code necessary to work with different resource types that GCR provides. When the custom resource using this lambda function is executed, Cloudformation sends an Event object to this lambda function with all the properties from the template along with its [Type](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cfn-customresource.html#aws-cfn-resource-type-name).
