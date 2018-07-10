
<a name="Setup"></a>
# Setup

<a name="Prerequisites"></a>
## Prerequisites

[aws-cli](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
```
sudo pip install awscli
```
[Create an S3 bucket to upload lambda source code](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/create-bucket.html)
```
aws s3 mb s3://cfnx-repository-<account-id>
```
<a name="Deployment"></a>
## Deployment
```
[cfnx] $ git clone <url>
[cfnx] $ bash cfnxcmd deploy-cfnx
S3 bucket [None]: <provide bucket name>
...
```
Above command will setup:
- Cloudformation Stack with name __awscfn-extension-gcr-DONOTDELETE__
- Macro named __CFNX__
- Export named __CFNXGenericCustomResource__ which points to the lambda function (gcr)

<a name="CLISetupUsage"></a>
## CLI Setup and Usage

**CLI Features**
  * Test transform templates locally
  * Run transform using sam local
  * Print debug messages (prints sensitive data as well)
  * Print Execution Order of resources (defaut in cli mode)
  * Generates cloudformation cli commands while testing templates for easy deployment (copy and paste)

__CFNX__ provides a CLI which makes it easy to test transform templates locally without having to deploy them to cloudformation or create change set every time.

It is recommended to use [virtualenvwrapper](http://virtualenvwrapper.readthedocs.io/en/latest/)] to avoid version compatibility issues

```
$ sudo pip install virtualenv virtualenvwrapper
$ source /usr/local/bin/virtualenvwrapper.sh
$ mkvirtualenv cfnx
```

Once installed setup CLI requirements as below:

```
(cfnx) $ pip install -r cli-requirements.pip
```

__MACOSX__ is known to have compatibility issues with six module.

```
pip install -r cli-requirements.pip --ignore-installed six
```

If you are still unable to setup the required modules, you can still work with __transform-sam-local__

## transform-local
[docs/samples/transform/cli-usage/basic.yaml](/docs/samples/transform/cli-usage/basic.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/cli-usage/basic.yaml
```

## --debug, --raw, --no-cli-mode, --no-dry-run, --no-aws-dry-run
```
--debug - outputs debug logs with all the data **BEWARE** of sensitive data in console output
--raw - executes the transforms with no try/catch exceptions. Used to debug any code issues.
--no-cli-mode - Stops printing resolution output objects to console
--no-dry-run - DryRun is enabled by default in CLI. Disable it.
--no-aws-dry-run - AWS api calls are run in DryRun mode by default. Disable it

```

```
bash cfnxcmd transform-local -t docs/samples/macro/cli-usage/basic.yaml --debug --raw --cli-mode
```

## transform-local-sam

Prerequisite: [Docker](https://docs.docker.com/) needs to be running locally

```
bash cfnxcmd transform-local-sam -t docs/samples/macro/cli-usage/basic.yaml
```

<a name="BasicTransformUsage"></a>

## Basic Transform Usage
__CFNX__ is declared at top level in the template. Similar to [AWS::Serverless-2016-10-31](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/transform-aws-serverless.html). It is not required nor appropriate to declare the transform at any other level.

```YAML
Transform: CFNX
```

[docs/samples/transform/setup/basic.yaml](/docs/samples/transform/setup/basic.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/setup/basic.yaml
```

```YAML
Transform: [CFNX]
```

[docs/samples/macros/setup/basic_list_transform.yaml](/docs/samples/macros/setup/basic_list_transform.yaml)

* For certain features we require stack name to be passed as follows

```YAML
Transform:
  - Name: CFNX
    Parameters:
      StackName: !Sub ${AWS::StackName}
```

[docs/samples/transform/setup/basic_transform_with_parameters.yaml](/docs/samples/transform/setup/basic_transform_with_parameters.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/setup/basic_transform_with_parameters.yaml
```

* Passing configuration to CFNX

```YAML
Transform:
  - Name: CFNX
    Parameters:
      StackName: !Sub ${AWS::StackName}
CFNXConfiguration:
  __GLOBALVARS__:
    ProductionTags:
      - Key: Environment
        Value: Production
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      Tags: $[GLOBAL::ProductionTags]
```

[docs/samples/transform/setup/basic_transform_cfnxconfiguration.yaml](/docs/samples/transform/setup/basic_transform_cfnxconfiguration.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/setup/basic_transform_cfnxconfiguration.yaml
```

* Using with other Transforms
```YAML
Transform:
  # Call CFNX initially
  - Name: CFNX
    Parameters:
      StackName: !Sub ${AWS::StackName}
  # Pass the transformed template to SAM
  - Name: AWS::Serverless-2016-10-31
  # Post process the template
  - Name: CFNX
    Parameters:
      StackName: !Sub ${AWS::StackName}
      FinalTemplateOverride: |
        - LogicalResourceIds: ['.+']
          Apply:
            DeletionPolicy: Retain
```

[docs/samples/transform/setup/multiple_transforms.yaml](/docs/samples/transform/setup/multiple_transforms.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/setup/multiple_transforms.yaml
```

<a name="CFNX"></a>
# CFNX Transform

<a name="TemplateProcessing"></a>
# Template Processing
CFNX Provides Rich template processing tools.

<a name="BuiltInMacros"></a>
## Built in Macros

Built in macros can be accessed using __$[MACRO::Literal]__ syntax.

_Note: Macros, Intrinsic functions and Includes can be used in any section of the template_

```YAML
Transform: CFNX
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      Tags:
        - Key: LastUpdatedAt
          Value: $[MACRO::DateTime]

Outputs:
  RandomUUID:
    Value: $[MACRO::RandomUUID]
```
[docs/samples/transform/funcs_macros/basic_sample.yaml](/docs/samples/transform/funcs_macros/basic_sample.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/basic_sample.yaml
```

Some of the macros referencing stack attributes require providing __StackName__ as part of parameters

```YAML
Transform:
  - Name: CFNX
    Parameters:
      StackName: !Sub ${AWS::StackName}
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-bucket-$[MACRO::StackSHASUM]
```
[docs/samples/transform/funcs_macros/basic_sample2.yaml](/docs/samples/transform/funcs_macros/basic_sample2.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/basic_sample2.yaml
```
##### Special Macros

- $[MACRO::SHASUM] - Provides first 10 characters of SHA256 of the string prefix

```YAML
string-to-create-shasum-from-$[MACRO::SHASUM]
---
string-to-create-macro-from-7e398284bf
```
- $[MACRO::StackSHASUM] - Provides first 10 characters of SHA256 of StackId. Use this to reuse the same template to create named resources, as this value will be highly unique and immutable per stack.

```YAML
BucketName: my-bucket-$[MACRO::StackSHASUM]
---
BucketName: my-bucket-8d4379b9ee
```

##### String Macros

- $[MACRO::RandomUUID]      - Returns random UUID (20f3d750-9291-4f0e-9da5-57bcbedc4aea)
- $[MACRO::RandomShortUUID] - Returns first 8 characters of random UUID (20f3d750)
- $[MACRO::StackUrl]        - Returns the URL for the stack to view from console
- $[MACRO::Region]          - Returns the region name
- $[MACRO::StackId]         - Returns the stack id
- $[MACRO::StackName]       - Returns the Name of the stack
- $[MACRO::AccountId]       - Returns the Account ID
- $[MACRO::RequestId]       - Returns the RequestId. Use this if you need unique value per stack deployment
- $[MACRO::DateTime]        - Current UTC time in string ISO format
- $[MACRO::StackStatus]     - Provides stack status. More appropriate usage of this would be while configuring runtime [Success](#CFNXSuccessActions) [Failure](#CFNXFailureActions) actions per resource

##### Number Macros

Number macros have a special property that if used with prefix or suffix it is converted to String and replaced into the value. When it has no prefix or suffix it returns value of Integer Type.

```YAML
ValueAsString: 'Current Epoch Seconds: $[MACRO::EpochSeconds]'
---
ValueAsString: 'Current Epoch Seconds: 1531068943'
```
```YAML
ValueAsNumber: $[MACRO::EpochSeconds]
---
ValueAsNumber: 1531068943
```

- $[MACRO::EpochSeconds]       - Returns current epoch seconds
- $[MACRO::RandomInteger]      - Returns a random integer between 1 - 32000
- $[MACRO::RandomBigInteger]   - Returns a random integer greater than 10^9
- $[MACRO::RandomShortInteger] - Returns a random integer between 1 - 127
[docs/samples/transform/funcs_macros/macros_basic.yaml](/docs/samples/transform/funcs_macros/macros_basic.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/macros_basic.yaml
```
<a name="IntrinsicFunctions"></a>
## Generic Intrinsic Functions

_Note: Macros, Intrinsic Functions, Includes can be used at any level in the template_

<a name="FnXType"></a>
- __FnX::Type__: Convert values from one data type to another. Eg: convert JSON string to JSON Object, or convert Number values to strings vice versa or even create 'None' values etc. Supported types are:

* Integer (int)
```YAML
FnX::Type:
  Value: '100'
  Type: int
---
100
```
* String (str)
```YAML
FnX::Type:
  Value: 100
  Type: str
---
'100'
```
* Float (float)
```YAML
FnX::Type:
  Type: float
  Value: '100.1'
---
100.1
```

* None (none) - Cloudformation does not allow you to pass 'null' values as part of template so you can use the conversion within the template for validations etc.

```YAML
FnX::Type:
  Type: none
  Value: 'none'
---
None
```
* Boolean (bool)
```YAML
FnX::Type:
  Type: bool
  Value: 100
---
True

---
True - True
False - False
0 - False
>0 - True
<0 - False
'T' | 't' | 'True' (any case) - True
Others - False
```

* JSON String from object (json)

```YAML
FnX::Type:
  Type: json
  Value:
    Key: Value
---
'{"Key": "Value"}'
```

* JSON Object from string (jsonobj)

```YAML
FnX::Type:
  Type: jsonobj
  Value: '{"Key": "Value"}'
---
{
    "Key": "Value"
}
```

* YAML String from object (yaml)

```YAML
FnX::Type:
  Type: yaml
  Value:
    Key: Value
---
"{Key: Value}\n"
```

* YAML Object from string
```YAML
FnX::Type:
  Type: yamlobj
  Value: "{Key: Value}\n"
---
Key: Value
```

* Using CFN Flip to JSON (cfnflipjson)
```YAML
FnX::Type:
  Type: cfnflipjson
  Value: "!Ref MyResource"
---
"{\n    \"Ref\": \"MyResource\"\n}"
```

* Using CFN Flip to YAML (cfnflipyaml)
```YAML
FnX::Type:
  Type: cfnflipyaml
  Value: '{"Fn::GetAtt": ["MyResource", "Arn"]}'
---
"!GetAtt 'MyResource.Arn'\n"
```

* Nested Types (nesting is allowed with any intrinsic function in CFNX)

```YAML
Value:
  FnX::Type:
    Value:
      FnX::Type:
        Value: |
          {"Key": "Value"}
        Type: cfnflipyaml
    Type: yamlobj
---
Value:
  Key: Value
```
[docs/samples/transform/funcs_macros/type_function.yaml](/docs/samples/transform/funcs_macros/type_function.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/type_function.yaml
```

<a name="FnXOperator"></a>
- __FnX::Operator__: Lets you make any valid operation between two operands. Apart from the operations supported by [operator](https://docs.python.org/2/library/operator.html) module CFNX provides support for __'iregex', 'ieq', 'ine', 'imemberof', 'icontains', 'memberof', 'regex'__.

_Note: Please note that only operations that take 2 operands are supported. For other advanced use cases see [FnX::Python](#FnXPython)_

```YAML
FnX::Operator: [Operand1, Operator, Operand2]
FnX::Operator: ['AWS', 'eq', 'AWS']
---
True
```

[docs/samples/transform/funcs_macros/operator_function.yaml](/docs/samples/transform/funcs_macros/operator_function.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/operator_function.yaml
```

<a name="FnXPython"></a>

- __FnX::Python__: Supports running arbitrary python code which returns a value as a result of the intrinsic function. Accepts any valid python program with a constraint that the last statement of the program should return a value. If you wish you return None, you still have to use 'return None'.

```YAML
FnX::Python: |
  return range(1, 10)
---
[1,2,3,4,5,6,7,8,9]
```

__Note__: Timeout for FnX::Python code execution is __30 seconds__

* __cmd__ is a pre baked function to run any arbitrary linux commands. The output of the command contains 3 values: ```[exit_code, output, error]```.

```YAML
FnX::Python: |
  exit_code, out, err = cmd("ls")
  return out
---
file1 file2 file3
```

__cmd__ also tries to convert the output to JSON object, and if it fails, returns the plain string.

```YAML
FnX::Python: |
  return cmd('echo {"Key": "Value"}')
---
[
    0,   --> exit code
    {
        "Key": "Value"
    },
    null --> error
]
```
* __boto3__ client makes it easy to run simple boto3 commands (considering 30 seconds time). However, if you require larger timeouts or read data from multiple boto3 api calls, use [Global Input Stores](#GlobalInputStores)

```YAML
FnX::Python: |
  result = boto3.client("cloudformation").describe_stack_resources(StackName="awscfn-extension-gcr-DONOTDELETE")
  return result['StackResources'][0]
---
{
    "LogicalResourceId": "CFNExtensionLambdaFunction",
    "ResourceType": "AWS::Lambda::Function",
    ... truncated ...
}
```

* __http_client__ makes it easy to run http api calls (considering 30 seconds time). However, if you require larger timeouts or read data from multiple http api calls, use [Global Input Stores](#InputStores)

Similar to __cmd__, __http_client__ also tries to convert result to JSON object before returning.

```YAML
ActiveOutages:
  FnX::Python: |
    return http_client.get_result('GET', 'https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/active_outages')
---
ActiveOutages:
  RESPONSE:
    active_outages: 0
  STATUS_CODE: 200
```
[docs/samples/transform/funcs_macros/python_function.yaml](/docs/samples/transform/funcs_macros/python_function.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/python_function.yaml
```

- __FnX::GetParam__: Query Parameters passed to to the template. This intrinsic function also supports macro usage __$[PARAM::ParameterName]__

<a name="WhyNotRef"></a>
**Why can't I use Ref instead?**: Cloudformation does not process or resolve __Ref__ or other CFN intrinsic functions before transform processing. Below are the reasons

1) __Ref__ may not always point to Parameters, as it may reference LogicalResourceIds as well.
2) Even though CFNX can parse all __Ref__ intrinsic functions and use templateParameterValues to resolve them, it is not appropriate to intrude with cloudformation processing as there could be other transforms dependent on __Ref__ values like Serverless (SAM).

**You cannot access protected values using this function**

Basic Syntax:

```YAML
Parameters:
  MyParameter:
    Type: String
CFNXConfiguration:
  GlobalVariables:
    MyParam:
      FnX::GetParam: MyParameter
    MyParamMacro: $[PARAM::MyParameter]
```

- __FnX::GetOldParam__: Query Parameters passed to to the template. This intrinsic function also supports macro usage __$[OLDPARAM::ParameterName]__

[Why can't I use Ref instead?](#WhyNotRef)

__OLDPARAM__ is used specially with validating templates to ensure the parameter values have not changed inappropriately. Check [Template Validations](#TemplateValidations)

**You cannot access protected values using this function**

Basic Syntax:

```YAML
Parameters:
  MyParameter:
    Type: String
CFNXConfiguration:
  GlobalVariables:
    MyParam:
      FnX::GetOldParam: MyParameter
    MyParamMacro: $[OLDPARAM::MyParameter]
```

- __FnX::Template__: Use [JQ](#UsingJQ) to query Values from Template object.

This is used specially with validating templates to ensure the template configuration has not changed inappropriately. Check [Template Validations](#TemplateValidations)

Basic Syntax:

```YAML
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket

Outputs:
  BucketType:
    Value:
      FnX::Template: .Resources.MyS3Bucket.Type
```
[docs/samples/transform/funcs_macros/template-function.yaml](/docs/samples/transform/funcs_macros/template-function.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/template-function.yaml
```

- __FnX::OldOriginalTemplate__: Use [JQ](#UsingJQ) to query Values from  Old Original Template object.

This is used specially with validating templates to ensure the template configuration has not changed inappropriately. Check [Template Validations](#TemplateValidations)

Basic Syntax:

```YAML
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket

Outputs:
  BucketType:
    Value:
      FnX::OldOriginalTemplate: .Resources.MyS3Bucket.Type
```
[docs/samples/transform/funcs_macros/old-original-template-function.yaml](/docs/samples/transform/funcs_macros/old-original-template-function.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/old-original-template-function.yaml
```

- __FnX::OldProcessedTemplate__: Use [JQ](#UsingJQ) to query Values from Old Processed Template object.

This is used specially with validating templates to ensure the template configuration has not changed inappropriately. Check [Template Validations](#TemplateValidations)

Basic Syntax:

```YAML
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket

Outputs:
  BucketType:
    Value:
      FnX::OldProcessedTemplate: .Resources.MyS3Bucket.Type
```
[docs/samples/transform/funcs_macros/old-processed-template-function.yaml](/docs/samples/transform/funcs_macros/old-processed-template-function.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/old-processed-template-function.yaml
```

<a name="FnXQuery"></a>

- __FnX::Query__: Query is a wrapper over [JQ](#UsingJQ) library and allows to query values from the passed in objects. Data is generally provided by other intrinsic functions.

```YAML
FnX::Query:
  Object:
    FnX::Python: |
      result = boto3.client("cloudformation").describe_stack_resources(StackName="awscfn-extension-gcr-DONOTDELETE")
      return result['StackResources']
  Query: '.[] | select(.LogicalResourceId == "CFNExtensionLambdaFunction") | .ResourceStatus'
---
UPDATE_COMPLETE
```

If you wish to run the same query for multiple properties, instead use [GlobalVariables](#GlobalVariables)

```YAML
...
CFNXConfiguration:
  __GLOBALVARS__:
    StackResources:
      FnX::Query:
        Object:
          FnX::Python: |
            result = boto3.client("cloudformation").describe_stack_resources(StackName="awscfn-extension-gcr-DONOTDELETE")
            return result['StackResources']
        Query: '.'
Resources:
  MyResource:
    Type: AWS::TypeFunction::Sample
    Properties:
      PartResultFromGlobalVariable:
        FnX::Query:
          Object: $[GLOBAL::StackResources]
          Query: '.[] | select(.LogicalResourceId == "CFNExtensionLambdaFunction") | .ResourceStatus'
```
[docs/samples/transform/funcs_macros/query_function.yaml](/docs/samples/transform/funcs_macros/query_function.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/query_function.yaml
```

<a name="GlobalVariables"></a>
## Global Variables

Global variables are handy to declare values at a single place and reuse them across the template. Global variables are declared inside __ GLOBALVARS __ of __CFNXConfiguration__. You can access the global variables using __$[GLOBAL::VarName]__ or __FnX::GetGlobalVariable__.

```YAML
Transform: CFNX
CFNXConfiguration:
  __GLOBALVARS__:
    ProductionTags:
      - Key: UpdatedAt
        Value: $[MACRO::DateTime]
    BucketName:
      FnX::Python: return 'MyBucket'
    SimpleValue: 100
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName:
        FnX::GetGlobalVariable: BucketName         # Using FnX::GetGlobalVariable
      Tags: $[GLOBAL::ProductionTags]              # Using $[GLOBAL::...]
      OtherProperty: $[GLOBAL::SimpleValue]
```

```YAML
Resources:
  MyS3Bucket:
    Properties:
      BucketName: MyBucket
      OtherProperty: 100
      Tags:
        - Key: UpdatedAt
          Value: 2018-07-08T18:27:56.328994 UTC
    Type: AWS::S3::Bucket
```

[docs/samples/transform/funcs_macros/global_variables.yaml](/docs/samples/transform/funcs_macros/global_variables.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/global_variables.yaml
```

_NOTE: You can use macros and other utility intrinsic functions (Type, Python, Query, Operator) inside GLOBALVARS. The only exception is you cannot use $[GLOBAL::...]_

<a name="GlobalInputStores"></a>
## Global Input Stores

Input Stores support downloding data from __S3, boto3, HTTP, Custom Python Script__ executions. Once the data is downloaded into the configured store, you can access the data by passing [JQ](#UsingJQ) queries to store intrinsic functions. __S3, boto3 and Script__ stores also support passing __AssumeRoleConfig__ to support for cross account data or privileged data access. You pass the input store configuration as part of __CFNXConfiguration__.

All Input Stores allow you to define a __StoreIdentifier__ under which you download the data. This allows you to define multiple input stores of different types.

_NOTE: All input store data providers should return a valid JSON or a YAML string_
_NOTE: Intrinsic functions and macros can be used to define input store configurations_

- __GlobalS3InputStores__: We provide the s3://bucket/key to the __StoreIdentifier__. In this case __StoreIdentifier__ is __S3Data__. We access the output data using __FnX::GlobalS3Store__ function.

```JSON
{
  "Tags": [
    {
      "Key": "Environment",
      "Value": "Dev"
    }
  ]
}
```
```YAML
Transform: CFNX
CFNXConfiguration:
  GlobalS3InputStores:
    Stores:
      S3Data: s3://mybucket/bucketconfig.json
      S3SecondaryData: s3://mybucket/bucketconfig.json
Resources:
  MyResource:
    Type: AWS::S3::SampleBucket
    Properties:
      Tags:
        FnX::GlobalS3Store: ".S3Data.Tags"
      SecondaryTags:
        FnX::GlobalS3Store: ".S3SecondaryData.Tags"
```
```YAML
Resources:
  MyS3Bucket:
    Properties:
      Tags:
        - Key: Environment
          Value: Dev
    Type: AWS::S3::Bucket
```

[docs/samples/transform/funcs_macros/global_s3_input_stores.yaml](/docs/samples/transform/funcs_macros/global_s3_input_stores.yaml)
```
<change bucket name in the template>
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/global_s3_input_stores.yaml
```

- __GlobalS3InputStores__ with __AssumeRoleConfig__
```YAML
GlobalS3InputStores:
  Stores:
    S3Data: s3://mybucket/bucketconfig.json
  AssumeRoleConfig:
    RoleArn: <cross-account-or-privileged-role-arn>
```
[docs/samples/transform/funcs_macros/global_s3_input_stores_assume_roles.yaml](/docs/samples/transform/funcs_macros/global_s3_input_stores_assume_roles.yaml)
```
<change bucket name in the template>
<provide role arn>
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/global_s3_input_stores_assume_roles.yaml
```

- __GlobalBoto3InputStores__: We provide a list of 2 elements to Boto3 Input stores, in the below format:

```
["awsservice/apiname", <optional arguments dictionary>]
---
["cloudformation/describe_stacks", {StackName: awscfn-extension-gcr-DONOTDELETE"}]
```

We access the output data using __FnX::GlobalBoto3Store__ function.

```YAML
Transform: CFNX
CFNXConfiguration:
  GlobalBoto3InputStores:
    Stores:
      StackData: [cloudformation/describe_stacks, {StackName: awscfn-extension-gcr-DONOTDELETE}]
Resources:
  MyResource:
    Properties:
      Configuration:
        FnX::GlobalBoto3Store: .StackData.Stacks[0].Outputs[] | select(.OutputKey == "CFNExtensionTransform") | .OutputValue
```
```YAML
Resources:
  MyResource:
    Properties:
      Configuration: ACCOUNTID::CFNX
```
[docs/samples/transform/funcs_macros/global_boto3_input_stores.yaml](/docs/samples/transform/funcs_macros/global_boto3_input_stores.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/global_boto3_input_stores.yaml
```

- __GlobalBoto3InputStores__ with __AssumeRoleConfig__

```YAML
Transform: CFNX
CFNXConfiguration:
  GlobalBoto3InputStores:
    Stores:
      StackData: [cloudformation/describe_stacks, {StackName: awscfn-extension-gcr-DONOTDELETE}]
    AssumeRoleConfig:
      RoleArn: <cross-account-or-privileged-role-arn>
...
```
[docs/samples/transform/funcs_macros/global_boto3_input_stores_assume_roles.yaml](/docs/samples/transform/funcs_macros/global_boto3_input_stores_assume_roles.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/global_boto3_input_stores_assume_roles.yaml
```
- __GlobalHTTPInputStores__: We can provide a simple url, a http method and url, a http method url and data to download the store data from and access the data using __FnX::GlobalHTTPStore__ function. The output of http stores is accessible in the below object format:

```json
{
  "STATUS": "<http_status_code>",
  "RESPONSE": "<json loaded object if possible else string>"
}
```

```YAML
Transform: CFNX
CFNXConfiguration:
  GlobalHTTPInputStores:
    Stores:
      JenkinsStore: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/jenkins-status-provider.json
Resources:
  MyResource:
    Properties:
      Status:
        FnX::GlobalHTTPStore: .JenkinsStore
```
```YAML
Resources:
  MyResource:
    Properties:
      Status:
        RESPONSE:
          jenkinsjobstatus: completed
        STATUS_CODE: 200
    Type: AWS::S3::Bucket
```
[docs/samples/transform/funcs_macros/global_http_input_stores.yaml](/docs/samples/transform/funcs_macros/global_http_input_stores.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/global_http_input_stores.yaml
```
- __GlobalHTTPInputStores__ with __POST__
```YAML
GlobalHTTPInputStores:
  Stores:
    JenkinsStore:
      - POST
      - <url>
```

- __GlobalHTTPInputStores__ with __POST__ and __DATA__
```YAML
GlobalHTTPInputStores:
  Stores:
    JenkinsStore:
      - POST
      - <url>
      - {'username': 'Bob'}
```

- __GlobalHTTPInputStores__ with __GET__ and __PARAMS__
```YAML
GlobalHTTPInputStores:
  Stores:
    JenkinsStore:
      - GET
      - <url>
      - {'username': 'Bob'}   --> will be translated to query params ?username=Bob
```

- __GlobalScriptInputStores__: Script input stores can be very logical in determining the state of the resource properties, like MinSize of AZ should be 2 times the number of Availability zones etc (sample below). We can also prepare arbitrary data by running Python scripts. Store Identifiers take a single value of type string which has the custom script.

The output of script stores is accessible in the below object format:

```json
{
  "EXIT_CODE": "<exit_code>",
  "OUTPUT": "<json loaded object if possible else string>"
}
```

Similar to [FnX::Python](#PythonFunction) below additional variables/clients are accessible within the code for convenience. We access the object using __FnX::GlobalScriptStore__ function.

```
boto3_session - pre baked boto3_session which is initialized with any AssumeRoleConfig passed (see second example below)
http_client - making http requests
cmd() - access to run shell commands
```

```YAML
Transform: CFNX
Parameters:
  AvailabilityZones:
    Type: CommaDelimitedList
    Default: ap-southeast-2a, ap-southeast-2b
CFNXConfiguration:
  GlobalScriptInputStores:
    Stores:
      CustomStore: |
        return len($[PARAM::AvailabilityZones]) * 2
Resources:
  MyAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      MinSize:
        FnX::GlobalScriptStore: .CustomStore.OUTPUT
```

[docs/samples/transform/funcs_macros/global_script_input_stores.yaml](/docs/samples/transform/funcs_macros/global_script_input_stores.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/global_script_input_stores.yaml  --parameters-file docs/samples/transform/data/sample_parameters.yaml
```

- __GlobalScriptInputStores__ with __AssumeRoleConfig__. With AssumeRoleConfig we use boto3_session which is pre baked session with Assume role credentials. For normal access you could still import boto3 and make simple boto3 calls.

```YAML
Transform: CFNX
CFNXConfiguration:
  GlobalScriptInputStores:
    Stores:
      CustomStore: |
        # boto3_session uses assume role config
        return boto3_session.client('cloudformation').describe_stacks()
    AssumeRoleConfig:
      RoleArn: <cross-account-or-privileged-role-arn>
```
[docs/samples/transform/funcs_macros/global_script_input_stores_assume_roles.yaml](/docs/samples/transform/funcs_macros/global_script_input_stores_assume_roles.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/funcs_macros/global_script_input_stores_assume_roles.yaml
```
<a name="TransformIncludes"></a>
## Template Transform Includes

__CFNX__ provides two include functions for including templates from __HTTP__ locations and processing __JINJA__ templates to generate template content. Both the functions allow specifying a single object or a list of objects which are processed in order.

* Input templates can be YAML / JSON formatted and can be of any valid JSON type including Integer, String, List, Dictionary, Boolean types. [Lists](#TemplateInheritanceLists) and [Dictionaries](#TemplateInheritanceDicts) have special configurable behavior.
* Input templates can contain both AWS and CFNX macros and intrinsic functions
* HTTP Templates are included before Jinja templates, so you can use Jinja to override sections of HTTP templates if need be (see sample below).
* Includes can be provided at any level in the template and the resultant JSON/YAML will be replaced in appropriate location.
* You can include multiple locations by passing a list of include configuration to the function.

<a name="HTTPTemplateInclude"></a>
## CFNX::Include::HTTPTemplate

The function takes the location of the HTTP template as part of __Location__ argument. We can provide basic authentication credentials to be used with downloading the data if required. These auth credentials can be provided at global level in __CFNXConfiguration/GlobalHTTPIncludeConfig__ or at individual include function level using __OverrideAuthConfig__ objects.

For example, in the below configuration, we are downloading Mappings section from an external URL and rest of the sections are defined within the template. Include functions add the downloaded template in the same level as where they have been declared, in this case it is the top level.

```YAML
Transform: [CFNX]
CFNX::Include::HTTPTemplate:
  Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/mappings.yaml
Resources:
  MyResource:
    ...
```
Transformed Template:
```YAML
Mappings:
  AWSInstanceType2Arch:
    c1.medium:
      Arch: PV64
    ...
Resources:
  MyResource:
    ...
```
[docs/samples/transform/includes/http_include.yaml](/docs/samples/transform/includes/http_include.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/includes/http_include.yaml
```

You can include HTTP templates at any level in the template as follows:

```YAML
MyIAMUser:
  Type: AWS::IAM::User
  Properties:
    CFNX::Include::HTTPTemplate:
      Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/iam-username.yaml
```
[docs/samples/transform/includes/http_include_any_level.yaml](/docs/samples/transform/includes/http_include_any_level.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/includes/http_include_any_level.yaml
```
The above snippet gets just the __UserName__ property from http url.

**If we look into the HTTP location we can see a macro defined as well.**

Contents of HTTP location:
```YAML
UserName: dev-user-$[MACRO::StackSHASUM]
```
Templates are downloaded after preparing __GLOBALVARS__ and __Global*InputStores__. This means you can use any macro/intrinsic function that references Input Stores in the templates and they will be processed as well.

* Using multiple HTTP includes. Below sample downloads iam deny policy and then ec2 allow policy

```YAML
MyIAMUser:
  Type: AWS::IAM::User
  Properties:
    CFNX::Include::HTTPTemplate:
      Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/iam-username.yaml
    Policies:
      - CFNX::Include::HTTPTemplate:
          Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/iam-user-policy-deny-iam.yaml
      - CFNX::Include::HTTPTemplate:
          Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/iam-user-policy-allow-ec2.yaml
```
[docs/samples/transform/includes/http_include_multiple.yaml](/docs/samples/transform/includes/http_include_multiple.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/includes/http_include_multiple.yaml
```

* Using Basic Auth with HTTP includes
```YAML
Transform: [CFNX]
CFNXConfiguration:
  GlobalHTTPIncludeAuthConfig:   #(Optional Global level)
    Username: bob
    Password: password
MyIAMPolicy:
  Type: AWS::IAM::Policy
  Properties:
    CFNX::Include::HTTPTemplate:
      OverrideAuthConfig:         #(Optional)
        Username: admin
        Password: password
      Location: <httpurl>
```
[docs/samples/transform/includes/http_include_basic_auth.yaml](/docs/samples/transform/includes/http_include_basic_auth.yaml)

<a name="TemplateInheritanceLists"></a>
**Inheritance and merging when included template (HTTP/Jinja) provides a JSON/YAML list**
___

Merging of lists has two options:
* (Default) Add downloaded template to previously downloaded templates in the same block even if it is a duplicate

Below template first adds universal allow actions, followed by deny to iam, cloudwatchlogs, database services. As you can the database deny yaml has iam deny included as well. The resultant output will have two deny actions for IAM.

```YAML
MyIAMPolicy:
  Type: AWS::IAM::Policy
  Properties:
    PolicyDocument:
      Statement:
        CFNX::Include::HTTPTemplate:
          - Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/all-allow-actions-list.yaml
          - Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/deny-iam-cloudwatchlogs-actions-list.yaml
          - Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/deny-iam-database-action-list.yaml
```

```YAML
Resources:
  MyIAMPolicy:
    Properties:
      PolicyDocument:
        Statement:
          - Action:
              - '*'
            Effect: Allow
            Resource: '*'
          - Action:
              - iam:*
            Effect: Deny
            Resource: '*'
          - Action:
              - logs:*
            Effect: Deny
            Resource: '*'
          - Action:
              - rds:*
              - elasticsearch:*
            Effect: Deny
            Resource: '*'
          - Action:
              - dynamodb:*
              - redshift:*
            Effect: Deny
            Resource: '*'
          - Action:
              - iam:*
            Effect: Deny
            Resource: '*'
    Type: AWS::IAM::Policy
```

* (Optional 2) __MergeIntoPrevious__: Add only if previous block does not already have the list element included. For example, you have some default tags managed at organization level which you want to be applied, but there are environment specific tags which are applied to specific resources. There could be duplicate key value pairs in both the configurations which you want to avoid while merging.

_NOTE: Available for Jinja template includes as well._

```YAML
production-tags.yaml

- Key: Environment
  Value: Production
- Key: Logging
  Value: Enabled
```

```YAML
production-tags.yaml

- Key: Environment
  Value: Production
- Key: Logging
  Value: Enabled
```

```YAML
default-tags.yaml

- Key: Logging
  Value: Enabled
- Key: cost-allocation
  Value: quotas
```

```YAML
MyS3Bucket:
  Type: AWS::S3::Bucket
  Properties:
    Tags:
      CFNX::Include::HTTPTemplate:
        - Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/production-tags.yaml
        -
          Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/default-tags.yaml
          MergeIntoPrevious: true
```

Result is __Logging__ tag is not duplicated when you enable __MergeIntoPrevious__
```YAML
MyS3Bucket:
  Properties:
    Tags:
      - Key: Environment
        Value: Production
      - Key: Logging
        Value: Enabled
      - Key: cost-allocation
        Value: quotas
  Type: AWS::S3::Bucket
```

<a name="TemplateInheritanceDicts"></a>
**Inheritance and merging when included template (HTTP/Jinja) provides a JSON/YAML dictionaries**
___
Merging of dictionaries has two options:
* (Default) Add top level keys previously downloaded templates in the same block only if it not already exists.

```YAML
bucket-name.yaml

BucketName: MyBucket
```

```YAML
bucket-properties.yaml

BucketName: PlaceholderBucketName
Tags:
  - Key: Logging
    Value: Enabled
```

```YAML
MyS3Bucket:
  Type: AWS::S3::Bucket
  Properties:
    CFNX::Include::HTTPTemplate:
      - Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/bucket-name.yaml
      - Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/bucket-properties.yaml
    OtherProperty: value
```

Result:
```YAML
MyS3Bucket:
  Properties:
    BucketName: MyBucket
    Tags:
      - Key: Logging
        Value: Enabled
    OtherProperty: Value
  Type: AWS::S3::Bucket
```

[docs/samples/transform/includes/http_include_dict_merging_if_not_exists.yaml](/docs/samples/transform/includes/http_include_dict_merging_if_not_exists.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/includes/http_include_dict_merging_if_not_exists.yaml
```

* (Option 2) __MergeIntoPrevious__: Even if the key already exist in previously downloaded object, overwrite it.

_NOTE: Available for Jinja template includes as well._

```YAML
bucket-tags.yaml

Tags:
  - Key: Logging
    Value: Disabled
```
```YAML
MyS3Bucket:
  Type: AWS::S3::Bucket
  Properties:
    CFNX::Include::HTTPTemplate:
      - Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/bucket-properties.yaml
      -
        Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/bucket-tags.yaml
        MergeIntoPrevious: true
```
Result:
```YAML
MyS3Bucket:
  Properties:
    BucketName: PlaceholderBucketName
    Tags:
      - Key: Logging
        Value: Disabled
  Type: AWS::S3::Bucket
```
[docs/samples/transform/includes/http_include_dict_merging_overwrite.yaml](/docs/samples/transform/includes/http_include_dict_merging_overwrite.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/includes/http_include_dict_merging_overwrite.yaml --account-id 999
```

<a name="JinjaTemplateInclude"></a>
## CFNX::Include::JinjaTemplate

[Jinja templates](http://jinja.pocoo.org/docs/2.10/) use two parameters:

- (Required) __Template__: which contains the Jinja code
- (Optional) __OverrideContext__: which overrides __CFNXConfiguration/GlobalJinjaIncludeContext__

The Context objects are used to provide data to Jinja templates for generating the final template.

```YAML
Transform: [CFNX]
CFNXConfiguration:
  GlobalJinjaIncludeContext:
    BucketNames: [MyDevBucket, MyStagingBucket, MyProductionBucket]
    GlobalTags:
      - Key: CreatedBy
        Value: CFNX
Resources:
  CFNX::Include::JinjaTemplate:
    Template: |
      {% for i in BucketNames %}
        {{i}}:
          Type: AWS::S3::Bucket
          Properties:
            Tags:
              {{ GlobalTags | tojson }}
      {% endfor %}
```
Result:
```YAML
Resources:
  MyDevBucket:
    Properties:
      Tags:
        - Key: CreatedBy
          Value: CFNX
    Type: AWS::S3::Bucket
  MyProductionBucket:
    Properties:
      Tags:
        - Key: CreatedBy
          Value: CFNX
    Type: AWS::S3::Bucket
  MyStagingBucket:
    Properties:
      Tags:
        - Key: CreatedBy
          Value: CFNX
    Type: AWS::S3::Bucket
```
[docs/samples/transform/includes/jinja_include.yaml](/docs/samples/transform/includes/jinja_include.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/includes/jinja_include.yaml --account-id 999
```

* Overriding context locally

```YAML
CFNX::Include::JinjaTemplate:
  OverrideContext:
    GlobalTags:
      - Key: CreatedBy
        Value: Jinja
  Template: |
    {% for i in BucketNames %}            # BucketNames come from global context
      {{i}}:
        Type: AWS::S3::Bucket
        Properties:
          Tags:
            {{ GlobalTags | tojson }}     # Tags come from local context
    {% endfor %}
```

* Similar to HTTP Includes you can chain Jinja Includes using a list

```YAML
Transform: [CFNX]

CFNXConfiguration:
  GlobalJinjaIncludeContext:
    BucketNames: [MyDevBucket, MyStagingBucket, MyProductionBucket]
    GlobalTags:
      - Key: CreatedBy
        Value: CFNX

Resources:
  CFNX::Include::JinjaTemplate:
    -
      Template: |
        {% for i in BucketNames %}
          {{i}}:
            Type: AWS::S3::Bucket
            Properties:
              Tags:
                {{ GlobalTags | tojson }}
        {% endfor %}
    -
      OverrideContext:
        BucketNames: [MyQABucket1, MyQABucket2]
      Template: |
        {% for i in BucketNames %}
          {{i}}:
            Type: AWS::S3::Bucket
            Properties:
              Tags:
                {{ GlobalTags | tojson }}
        {% endfor %}
```
[docs/samples/transform/includes/jinja_include_multiple.yaml](/docs/samples/transform/includes/jinja_include_multiple.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/includes/jinja_include_multiple.yaml --account-id 999
```

* Inheritance of [Lists](#TemplateInheritanceLists) and [Dictionaries]((#TemplateInheritanceDicts) in Jinja Includes work exactly the same as HTTP Includes

* Jinja templates have access to [built-in filters](#http://jinja.pocoo.org/docs/2.10/templates/#list-of-builtin-filters). Along with these, CFNX provides custom filters like __tojson__, __toyaml__, __cfn_flip_json__, __cfn_flip_yaml__.

* CFNX provides all [python builtins](https://docs.python.org/2/library/functions.html) and [operators](https://docs.python.org/2/library/operator.html) for use within Jinja Templates.

* CFNX Provides __special filters__ to work with __Tag__ and __TagFilter__ properties for resources.

__merge_into_tags__: Declare tags specific to the resource in override context and merge them into tags. The filter internally checks __Key__ property of tags to merge the values appropriately

```YAML
CFNXConfiguration:
  GlobalJinjaIncludeContext:
    GlobalTags:
      - Key: Environment
        Value: Non Production
      - Key: Logging
        Value: Enabled

Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      Tags:
        CFNX::Include::JinjaTemplate:
          OverrideContext:
            ProductionTags:
              - Key: Environment
                Value: Production
          Template: |
              {{ ProductionTags | merge_into_tags(GlobalTags) | tojson }}
```
Resultant tags:
```YAML
Tags:
  - Key: Environment
    Value: Production       # Value used from Override Context
  - Key: Logging
    Value: Enabled
```
[docs/samples/transform/includes/jinja_include_merge_tags.yaml](/docs/samples/transform/includes/jinja_include_merge_tags.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/includes/jinja_include_merge_tags.yaml --account-id 999
```

__aws_tags__: Allows you to declare Key/Value object of Tags and the filter converts it into List of {Key,Value} objects

```YAML
CFNX::Include::JinjaTemplate:
  OverrideContext:
    ProductionTags:
      Environment: Production
      Logging: Enabled

  Template: |
      {{ ProductionTags | aws_tags | tojson }}
```
Resultant tags:
```YAML
Tags:
  - Key: Environment
    Value: Production
  - Key: Logging
    Value: Enabled
```
[docs/samples/transform/includes/jinja_include_aws_tags.yaml](/docs/samples/transform/includes/jinja_include_aws_tags.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/includes/jinja_include_aws_tags.yaml --account-id 999
```

__append_lists__: Allows you to add multiple lists and process them at once instead of writing multiple for loops for each list. Below we are merging __BusinessTags__, __LoggingTags__, __ProductionTags__ to generate the final list of tags for the resource.

```YAML
Transform: [CFNX]

CFNXConfiguration:
  GlobalJinjaIncludeContext:
    BusinessTags:
      - Key: Business
        Value: Cloudformation
    LoggingTags:
      - Key: Logging
        Value: Enabled
...
    Properties:
      Tags:
        CFNX::Include::JinjaTemplate:
          OverrideContext:
            ProductionTags:
              - Key: Environment
                Value: Production
          Template: |
              {{ ProductionTags | append_lists(BusinessTags, LoggingTags) | tojson }}
```

[docs/samples/transform/includes/jinja_include_append_lists.yaml](/docs/samples/transform/includes/jinja_include_append_lists.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/includes/jinja_include_append_lists.yaml --account-id 999
```

__until__ filter lets you generate range of numbers. Below template creates 10 S3 buckets.

```YAML
Transform: [CFNX]
Resources:
  CFNX::Include::JinjaTemplate:
    Template: |
      {% for i in 1 | until(10) %}
        MyS3Bucket{{i}}:
          Type: AWS::S3::Bucket
      {% endfor %}
```
```
bash cfnxcmd transform-local -t docs/samples/transform/includes/jinja_include_until_function.yaml --account-id 999
```

* Jinja templates have a special feature that it can override specific sections of the templates downloaded with HTTP in the same block. This is because Jinja templates are processed after all HTTP templates are processed.

For example, we download all resources for the template using HTTP, but change the configuration of __RolePolicies__ resource to suit our requirements. This again can work at any level in the template.

```YAML
Transform: [CFNX]
Resources:
  CFNX::Include::HTTPTemplate:
    Location: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/iam-roles-yaml

  ### Above includes RootRole, RolePolicies, RootInstanceProfile
  # We override RolePolicies to make it more strict for this stack
  CFNX::Include::JinjaTemplate:
    Template: |
      RolePolicies:
        Type: "AWS::IAM::Policy"
        Properties:
          PolicyName: "root"
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              -
                Effect: "Allow"
                Action: "ec2:describe*"
                Resource: "*"
          Roles:
            -
              Ref: "RootRole"
```
[docs/samples/transform/includes/jinja_http_include_override_with_jinja.yaml](/docs/samples/transform/includes/jinja_http_include_override_with_jinja.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/includes/jinja_http_include_override_with_jinja.yaml --account-id 999
```

* JSON Sample:

[docs/samples/transform/includes/jinja_include.json](/docs/samples/transform/includes/jinja_include.json)
```
bash cfnxcmd transform-local -t docs/samples/transform/includes/jinja_include.json --account-id 999
```

* Advanced Sample (not deployable but for reference):

[docs/samples/transform/includes/jinja_include_advanced_for_demo_only.yaml](/docs/samples/transform/includes/jinja_include_advanced_for_demo_only.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/includes/jinja_include_advanced_for_demo_only.yaml --account-id 999
```

<a name="TemplateOverride"></a>
## Overriding Template Sections
We can override different sections of the template either in the beginning of CFNX execution __InitialTemplateOverride__ or towards the end of all processing __FinalTemplateOverride__. Both the properties provide rich configuration options with optimal defaults.

<a name="FinalTemplateOverride"></a>
__FinalTemplateOverride__ makes sense to start with as the other option is used for Hybrid transform configurations which we will discuss post this.

**Overriding Resources Section**
---

The below configuration, overrides __Resources__ block by applying rules in te list in order, to add __DeletionPolicy: Retain__ to all resources matching ResourceType __AWS::S3::.*__ and __DeletionPolicy: Snapshot__ to all resources matching LogicalResourceIds match __MyRDS.* or MyRedShift.*__. To apply to all resources use __.*__.

```YAML
CFNXConfiguration:
  FinalTemplateOverride:
    Resources:  #--------> Section to override
      -
        ResourceTypes: ['AWS::S3::.*']
        Apply:
          DeletionPolicy: Retain
      -
        LogicalResourceIds: ['MyRDS.*', 'MyRedShift.*']
        Apply:
          DeletionPolicy: Snapshot
```

However, if an S3 bucket already has __DeletionPolicy: Delete__ for example, then it won't be touched. To force all Properties to match the rule use __Type: MERGE__

```YAML
CFNXConfiguration:
  FinalTemplateOverride:
    Resources:
      -
        Type: MERGE
        ResourceTypes: ['AWS::S3::.*']
        Apply:
          DeletionPolicy: Retain
```

You can control the behavior of how the properties are merged/overwritten using 3 different Types:

__ADD_IF_MISSING__(Default): This is the default mode, where properties in __Apply__ section are added to matched resources only if they do not already exist, including lists. __Tag__ properties are automatically handled to check for __Key__ value in the tag list to check for existence.

__MERGE__: In this mode, properties in __Apply__ section are added to matched resources if they do not already exist, and existing property values are forced to use the new values. Lists elements are added to existing lists if they exist.

__OVERWRITE__: This mode takes a slight different type of configuration. If we want to mention nested properties we use __Properties/SubProperty/SubSubProperty__ syntax and provide the value that should be overwritten into the specific path. Lists are not supported. If the resource property is a list whereas the __Apply__ section provides a dict, a dict is created and overwritten into Resource properties.

For example, below configuration will overwrite entire __Properties/Tags__, __DeletionPolicy__, __UpdatePolicy/AutoScalingReplacingUpdate__ to match the rule for all __LogicalResourceIds__ matching the pattern.

```YAML
CFNXConfiguration:
  FinalTemplateOverride:
    Resources:
      -
        Type: OVERWRITE
          LogicalResourceIds: ['MyAutoScaling.*']
          Apply:
            Properties/Tags:
                - Key: OnlyOneTag
                  Value: overwritten
            DeletionPolicy: Retain
            UpdatePolicy/AutoScalingReplacingUpdate:
              WillReplace: true
```

Apart from __ResourceTypes__ and __LogicalResourceIds__ we can also add further filtering using __Conditions__. Conditions are applied to resources that match other two properties only.

For example, the below rule, applies to all RDS instances, who have a tag __Environment__ with value __Production__. For more complex conditions involving multiple AND OR NOT statements refer to [Conditions](#Conditions)

```YAML
Type: OVERWRITE
ResourceTypes: ['AWS::RDS::DBInstance']
Conditions:
  HasProductionTag: ['.Properties.Tags[] | select(.Key=="Environment") | .Value', 'eq', 'Production']
Apply:
  DeletionPolicy: Snapshot
```
* Overwriting non Resource Blocks does not support any filters. It supports __Type__ property which can be configured. For example you can override __Outputs__ section as follows:

```YAML
FinalTemplateOverride:
  Outputs:
    -
      Apply:
        NewOutput:
          Value: $[MACRO::RandomUUID]
```

[docs/samples/transform/cfnx_configuration/override-sections.yaml](/docs/samples/transform/cfnx_configuration/override-sections.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/cfnx_configuration/override-sections.yaml
```

<a name="FinalTemplateOverride"></a>
## InitialTemplateOverride

_(For Advanced use cases)_

Similar to FinalTemplateOverride, InitialTemplateOverride lets you override Template sections before Transform Processing (early in the stage).

The best use case for this is when working with multiple transforms and if you need to call __CFNX__ multiple times while creating change sets, you can use __InitialTemplateOverride__ to configure CFNX properties for resources for each run. This is because, when CFNX processes the template, it removes the __CFNXConfiguration__ block in the Result template as it is not a valid cloudformation Section.

When using SAM templates, run CFNX first to pass values to SAM template using intrinsic functions/macros/inputstores etc. Once SAM processes the template, Apply new configuration resources created by SAM templates.

Since, we cannot use __CFNXConfiguration__ more than once, we need to pass all the configuration as part of __Transform Parameters__. All properties Supported by __CFNXConfiguration__ are supported in Parameters with one __limitation__. Cloudformation does not allow passing Parameters with values other than basic data types. To help with this situation, __CFNX__ allows you to pass the value of every config parameter as either a JSON/YAML string. This way we can still pass configuration for CFNX to work with.

Eg:

- Call CFNX first to resolve template functions variables and other features
- Call AWS::Serverless-2016-10-31 to create lambda resource definitions
- Call CFNX to add
  - [CFNXPostExecuteDelay](#CFNXPostExecuteDelay) runtime property to add 10 seconds delay after updates to IAM policies generated by SAM
  - Also add __DeletionPolicy__ as Retain to all Lambda resources, by passing YAML config to __FinalTemplateOverride__.

Here is the flow:

- CFNX will process __InitialTemplateOverride__ first and add __CFNXPostExecuteDelay__ to all IAM Policies
- CFNX will process resources and find out that IAM policies have __CFNXPostExecuteDelay__ enabled, so it starts applying delay resources to the template.
- CFNX will process __FinalTemplateOverride__ in the end and apply Deletion Policy to all Lambda resources.

You can similarly configure any CFNX configuration for that run.

```YAML

Transform:
  - Name: CFNX
    Parameters:
      StackName: !Sub ${AWS::StackName}
  - Name: AWS::Serverless-2016-10-31
  - Name: CFNX
    Parameters:
      InitialTemplateOverride: |
        Resources:
          -
            ResourceTypes: ["AWS::IAM::Policy"]
            Apply:
              CFNXPostExecuteDelay: 10
      FinalTemplateOverride: |
        Resources:
          -
            ResourceTypes: ["AWS::Lambda::*"]
            Apply:
              DeletionPolicy: Retain
Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: mylambda-$[MACRO::StackSHASUM]
      Handler: index.handler
      Runtime:
        FnX::Python: return 'python2.7'
      CodeUri: hello_world_lambda
```

<a name="StackLevelProperties"></a>
# Stack Level Configuration

__CFNXConfiguration__ provides certain properties that can be applied at stack level. These include:

<a name="EnableTemplateBackups"></a>
## Enable Template Backups
```YAML
Transform:
  - Name: CFNX
    Parameters:
      StackName: !Sub ${AWS::StackName}
CFNXConfiguration:
  EnableTemplateBackups: true
Resources:
...
```

The above configuration will keep a backup of __Unprocessed__ and __Transformed__ templates in S3 automatically for every run. The S3 bucket used for backups is __awscfn-extension-gcr-ACCOUNTID-REGION-do-not-delete__ which is created as part of the setup process.

[docs/samples/transform/cfnx_configuration/enable_template_backups.yaml](/docs/samples/transform/cfnx_configuration/enable_template_backups.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/cfnx_configuration/enable_template_backups.yaml

aws cloudformation deploy --template-file docs/samples/transform/cfnx_configuration/enable_template_backups.yaml --stack-name enable-template-backups-yaml-1
```

When you deploy the template to cloudformation you will see a s3 backups created inside __awscfn-extension-gcr-ACCOUNTID-REGION-do-not-delete/cfnx-backups/REGION/STACKID/STACKGUID/<timestamp>-unprocessed__ and
__<timestamp>-transformed__

<a name="EnableNOOPUpdates"></a>
## Enable NOOP Updates

Specially in CI/CD integration where builds run from multiple sources and get deployed to multiple stacks, you may need to do a stack update without modifying the configuration of resources.

```YAML
Transform:
  - Name: CFNX
    Parameters:
      StackName: !Sub ${AWS::StackName}
CFNXConfiguration:
  EnableNOOPUpdates: true
Resources:
... NO CHANGES ...
```

The above configuration will change the value of __CFNXNOOPUpdateExecutionTimestamp7c6bf__ output adding the current timestamp including milli seconds. This will be considered as a valid update for the cloudformation stack.

[docs/samples/transform/cfnx_configuration/enable_noop_updates.yaml](/docs/samples/transform/cfnx_configuration/enable_noop_updates.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/cfnx_configuration/enable_noop_updates.yaml

aws cloudformation deploy --template-file docs/samples/transform/cfnx_configuration/enable_noop_updates.yaml --stack-name enable-noop-updates-yaml-1
```
You can run the above "deploy" command multiple times with no changes to template and still see the stack update successful.

<a name="SetStackUpdateTimeout"></a>
## SetStackUpdateTimeout

Cloudformation supports [stack creation timeout](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-add-tags.html#Timeout). However, if you want to set timeouts on Stack Updates, you can configure this:

**Max Timeout allowed is 3500 seconds**

```YAML
Transform: [CFNX]
CFNXConfiguration:
  SetStackUpdateTimeout: 10
Resources:
...
```
[docs/samples/transform/cfnx_configuration/enable_stack_update_timeout.yaml](/docs/samples/transform/cfnx_configuration/enable_stack_update_timeout.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/cfnx_configuration/enable_stack_update_timeout.yaml

aws cloudformation deploy --template-file docs/samples/transform/cfnx_configuration/enable_stack_update_timeout.yaml --stack-name enable-stack-update-timeout-yaml-1
```

The above sample introduces a sleep resource with 120 seconds and timeout of 10 seconds for demonstration.

<a name="ConcurrencyControl"></a>
## ConcurrencyControl

When you have multiple resources created/updated in a stack you might have to control the concurrency of number of resources being created/updated in parallel. This is to avoid __Throttling__ exceptions.

```YAML
Transform: [CFNX]
CFNXConfiguration:
  ConcurrencyControl: true
Resources:
  ...
```

The default configuration of __true__ will ensure that no more than __MaxConcurrency__ (Default: 5) number of resources of the same ResourceType can execute in parallel. This means, not more than 5 S3 resources, or more than 5 IAM resource etc can be in IN_PROGRESS state at once.

__How this works__
---
CFNX groups resources of same type (AWS::S3, AWS::IAM, Custom::) and batches them into multiple groups based on __MaxConcurrency__.

- Say __Group1__ has 5 resources IAMR1, IAMR2, IAMR3, IAMR4, IAMR5.
- All resources in __Group2__ are made dependent on one of the randomly selected __Group1__ resources, say __R3__. To make __Group2__ dependent on all IAMR1-IAMR5 resources, use __StrictConcurrency: true__ in configuration (see below). However, it is not recommended to use strict concurrency when you have huge number of resources in the template as it increases the total execution time by considerable amount.
- Once __R3__ finished execution, __Group2__ starts execution
- This means that when __Group2__ is executing IAMR1, IAMR2, IAMR4, IAMR5 may still be executing, but this is very unlikely to happen as similar resources usually finish around the same time.
- __Group3__ (if exists) is similarly configured to be dependent on __Group2__

_Note: ConcurrencyControl only limits the maximum concurrency and does not ensure minimum concurrency of any value_

[docs/samples/transform/cfnx_configuration/concurrency-control.yaml](/docs/samples/transform/cfnx_configuration/concurrency-control.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/cfnx_configuration/concurrency-control.yaml

aws cloudformation deploy --template-file docs/samples/transform/cfnx_configuration/concurrency-control.yaml --stack-name concurrency-control-yaml-1
```

The above deployment will create 20 SSM parameters with maximum concurrency of 5. To set other configuration parameters use below method:

```YAML
CFNXConfiguration:
  ConcurrencyControl:
    MaxConcurrency: 2             # --> Maximum parallel executions
    NameSpaces: ['AWS::SSM']      # --> Apply to only selected namespaces
    StrictConcurrency: True       # ---> Make every group dependent on every resource in previous group
    EnableNamespaceGrouping: False # ---> Disable namespace group and consider all resources as part of same namespace
```

<a name="TemplateValidations"></a>
## TemplateValidations
<a name="TemplatePreValidations"></a>
## Pre Validations

**PreRequisites: [Quick reference to Conditions usage in CFNX](#Conditions)**

Validations help you define a group of __Validations__ with each Validation accepting __Conditions__ that evaluate to True/False.

Generic Syntax:

```YAML
CFNXConfiguration:
  PreValidations:
    ValidateStorageConfiguration:
      Conditions:
        EnsureSizeNotReduced: [$[PARAM::StorageSize], 'ge', $[OLDPARAM::StorageSize]]
        EnsureTypeNotChanged: [$[PARAM::StorageType], 'eq', $[OLDPARAM::StorateType]]
```

All the Validations should be successful for processing to proceed further.

Every Validation group takes a set of __Conditions__ all of which should evaluate to __True__ for the Validation group to be considered successful. To configure a complex evaluation method use __MasterCondition__ as below. See [Validations](#Validations)

* You can use any intrinsic function, input stores, macros for passing values to conditions.
* You cannot access secret parameter values using __PARAM__ or __OLDPARAM__ macros

```YAML
CFNXConfiguration:
  PreValidations:
    ValidateStorageConfiguration:
      Conditions:
        SizeReduced: [$[PARAM::StorageSize], 'lt', $[OLDPARAM::StorageSize]]
        TypeChanged: [$[PARAM::StorageType], 'ne', $[OLDPARAM::StorateType]]
      MasterCondition: NOT SizeReduced AND NOT TypeChanged
```

Reference to further details on configuring complex [Conditions](#Conditions)

<a name="TemplatePostValidations"></a>
## Post Validations

**PreRequisites: [Quick reference to Conditions usage in CFNX](#Conditions)**

Post validations are similar to Pre Validations, just that they are executed after the template processing has completed. Post validations are very useful in protecting the templates from catastrophic changes. Below config ensures that the production database deletion policy is always set appropriately.

```YAML
CFNXConfiguration:
  PostValidations:
    ValidateStorageConfiguration:
      Conditions:
        EnsureCorrectAZ:
          - FnX::Template: .Resources.ProductionDatabase.DeletionPolicy
          - 'eq'
          - 'Retain'
```
[docs/samples/transform/cfnx_configuration/post-template-validations.yaml](/docs/samples/transform/cfnx_configuration/post-template-validations.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/cfnx_configuration/post-template-validations.yaml
```

For more details on configuring Validations and Conditions see [TemplatePreValidations](#TemplatePreValidations)

<a name="ResourceLevelProperties"></a>
# Resource Runtime Behavior Configuration

Resource level properties add custom resources to the template which gets executed at runtime. These are more flexible than Stack level properties as you can use __Ref__ and __GetAtt__ to reference other resources in the template while performing actions.

<a name="CFNXTimeout"></a>
## Resource Timeouts

You can configure create/update timeouts on a per resource basis using __CFNXTimeout__. Maximum allowed timeout is 3500 seconds.

```YAML
Transform: [CFNX]
Resources:
  MyS3Bucket:
    CFNXTimeout: 10
    Type: AWS::S3::Bucket
    ...
```

[docs/samples/transform/resource_properties/set-timeout.yaml](/docs/samples/transform/resource_properties/set-timeout.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/resource_properties/set-timeout.yaml

aws cloudformation deploy --template-file docs/samples/transform/resource_properties/set-timeout.yaml --stack-name set-timeout-yaml-1
```

<a name="CFNXPostExecuteDelay"></a>
## Add Delays

You can configure sleep delays post resource create/update using __CFNXPostExecuteDelay__. If there are any other dependent resources in the template, they are made to wait for the Delay as well.

```YAML
Transform: [CFNX]
Resources:
  MyS3Bucket:
    CFNXPostExecuteDelay: 10
    Type: AWS::S3::Bucket
  MyS3Bucket2:
    DependsOn: MyS3Bucket
    Type: AWS::S3::Bucket
```

In the above sample __MyS3Bucket2__ waits for 10 seconds after __MyS3Bucket__ is created or updated. If __MyS3Bucket__ is not updated, the delay is not introduced.

[docs/samples/transform/resource_properties/post-execute-delay.yaml](/docs/samples/transform/resource_properties/post-execute-delay.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/resource_properties/post-execute-delay.yaml

aws cloudformation deploy --template-file docs/samples/transform/resource_properties/post-execute-delay.yaml --stack-name post-execute-delay-yaml-1
```

<a name="CFNXDependsOn"></a>
## Advanced DependsOn

1) When working with transforms like SAM templates, Including Jinja Tempaltes, etc the resource names are not pre determined and cannot be used to set __DependsOn__ property of cloudformation resources. __CFNXDependsOn__ provides a way to provide DependsOn using regex which helps us add Dependencies just by knowing the resource prefixes.
2) Depending on Cross Stack resources. If you want to wait until another dependent resource in a different stack you can add the same using __CFNXDependsOn__

```YAML
ProductionS3Bucket:
  CFNXDependsOn: [MyS3Bucket.+, cross-stack-name/MyS3Bucket]
  Type: AWS::S3::Bucket
```
Above configuration makes __ProductionS3Bucket__ depend on all buckets matching __MyS3BUcket.+__ in the same stack and also on __MyS3Bucket__ in stack __cross-stack-name__.

Cross stack dependencies wait for a maximum of __1800__ seconds before timing out.

[docs/samples/transform/resource_properties/depends-on.yaml](/docs/samples/transform/resource_properties/depends-on.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/resource_properties/depends-on.yaml
```
<a name="CFNXProtectPropertiesFromUpdate"></a>
## Protect Properties From Update

Helps in protecting resource properties from accidental updates. Also, serves as a good way to reference protected attributes in the template itself.

```YAML
MyEc2Volume:
  CFNXProtectPropertiesFromUpdate: ["Size", "VolumeType"]
  Type: AWS::EC2::Volume
  Properties:
    Size: 10
    VolumeType: gp2
    AvailabilityZone: ap-southeast-2a
```

Above configuration ensures that _Size_ and _VolumeType_ are not changed for the resource. If you have to change those, remove those properties from __CFNXProtectPropertiesFromUpdate__ and then update the stack.

You can also specify properties in deeper levels using '/' separated property paths:

```YAML
MyS3Bucket:
  CFNXProtectPropertiesFromUpdate: ["BucketEncryption/ServerSideEncryptionConfiguration"]
  Type: AWS::S3::Bucket
  Properties:
    BucketEncryption:
      ServerSideEncryptionConfiguration:
        - ServerSideEncryptionByDefault:
            SSEAlgorithm: aws:kms
```

[docs/samples/transform/resource_properties/protect-properties.yaml](/docs/samples/transform/resource_properties/protect-properties.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/resource_properties/protect-properties.yaml

aws cloudformation deploy --template-file docs/samples/transform/resource_properties/protect-properties.yaml --stack-name protect-properties-yaml-1
```

<a name="CFNXOutputAttributes"></a>
## Auto add attributes to Outputs
```YAML
MyS3Bucket:
  CFNXOutputAttributes: true # all attributes and Refs
  Type: AWS::S3::Bucket
```

Above configuration will add all attributes and refs that are allowed on the resource to the outputs section of the template. If you choose to add only specific attributes or Ref use below config:

```YAML
MyS3Bucket2:
  CFNXOutputAttributes: ['Arn', 'Ref'] # only few attributes
  Type: AWS::S3::Bucket
```

<a name="CFNXEnsureLatency"></a>
## Ensure HTTP Latency

Though you are not restricted to use this on any logical resource, it makes sense to set this on properties which perform changes that might effect latency of your applications, like Load balancers, Beanstalk environments, ECS services, Code Deploy projects, Auto scaling groups etc.

```YAML
MyECSService:
  CFNXEnsureLatency:
    Url: http://production.endpoint.net/status
    MaxLatency: 0.1 # seconds
  Type: AWS::ECS::Service
```

Every time the ECS service is updated, the production endpoint latency is ensured to be < 0.1 seconds. You can configure the behavior using below properties.

Other Configurable options are:

__NumOfConsecutiveResults__: (Default: 30) Number of consecutive attempts that should result in desired latency
__NumOfSuccessConsecutiveResults__: (Default: 30) number of success consecutive results
__NumOfFailureConsecutiveResults__: (Default: 30) number of failure consecutive results
__TotalTimeout__: (Default: 900) Max configurable timeout is 3500 seconds

[docs/samples/transform/resource_properties/ensure-latency.yaml](/docs/samples/transform/resource_properties/ensure-latency.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/resource_properties/ensure-latency.yaml

aws cloudformation deploy --template-file docs/samples/transform/resource_properties/ensure-latency.yaml --stack-name ensure-latency-yaml-1
```

<a name="CFNXSuccessActions"></a>
## Success Actions

**PreRequisites: [Quick reference to Boto3, HTTP, Python Script Apis usage in CFNX](#WorkingWithApis)**

Whenever a resource is created or updated successfully, you can execute Boto3 APIs, HTTP apis, or run custom scripts using __CFNXSuccessActions__.

For example if you want to send an SMS after every successful update to S3 Bucket.

```YAML
MyS3Bucket:
  CFNXSuccessActions:
    - Api: sns/publish
      Arguments:
        PhoneNumber: '+61435014009'
        Message: !Sub Production S3 Bucket '${MyS3Bucket}' Update Succeeded at - $[MACRO::DateTime]
  Type: AWS::S3::Bucket
```

As you can see we can refer to the Bucket logical ID in the success actions, as Success actions are executed post resource create/update. It is invalid to do the same for [Failure Actions](#CFNXFailureActions).

Making HTTP Api calls on success

```YAML
MyS3Bucket:
  CFNXSuccessActions:
    - ServiceType: HTTP
      Api: http://internal.updatetracker.net/status/update
      Method: POST
      Arguments: # Passed in data
        Message: Production S3 Bucket Update Succeeded at - $[MACRO::DateTime]
  Type: AWS::S3::Bucket
```

_NOTE: IT IS NOT VALID TO REFERENCE THE SAME RESOURCE IN FAILURE ACTIONS (unlike CFNXSuccessActions)._. However, you can reference other resources or parameters.

[docs/samples/transform/resource_properties/success-failure-actions-http.yaml](/docs/samples/transform/resource_properties/success-failure-actions-http.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/resource_properties/success-failure-actions-http.yaml

aws cloudformation deploy --template-file docs/samples/transform/resource_properties/success-failure-actions-http.yaml --stack-name success-failure-actions-http-yaml-1
```

Running Python scripts on success

```YAML
MyS3Bucket:
  CFNXSuccessActions:
    - ServiceType: ScriptExecutor
      Api: |
        print "Success"
  Type: AWS::S3::Bucket
```

[docs/samples/transform/resource_properties/success-failure-actions-script.yaml](/docs/samples/transform/resource_properties/success-failure-actions-script.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/resource_properties/success-failure-actions-script.yaml

aws cloudformation deploy --template-file docs/samples/transform/resource_properties/success-failure-actions-script.yaml --stack-name success-failure-actions-script-yaml-1
```

<a name="CFNXFailureActions"></a>
## Failure Actions:

**PreRequisites: [Quick reference to Boto3, HTTP, Python Script Apis usage in CFNX](#WorkingWithApis)**

Similar to [Success Actions](#CFNXSuccessActions) you can run failure actions using __CFNXFailureActions__.

```YAML
CFNXFailureActions:
  - Service: sns
    Api: publish
    Arguments:
      PhoneNumber: '+61438499846'
      Message: Production S3 Bucket Update Failed at - $[MACRO::DateTime]
```

You can similarly run HTTP and Scripts as well. See [Success Actions](#CFNXSuccessActions) for usage and samples.

[docs/samples/transform/resource_properties/success-failure-actions-script.yaml](/docs/samples/transform/resource_properties/success-failure-actions-script.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/resource_properties/success-failure-actions-script.yaml

aws cloudformation deploy --template-file docs/samples/transform/resource_properties/success-failure-actions-script.yaml --stack-name success-failure-actions-script-yaml-1
```

<a name="CFNXPreValidations"></a>
## Runtime Pre Validations

**PreRequisites: [Quick reference to Boto3, HTTP, Python Script Apis usage in CFNX](#WorkingWithApis)**
**PreRequisites: [Quick reference to Conditions usage in CFNX](#Conditions)**

Pre validations can be useful for:

1) Ensure parameters are valid
2) Ensure dependent service is in desired state
3) Ensure an external service (HTTP) is in desired state
4) Run some custom validations using python scripts

Pre validations accept [Validations](#Validations) object which allows you to define Validation names and their conditions to consider the state as successful.

Pre Validations are run only if the Resource it references (__MyS3Resource__) to has changed its properties.

```YAML
CFNXPreValidations:
  Validation1:
    Conditions:
      Condition1:
      Condition2:
      ...
    MasterCondition: (Optional)
  Validation2: ...
MyS3Resource:
  Type: AWS::S3::Bucket
```

  * You can provide data to validation conditions using any intrinsic / macro functions.

```YAML
MyCFNResource:
  Type: AWS::S3::Bucket
  CFNXPreValidations:
    Validations:
      EnsureNoUpdate:
        Conditions:
          Condition1: [ '$[MACRO::StackStatus]', 'memberof', ['CREATE_IN_PROGRESS', 'DELETE_IN_PROGRESS'] ]
```
[docs/samples/transform/resource_properties/pre-validations-simple.yaml](/docs/samples/transform/resource_properties/pre-validations-simple.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/resource_properties/pre-validations-simple.yaml

aws cloudformation deploy --template-file docs/samples/transform/resource_properties/pre-validations-simple.yaml --stack-name pre-validations-simple-yaml-1
```

The above validation ensures that you can create the Resource and Delete the stack, but not Update or Delete the resource itself from the stack.

  * You can download data from [runtime input stores](#RuntimeInputStores) and pass it to Validations. Runtime Input Stores are different from Global Input Stores, as they are downloaded during the stack execution.

For example:

1) Ensure that the specified cloudwatch alarm is not ALARM before proceeding to updating the resource
2) Ensure there are no route 53 health check failures
3) Ensure an other dependent application based on HTTP is healthy before updating the resource
4) Ensure there are no active outages
5) Ensure it is a weekend or off business hours before updating critical resources etc.

You can reference other resources in the template using __Ref__ and __GetAtt__. _Referencing the same resource will cause circular dependency for this property_.

  * **Scenario1:** Create a Cloudwatch Alarm monitoring database connections and Auto scaling group in the same template. Every time the ASG properties are changed, run pre validation to ensure the cloudwatch alarm is not in active state. Update to ASG is not even triggered if the condition is not met.

```YAML
Transform:
  - Name: CFNX
    Parameters:
      StackName: !Sub ${AWS::StackName}

Parameters:
  MyImageId:
    Type: AWS::SSM::Parameter::Value<String>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2

Resources:
  CloudwatchAlarm:
    Type: "AWS::CloudWatch::Alarm"
    Properties:
      ComparisonOperator: GreaterThanOrEqualToThreshold
      EvaluationPeriods: 3
      MetricName: DatabaseConnections
      Namespace: AWS/RDS
      Period: 60
      Statistic: Average
      Threshold: 50

  ASLC:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      ImageId: !Ref MyImageId
      SecurityGroups: ['default']
      InstanceType: m1.small

  ASG1:
    CFNXPreValidations:
      Boto3InputStores:
        Stores:
          AlarmStore:
            - cloudwatch/describe_alarms
            - AlarmNames:
              - !Ref CloudwatchAlarm
      Validations:
        EnsureAlarmHealthy:
          Conditions:
            StateOK:
              - FnX::Boto3Store: .AlarmStore.MetricAlarms[0].StateValue
              - eq
              - OK
    Type: "AWS::AutoScaling::AutoScalingGroup"
    Properties:
      AvailabilityZones:
        Fn::GetAZs:
          Ref: "AWS::Region"
      LaunchConfigurationName:
        Ref: "ASLC"
      MaxSize: "3"
      MinSize: "2"
```

[docs/samples/transform/resource_properties/pre-validations-refs.yaml](/docs/samples/transform/resource_properties/pre-validations-refs.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/resource_properties/pre-validations-refs.yaml

aws cloudformation deploy --template-file docs/samples/transform/resource_properties/pre-validations-refs.yaml --stack-name pre-validations-refs-yaml-1
```

  * **Scenario2:** Ensure that there are no active outages by monitoring an internal service endpoint, and also make sure that it is not a Weekday or active Business hours before updating an S3 bucket. See [Configuring VPC Access](#ConfiguringVPCAccess) if you need to work with internal services.

```YAML
Transform: [CFNX]

Resources:
  MyS3Bucket:
    CFNXPreValidations:
      HTTPInputStores:
        Stores:
          ProductionStore: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/production-health.json
      Validations:
        CheckOutages:
          Conditions:
            EnsureNoOutages:
              - FnX::HTTPStore: .ProductionStore.RESPONSE.active_outages_num
              - eq
              - 0
        CheckBusinessHours:
          Conditions:
            IsNonBusinessHours:
              -
                FnX::Python: |
                  from datetime import datetime
                  return datetime.now().hour not in range(9, 17)
              - 'eq'
              - True
            IsWeekend:
              -
                FnX::Python: |
                  from datetime import datetime
                  return datetime.today().weekday()
              - 'memberof'
              - [5, 6]
          MasterCondition: IsNonBusinessHours AND IsWeekend
          # Master condition is Optional and by default is AND or all conditions
    Type: AWS::S3::Bucket
```

  __CFNXPreValidations__ allows you to also specify an addition __Api__ to work with in the same Configuration. This allows for passing data from InputStores to Api to make further API calls before final Validations.

  * **Scenario3**: Read the DynamoDB table name from a HTTP config service, pass it to the Api to get the status of the DynamoDB table, and ensure DynamoDB table is in valid state before creating/updating an S3 Bucket.

```YAML
MyS3Bucket:
  Type: AWS::S3::Bucket
  CFNXPreValidations:
    # Get production configuration object
    HTTPInputStores:
      Stores:
        ProductionConfig: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/config-service.json
    # Get table status
    Api: dynamodb/describe_table
    Arguments:
      TableName:
        FnX::HTTPStore: .ProductionConfig.RESPONSE.ProductionTable
    # Validate subscription is confirmed
    Validations:
      SubStatus:
        Conditions:
          01-Status200: ['Resp::.ResponseMetadata.HTTPStatusCode', 'eq', 200]
          02-SubConfirmed: ['Resp::.Table.TableStatus', 'eq', 'ACTIVE']
```

<a name="CFNXPostValidations"></a>
## Runtime Post Validations

**PreRequisites: [Quick reference to Boto3, HTTP, Python Script Apis usage in CFNX](#WorkingWithApis)**
**PreRequisites: [Quick reference to Conditions usage in CFNX](#Conditions)**

Post validations can be useful for validating the state of the resource it is defined on. It works exactly the same way as [Pre Validations](#PreValidations) however since it is run after resource creation/updation, you can __Ref__ or __GetAtt__ the same resource to provide arguments to the Input Stores or Apis being used.

Post validations accept [Validations](#Validations) object which allows you to define Validation names and their conditions to consider the state as successful.

Post Validations are run only if the Resource it references (__MyS3Resource__) to has changed its properties.

```YAML
CFNXPostValidations:
  Validation1:
    Conditions:
      Condition1:
        - !Ref MyS3Resource
        ...
      Condition2:
      ...
    MasterCondition: (Optional)
  Validation2: ...
MyS3Resource:
  Type: AWS::S3::Bucket
```

  * **Scenario** Create an S3 bucket to host production application APK. Every time the bucket is updated ensure the APK is accessible, has the right content length and content type, and ensure the bucket has encryption enabled.

To work with this scenario we introduce a new property called __SkipPhase2ForEvents__ which disables Validations for Create and Delete phase. This is because when bucket is created we do not have the production APK available yet to verify its content. (The demo template however introduces a CFNX resource to upload the s3 production apk with wrong content type for demo purposes)

```YAML
MyS3Bucket:
  ### Validations
  CFNXPostValidations:
    SkipPhase2ForEvents: ['Create', 'Delete']
    HTTPInputStores:
      Stores:
        ProductionConfig: https://raw.githubusercontent.com/jarvisdreams9/cfn-samples/master/config-service.json
    Api: s3/get_object
    Arguments:
      Bucket:
        !Ref MyS3Bucket
      Key:
        FnX::HTTPStore: .ProductionConfig.RESPONSE.ProductionAPK
    Validations:
      EnsureProductionSanity:
        Conditions:
          EnsureAPKContentLength: ['Resp::.ContentLength', 'gt', 1]
          IsAndroidType: ['Resp::.ContentType', 'eq', 'application/vnd.android.package-archive']
          IsJarType: ['Resp::.ContentType', 'eq', 'application/java-archive']
          EnsureServerSideEncryption: ['Resp::.ServerSideEncryption', 'eq', 'AES256']
        MasterCondition: ( IsAndroidType OR IsJarType ) AND EnsureAPKContentLength AND EnsureServerSideEncryption
        # Master condition is Optional and by default is AND or all conditions

  ## Bucket Properties ##
  Type: AWS::S3::Bucket
  Properties:
    BucketEncryption:
      ServerSideEncryptionConfiguration:
        - ServerSideEncryptionByDefault:
            SSEAlgorithm: AES256
```

  * Similarly you can work with [Python Scripts Apis](#ScriptExecutorApiUsage) as well

[docs/samples/transform/resource_properties/post-validations0.yaml](/docs/samples/transform/resource_properties/post-validations0.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/resource_properties/post-validations0.yaml

aws cloudformation deploy --template-file docs/samples/transform/resource_properties/post-validations0.yaml --stack-name post-validations0-yaml-1
```

<a name="CFNXOutputAttributesToS3"></a>
## Output Attributes of a resource to s3

Similar to Exporting all the attributes/Ref to Outputs section, we can output the attributes to a defined S3 location. This is useful in cross-account/cross-region data sharing between cloudformation stacks. __CFNXOutputAttributesToS3__ takes two properties:

  * __Location__: S3 bucket/key location where the attributes should be stored
  * __JsonPath__: Inside the key, the outputs are structured in this path

CFNX provides a special macro __$[MACRO::S3LocalPath]__ which defaults to __aws/cfnx/ACCOUNTID/REGIONNAME/STACKNAME__ which can be used to store this outputs. This is required if you are using the same output location in multiple stacks so that they do not overwrite each other.

Once the outputs are exported to S3, you can use [CFNXGlobalImportExporter](#CFNXGlobalImportExporter) to read the data from S3 in other stacks in with in and across regions or accounts.

```YAML
MyS3Bucket:
  CFNXOutputAttributesToS3:
    Location: s3://mys3bucket/cfnx-all-attributes.json
    JsonPath: $[MACRO::S3LocalPath]/MyS3Bucket
  Type: AWS::S3::Bucket
```

With above configuration the outputs are saved in __s3://mys3bucket/cfnx-all-attributes.json__ as follows:

```JSON
{
    "aws": {
        "cfnx": {
            "ACCOUNTID": {
                "REGIONNAME": {
                    "STACKNAME": {
                        "MyS3Bucket": {
                            "Outputs": {},
                            "S3Outputs": {
                                "MyS3BucketWebsiteURL": "value",
                                "MyS3BucketDualStackDomainName": "value",
                                "MyS3BucketArn": "value",
                                "MyS3BucketDomainName": "value"
                            }
                        }
                    }
                }
            }
        }
    }
}
```

  * You can choose to export only selected attributes or 'Ref'

```YAML
MyS3Bucket2:
  CFNXOutputAttributesToS3:
    Location: s3://bhagangu-crh-repos/cfnx-selected-attributes.json
    Attributes: ['Arn', 'Ref']
    JsonPath: $[MACRO::S3LocalPath]/MyS3Bucket2
```

Output Stored:

```
{
    "aws": {
        "cfnx": {
            "ACCOUNTID": {
                "REGIONNAME": {
                    "STACKNAME": {
                        "MyS3Bucket2": {
                            "Outputs": {},
                            "S3Outputs": {
                                "MyS3Bucket2Ref": "value",
                                "MyS3Bucket2Arn": "value"
                            }
                        }
                    }
                }
            }
        }
    }
}
```

[docs/samples/transform/resource_properties/output-attributes-to-s3.yaml](/docs/samples/transform/resource_properties/output-attributes-to-s3.yaml)
```
bash cfnxcmd transform-local -t docs/samples/transform/resource_properties/output-attributes-to-s3.yaml

aws cloudformation deploy --template-file docs/samples/transform/resource_properties/output-attributes-to-s3.yaml --stack-name output-attributes-to-s3-yaml-1
```

<a name="CFNXGlobalImportExporter"></a>
## Output Attributes of a resource to s3

**PreRequisites: [Working with I/O Stores](#WorkingWithIOStores) [Working with Apis](#WorkingWithApis)**

Though __CFNXGlobalImportExporter__ is defined at resource level, it is called __Global__ since it can be referenced by other resources in the stack. This property accepts a __ProviderName__ which can be used by any resource including the resource declaring the property.

We import the data into the provider using:

  * __S3InputStores__: These are similar to [GlobalS3InputStores](#GlobalS3InputStores), but can reference logical ids of other resources. Values can be accessed using __FnX::S3Store__ by passing a JQ query.
  * __Boto3InputStores__: These are similar to [GlobalBoto3InputStores](#GlobalBoto3InputStores), but can reference logical ids of other resources. Values can be accessed using __FnX::Boto3Store__ by passing a JQ query.
  * __HTTPInputStores__: These are similar to [GlobalHTTPInputStores](#GlobalHTTPInputStores), but can reference logical ids of other resources. Values can be accessed using __FnX::HTTPStore__ by passing a JQ query.
  * __ScriptInputStores__: These are similar to [GlobalScriptInputStores](#GlobalScriptInputStores), but can reference logical ids of other resources. Values can be accessed using __FnX::ScriptStore__ by passing a JQ query.
  * __Api__: Make an Api call based on data from input stores and query the data using __FnX::Resp__ or __Resp::__ string prefix. Checkout [Working with Apis](#WorkingWithApis)
  * __Arguments__: when Api is used, you can pass arguments to the same

We export the data using below properties:

  * __Outputs__: Accepts a dictionary object and all key values in this section are available for other resources to reference using __GetAtt__.
  * __S3Outputs__: Accepts a dictionary object and all key values similar to Outputs, but stored in __OutputStore__ instead. These are not accessible with __GetAtt__.
  * __OutputStore__: Accepts __Location__ (s3://mybucket/outputs.json) and __JsonPath__ (outputs/from/stack) where the outputs declared in __S3Outputs__ will be stored
  * __ExportAllOutputsToS3__: If set to __true__ all __Outputs__ and __S3Outputs__ are uploaded to __OutputStore__. If there is a conflict in output name, __S3Outputs__ take precedence.

**Outputs here can be of any valid JSON data type including lists, dicts.**

  * __S3InputStores__ and __Boto3InputStores__ accept optional __AssumeRoleConfig__ to pass priviliged or cross account role configurations for downloading data.

**Scenario**:
Complete Configuration Reference:
```YAML
Resources:
  MyCFNResource:
    CFNXGlobalImportExporter:
      ProviderName: MyDataProvider

      ### IMPORTER CONFIGURATION ###
      Boto3InputStores:
        Stores:
          Store1: ...
          Store1: ...
        AssumeRoleConfig: # (Optional)
          RoleArn: <priviliged role>
      S3InputStores:
        Stores:
          MyLocalStore: <self account bucket>
          FromCrossAccount: <cross-account-s3-bucket>
        AssumeRoleConfig: # (Optional)
          RoleArn: <priviliged role>
      HTTPInputStores:
        Stores:
          Store1: http://publicendpoint.com/sub/url
          Store2: http://internal.net/sub/url
      ScriptInputStores:
        Stores:
          Store1: |
            return {"Key": "Value"}
          Store2: |
            return {"AnotherKey": "AnotherValue"}

      ### OPTIONAL API CONFIGURATION ###
      Api:
      ServiceType: boto3 (default) | HTTP | ScriptExecutor
      Arguments:

      ### INTERNAL EXPORTER CONFIGURATION accessible via GetAtt ###
      Outputs:
        DatabasePassword:
          FnX::Boto3Store: .Store1.<JQ>
        OtherOutput:
          FnX::ScriptStore: .Store1.Key
        ApiResponse:
          FnX::Resp: <JQ>

      ### EXTERNAL S3 EXPORTER CONFIGURATION accessible via another CFNXGlobalImportExporter ###
      OutputStore:
        Location: s3://mybucket/key.json
        JsonPath: inside/json/path
        AssumeRoleConfig: # Optional
          RoleArn:
      S3Outputs:
        OutputToS3: MyValue
      ExportAllOutputsToS3: true    # --> Export all Outputs section and S3Outputs section to S3
```
