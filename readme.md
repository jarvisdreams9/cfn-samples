<a name="References"></a>
# References

<a name="WorkingWithApis"></a>
## How to use APIs in CFNX

__CFNX__ provides support for 3 types of apis and provides the output in a generic way to make it feel seamless to work with different integration types.

1) [boto3](#Boto3ApiUsage) (Default)
2) [HTTP](#HTTPApiUsage)
3) [ScriptExecutor](#ScriptExecutorApiUsage) (Python Scripts)

Please read through the specific api type you like to use with above links.

Apis are use in CFNX in two different contexts:

1) Making Simple Api calls in __CFNXPreValidations__, __CFNXPostValidations__, __CFNXDataProvider__, __CFNXDataExporter__. In this context you can query the output of the api using:

- __FnX::Resp__ intrinsic function
```YAML
Value:
  FnX::Resp: ".ResponseMetadata.HTTPStatusCode" # For boto3
```
- __Resp::__ string prefix
```YAML
Value: 'Resp::.ResponseMetadata.HTTPStatusCode'
```

2) Polling context where apis are made every PollDelay seconds in __CFNXWaitOn__, __CFNXStabilizeOn__.
In this context we are waiting for a particular state to be achieved. You can query the final output of the api using:

- __FnX::WaitResp__ intrinsic function
```YAML
Value:
  FnX::WaitResp: ".ResponseMetadata.HTTPStatusCode" # For boto3
```
- __WaitResp::__ string prefix
```YAML
Value: 'WaitResp::.ResponseMetadata.HTTPStatusCode'
```

You can use the above methods to query values for properties inside config objects like __SuccessOn__, __FailOn__, __Outputs__, __S3Outputs__.

<a name="Boto3ApiUsage"></a>
### Boto3
___

