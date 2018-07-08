
<a name="Setup">
### Setup

##### Prerequisites

[aws-cli](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)

[Create an S3 bucket to upload lambda source code](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/create-bucket.html)

##### Deployment
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

<a name="BasicTransformUsage">
__CFNX__ is declared at top level in the template. Similar to [AWS::Serverless-2016-10-31](#https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/transform-aws-serverless.html). It is not required or appropriate to declare the transform at any other level.
### Basic Transform Usage
```YAML
Transform: CFNX
```

[docs/samples/macros/setup/basic.yaml](#/docs/samples/macros/setup/basic.yaml)

```YAML
Transform: [CFNX]
```

* For certain features we require stack name to be passed as follows
```YAML
Transform:
  - Name: CFNX
    Parameters:
      StackName: !Sub ${AWS::StackName}
```

* Passing configuration to CFNX

```YAML
Transform:
  # Call CFNX initially
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
