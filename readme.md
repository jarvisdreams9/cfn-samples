
<a name="Setup"></a>
# Setup

## Prerequisites

[aws-cli](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)
```
sudo pip install awscli
```
[Create an S3 bucket to upload lambda source code](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/create-bucket.html)
```
aws s3 mb s3://cfnx-repository-<account-id>
```

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

<a name="BasicTransformUsage"></a>

## Basic Transform Usage
__CFNX__ is declared at top level in the template. Similar to [AWS::Serverless-2016-10-31](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/transform-aws-serverless.html). It is not required or appropriate to declare the transform at any other level.

```YAML
Transform: CFNX
```

[docs/samples/macros/setup/basic.yaml](/docs/samples/macros/setup/basic.yaml)

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

[docs/samples/macros/setup/basic_transform_with_parameters.yaml](/docs/samples/macros/setup/basic_transform_with_parameters.yaml)

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

[docs/samples/macros/setup/basic_transform_cfnxconfiguration.yaml](/docs/samples/macros/setup/basic_transform_cfnxconfiguration.yaml)

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

[docs/samples/macros/setup/multiple_transforms.yaml](/docs/samples/macros/setup/multiple_transforms.yaml)

<a name="CFNX"></a>
# CFNX Transform

<a name="CLISetupUsage"></a>
## CLI Setup and Usage

__CFNX__ provides a CLI which makes it easy to test transform templates locally without having to deploy them to cloudformation or create change set every time.

It is recommended to use virtualenv to avoid version compatibility issues

```
$ sudo pip install virtualenvwrapper
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

```
bash cfnxcmd transform-local -t docs/samples/macro/cli-usage/basic.yaml
```

## transform-local-sam

Prerequisite: [Docker](https://docs.docker.com/) needs to be running locally

```
bash cfnxcmd transform-local-sam -t docs/samples/macro/cli-usage/basic.yaml
```

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

##### Special Macros

- $[MACRO::SHASUM] - Provides first 10 characters of SHA256 of the string prefix

```
string-to-create-shasum-from-$[MACRO::SHASUM]
---
string-to-create-macro-from-7e398284bf
```
- $[MACRO::StackSHASUM] - Provides first 10 characters of SHA256 of StackId. Use this to reuse the same template to create named resources, as this value will be highly unique and immutable per stack.

```
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

```
ValueAsString: 'Current Epoch Seconds: $[MACRO::EpochSeconds]'
---
ValueAsString: 'Current Epoch Seconds: 1531068943'
```
```
ValueAsNumber: $[MACRO::EpochSeconds]
---
ValueAsNumber: 1531068943
```

- $[MACRO::EpochSeconds]       - Returns current epoch seconds
- $[MACRO::RandomInteger]      - Returns a random integer between 1 - 32000
- $[MACRO::RandomBigInteger]   - Returns a random integer greater than 10^9
- $[MACRO::RandomShortInteger] - Returns a random integer between 1 - 127

<a name="IntrinsicFunctions"></a>
## Generic Intrinsic Functions

_Note: Macros, Intrinsic Functions, Includes can be used at any level in the template_

- __FnX::Type__: Convert values from one data type to another. Eg: convert JSON string to JSON Object, or convert Number values to strings vice versa or even create 'None' values etc. Supported types are: __int, float, bool, json, jsonobj, yamlobj, str, none__

```
FnX::Type:
  Value: 100
  Type: str
```

- __FnX::Type__: Convert values from one data type to another. Eg: convert JSON string to JSON Object, or convert Number values to strings vice versa or even create 'None' values etc. Supported types are:

* Integer (int)
```
FnX::Type:
  Value: '100'
  Type: int
---
100
```
* String (str)
```
FnX::Type:
  Value: 100
  Type: str
---
'100'
```
* Float (float)
```
FnX::Type:
  Type: float
  Value: '100.1'
---
100.1
```

* None (none) - Cloudformation does not allow you to pass 'null' values as part of template so you can use the conversion within the template for validations etc.

```
FnX::Type:
  Type: none
  Value: 'none'
---
None
```
* Boolean (bool)
```
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

```
FnX::Type:
  Type: json
  Value:
    Key: Value
---
'{"Key": "Value"}'
```

* JSON Object from string (jsonobj)

```
FnX::Type:
  Type: json
  Value: '{"Key": "Value"}'
---
{
    "Key": "Value"
}
```

* YAML String from object (yaml)

```
FnX::Type:
  Type: yaml
  Value:
    Key: Value
---
"{Key: Value}\n"
```

* YAML Object from string
```
FnX::Type:
  Type: yaml
  Value: "{Key: Value}\n"
---
Key: Value
```

* Using CFN Flip to JSON (cfnflipjson)
```
FnX::Type:
  Type: cfnflipjson
  Value: "!Ref MyResource"
---
"{\n    \"Ref\": \"MyResource\"\n}"
```

* Using CFN Flip to YAML (cfnflipyaml)
```
FnX::Type:
  Type: cfnflipyaml
  Value: '{"Fn::GetAtt": ["MyResource", "Arn"]}'
---
"!GetAtt 'MyResource.Arn'\n"
```

* Nested Types (nesting is allowed with any intrinsic function in CFNX)

```
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