[Boto3](http://boto3.readthedocs.io/en/latest/reference/services/) is a python library which provides apis to work with different api services like create s3 buckets, create ec2 instances, read data from s3, send SMS / SNS etc. Every api has a __ServiceName__ and __ApiName__. For example to create an s3 bucket use __ServiceName: s3__ and __ApiName: create_bucket__.

To work with boto3 apis in CFNX, we provide one argument named __Api__ which is of format __ServiceName/ApiName__. So the create bucket operation is specified as __Api: s3/create_bucket__.

boto3 apis also support passing __Arguments__ to the apis. With CFNX we pass those using __Arguments__ property. Eg:

```YAML
Api: s3/create_bucket
Arguments:
  Bucket: MyS3Bucket
```

**Reading response from Boto3 apis**
__CFNX__ supports using [JQ](#UsingJQ) queries on the response of boto3. The output from the API is provided as it is for queries to work with. Eg: create_bucket will provide a response with not just the location of the new bucket but also some response metadata:

```JSON
{
    "ResponseMetadata": {
        "HTTPHeaders": {
            "content-length": "123",
            "date": "Sun, 08 Jul 2018 17:54:27 GMT",
            "x-amzn-requestid": "f4f8b71d-82d7-11e8-8ff6-93d137616026"
        },
        "HTTPStatusCode": 200,
        "RequestId": "f4f8b71d-82d7-11e8-8ff6-93d137616026",
        "RetryAttempts": 0
    },
    "Location": "MyS3Bucket"
}
```

We can now make some queries on this output as follows:
```
.ResponseMetadata.HTTPStatusCode
---
200
```
```
.Location
---
MyS3Bucket
```

However, when you reference an invalid property you get a __None__ response which will throw an exception. See [Working with None in JQ](#JQUnsafe)

```
.InvalidProperty
---
None
```

**Advanced Boto3 Config**
You can also configure boto3 to use different connection settings like __region__, __timeout__, __retries__ etc. For full list see [Boto3 Config Reference](https://botocore.readthedocs.io/en/latest/reference/config.html)

```YAML
Boto3ConnectionConfig:
  region_name: us-east-1
  connect_timeout: 30      # Default: 30
  read_timeout: 60         # Default: 60
  retries:
    max_attempts: 0        # Default: 0
```

You can configure __AssumeRoleConfig__ for privileged access or cross account access using below syntax:

```YAML
AssumeRoleConfig:
  RoleArn: <arn>
```

Complete Config Reference:
```YAML
ServiceType: boto3       # ---> Optional for boto3 as it is the default
Api: s3/create_bucket
Arguments:
  BucketName: MyBucket
Boto3ConnectionConfig:
  region_name: us-east-1
  connect_timeout: 30      # Default: 30
  read_timeout: 60         # Default: 60
  retries:
    max_attempts: 0        # Default: 0
AssumeRoleConfig:
  RoleArn: <arn>
```

<a name="HTTPApiUsage"></a>
### HTTP
___

CFNX makes working with HTTP similar to Boto3, with a simple addition that you need to explicitly provide __ServiceType: HTTP__ in the configuration. Apart from this it accepts:

- __Api__: which should be the HTTP URL to work with.
- __Method__: (Default: GET) HTTP method to work with.
- __Arguments__: If method is _['GET', 'HEAD', 'OPTIONS']_ it is converted to query parameters otherwise (eg: POST) it is converted to JSON and passed as part of __data__

```YAML
ServiceType: HTTP
Api: https://jsonplaceholder.typicode.com/posts
Method: GET
Arguments:
  userId: 1
```

```YAML
ServiceType: POST
Api: https://jsonplaceholder.typicode.com/posts
Method: GET
Arguments:
  title: My New Post
  body: useful blog
```

**Reading response from HTTP apis**
CFNX automatically tries to convert the response into a JSON object if response is a valid JSON string otherwise, the response is provided as a string. Response is available inside __RESPONSE__ and status code of the response is available in __STATUS_CODE__.

For example making a GET request to __https://jsonplaceholder.typicode.com/posts?userId=1__

```JSON
{
    "STATUS_CODE": 200,
    "RESPONSE": [
      {
        "userId": 1,
        "id": 1,
        "title": "My First Post",
        "body": "useful content"
      }
    ]
}
```

Once we have the JSON response, we can make [JQ](#UsingJQ) queries.

```YAML
.STATUS_CODE
...
200
```

```YAML
.RESPONSE[0].title
...
My First Post
```

**Advanced HTTP Config**
Similar to boto3 you can configure connection settings with below object:

```YAML
HTTPConnectionConfig:
  retries: 0             # Default 0
  timeout: 60            # Default 60
```

Complete Config Reference:
```YAML
ServiceType: HTTP
Api: http://myservice.com
Method: POST
Arguments:
  Username: bob
HTTPConnectionConfig:
  retries: 0             # Default 0
  timeout: 60            # Default 60
```

<a name="ScriptExecutorApiUsage"></a>
### ScriptExecutor
___

CFNX makes it easy to work with ScriptExecutor by making it compatible with API services. We use __ServiceType__ as __ScriptExecutor__ for scripts. We provide the code to execute in ScriptExecutor directly into the __Api__ property. Along with this we can also pass any arguments that we would like to access within the script using __Arguments__

```YAML
ServiceType: ScriptExecutor
Api: |
  return args['UserName']
Arguments:
  UserName: bob
```

We can also add __ScriptExecutorConnectionConfig__ to configure retries and timeouts:

```YAML
ScriptExecutorConnectionConfig:
  retries: 0                              # Default 0
  timeout: 60                             # Default 60
```

When working with boto3/http you can also pass __Boto3ConnectionConfig__ and __AssumeRoleConfig__ and __HTTPConnectionConfig__ which are pre-baked into the executor shell for easy access.

Along with this you have access to pre-backed functions and sessions as below:

__args__: access arguments passed to Api using args['ArgName']
__cmd__: Run Shell commands using this function. Output of this function provides 3 values
```
exit_code, output, error = cmd("ls")
```
__event__: access the 'event' object passed to lambda
__request__: access CFNX/GCR request object for advanced context information like start time, etc.
__properties__: for CFNX (transforms) this is set to __CFNXConfiguration__. In GCR this is set to __ResourceProperties__.
__boto3_session__: pre baked boto3 session. If __AssumeRoleConfig__ is passed it initializes the session with role credentials, else uses default lambda execution role.
__boto3_config__: pre baked config object having __Boto3ConnectionConfig__
__http_client__: provides a simple interface to work with http requests.
```
http_client.get_result(method, url, retries=0, timeout=60, data={}, headers={})
```
__http_config__: pre baked config object having __HTTPConnectionConfig__ Eg: {'retries': 0, 'timeout': 60}
__log_client__: helps in logging 'info', 'error', 'debug', 'exception' messages.
```
log_client.info("Starting execution")
log_client.exception("Raise exception and fail the request")
```
__dry_run__: helps to determine if request is being run in DryRun mode. Used in CLI.
__utils__: multiple utility functions including is_string(), is_boolean() etc. For complete list see helpers/utils.py in source code.

Complete Config Reference:
```YAML
ServiceType: ScriptExecutor
Api: |
  client = boto3_session.client('rds')
  return args['UserName']
Arguments:
  UserName: bob
ScriptExecutorConnectionConfig:
  retries: 0          # Default 0
  timeout: 60                             # Default 60

HTTPConnectionConfig:    # --> Passed to http_config inside executor shell
  retries: 0             # Default 0
  timeout: 60            # Default 60

Boto3ConnectionConfig:   # --> Passed to boto3_config inside executor shell
  region_name: us-east-1
  connect_timeout: 30      # Default: 30
  read_timeout: 60         # Default: 60
  retries:
    max_attempts: 0        # Default: 0
AssumeRoleConfig:         # --> Initialized into boto3_session inside executor shell
  RoleArn: <arn>
```

<a name="Conditions"></a>
## Working with Conditions

Conditions are used at various levels in CFNX. Some of the properties where conditions are used are __SuccessOn__, __FailOn__, __PreValidations__, __PostValidations__. Conditions determine the final True/False status based on mentioned rules and an optional __MasterCondition__. Every condition is made of two properties:

__Conditions__: key value object with format ConditionName: <rule>
__MasterCondition__(Optional): By default all Conditions should be True. You can specify a master condition to alter the behavior by using __NOT__, __AND__, __OR__ operators.

- Every rule in __Conditions__ block take 3 elements, __Operand1__, __Operator__, __Operand2__.
- The values for __Operand1__ and __Operand2__ can be provided by intrinsic functions, macros, responses from API requests etc.
- __Operator__ support all operations supported by FnX::Operator function [Operations](#OperatorFunction)

Basic Conditions Block:
```YAML
Conditions:
  MyCondition: [1, 'eq', '1']
```
Though this solves many simple use cases, for slight intermediate levels we will need to write this:
```YAML
Conditions:
  MyCondition1: [1, 'eq', '1']
  ### Default AND ###
  MyCondition2: [2, 'eq', '2']
```
The final state is determined by using __AND__ of all the mentioned conditions. To change this Default behavior use __MasterCondition__.

```YAML
Conditions:
  MyCondition1: [1, 'eq', '1']
  MyCondition2: [2, 'eq', '2']
MasterCondition: 'MyCondition1 OR MyCondition2'
```

The __MasterCondition__ can only have condition names and __AND__, __OR__, __NOT__, __(__ and __)__ literals. **All literals should be separated by at least one space character**

For example the below __MasterCondition__ is possible:

```YAML
Conditions:
  LatencyUnderControl:
    - FnX::GlobalBoto3InputStore: .MyCloudwatchMetric.Latency
    - lt
    - 0.1
  IsWeekday:
    -
      FnX::Python: |
        from datetime import datetime
        return datetime.now().weekday()
    - memberof
    - [0,1,2,3,4]
  NoActiveOutages:
    - FnX::GlobalHTTPInputStore: .LatencyStore.ActiveOutages
    - eq
    - 0
MasterCondition: ( LatencyUnderControl AND NoOutagesActive ) OR ( NOT IsWeekDay )
```

<a name="Validations"></a>
## Validations

Validations are higher level to Conditions. Validations group [Conditions](#Conditions) objects allowing you to define bigger scenarios to test for success or failure evaluation.

Basic Validation:
```YAML
Validations:
  StorageValidation:
    [Conditions](#Conditions):
      EnsureStorageTypeIsGp2: ...
      EnsureStorageSizeIsMinimum: ...
      EnsureStorageIsEncrypted: ...
    MasterCondition: ...
  AutoScalingValidation:
    [Conditions](#Conditions):
      EnsureInstancesAreEvenNumbered: ...
      EnsureUpdatePolicyIsRolling: ...
    MasterCondition: ...
```
