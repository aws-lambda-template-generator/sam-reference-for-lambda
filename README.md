# SAM Reference

## What You Need To Know About SAM Templates

AWS SAM template is a thin abstraction of a CloudFormation template to develop and deploy lambda by a AWS tool, SAM (Serverless Application Model). It has shorter syntax compared to writing infrastructure as code with CloudFormation. We still do a little bit of learning when it comes to writing SAM templates. Using `sam init` will create a template, so it is a good starting point. Then we can build the template according to our needs.

Quick Example

This creates Lambda function, IAM role, API Gateway, DynamoDB table.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: 'ts-simple-api-proxy-with-sam

  Sample SAM Template for ts-simple-api-proxy

'

Resources:
  TsSimpleProxyApiFunction: # (1) Creates Lambda function
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: main.example
      Runtime: nodejs14.x
      Events:
        TsSimpleProxy:
          Type: Api # (2) Creates API Gateway
          Properties:
            Path: /get-users
            Method: get
      Policies: # (3) Creates IAM Role
        # We can choose from a list of policy templates which are specific to SAM
        # See: https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-policy-templates.html
        - DynamoDBReadPolicy:
            TableName: !Ref ProductTable
        - SSMParameterReadPolicy:
            ParameterName: mock-parameter
      ProductTable: # (4) Creates DynamoDb table
        Type: AWS::Serverless::SimpleTable
```

<strong><u>SAM Resources</u></strong>

We can use both CloudFormation Resources and SAM specific resources. At the moment, there are 7 serverless resource types supported by SAM.

1. AWS::Serverless::Function
2. AWS::Serverless::Api
3. AWS::Serverless::HttpApi
4. AWS::Serverless::SimpleTable
5. AWS::Serverless:::LayerVersion
6. AWS::Serverless::Application

<strong>1. AWS::Serverless::Function</strong>

This creates a lambda function.

```yaml
MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: index.handler
      Runtime: nodejs14.x
      Description: Example Lambda Function
      MemorySize: 1024
      Timeout: 15
      Tracing: Active|PassThrough
      Events:
        TsSimpleProxyApi:
          Type: HttpApi
          Properties:
            Path: /
            Method: get
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref ProductTable
      
      Metadata:
        # Leaving blank will use the default build system
        # We can set BuildMethod to makefile to use a MakeFile directed build in this example.
        BuildMethod: nodejs12.x | makefile 
```

At the moment there are 15 event sources.

- S3
- SNS
- Kinesis
- DynamoDB
- SQS
- Api
- HttpApi
- Schedule
- CloudWatchEvent
- CloudWatchLogs
- IoTRule
- AlexaSkill
- Cognito
- EventBridgeRule
- MSK

For the IAM Policies, there are over 70+ pre-configured one. If you cannot find one, we can use CloudFormation syntax to create custom one.

<strong>2. AWS::Serverless::Api</strong>

This is where we can create Api Gateway to build REST API. You can use OpenAPI to define it and refer to the swagger file.

```yaml
TsSimpleProxyApi:
  Type: AWS::Serverless::Api # Short hand Api works, too.
  Properties:
    StageName: prod
    DefinitionUri: ./swagger.yml
```

<strong>3. AWS::Serverless::HttpApi</strong>

This is V2 of REST API (supposed to be faster, lower cost and easy to use compared to Api) for creating REST APIs.

```yaml
TsSimpleProxyApi:
  Type: AWS::Serverless::HttpApi # or just HttpApi
  Properties:
    CorsConfiguration:
      AllowMethods:
        - GET
        - POST
      AllowOrigins:
        - http://localhost:8080
        - http://another.domain.com
```

<strong>4. AWS::Serverless::SimpleTable</strong>

This is to create a DynamoDB table with Primary key id, Dynamic capacity and AWS managed encryption.

```yaml
MyTable:
  Type: AWS::Serverless::SimpleTable
  Properties:
    TableName: my-table
    PrimaryKey:
      Name: id
      Type: String
    ProisionedThroughput:
      ReadCapacityUnits: 5
      WriteCapacityUnits: 5
    Tags:
      Department: Engineering
      AppType: Serverless
```

<strong>5. AWS::Serverless::LayerVersion</strong>

Layer to share code across multiple lambda functions. We can build layers accordingly as an option.

```yaml
MyLayer:
  Type: AWS::Serverless::LayerVersion
  Properties:
    LayerName: static-data
    Description: Static data layer for app
    ContentUri: layer/
    CompatibleRuntimes:
      - nodejs12.x
    LicenseInfo: 'MIT'
    RetentionPolicy: Retain
  Metadata:
    BuildMethod: nodejs12.x | makefile
```

<strong>6. AWS::Serverless::Application</strong>

It allows to use an external application and pull it in.

```yaml
MyApplication:
  Type: AWS::Serverless::Application
  Properties:
    Location:
      ApplicationId: 'arn:aws:serverlessrepo:...'
      SemanticVersion: 1.0.0
    Parameters:
      StringParameter: parameter-value
      IntegerParameter: 2

MyApplication:
  Type: AWS::Serverless::Application
  Properties:
    Location: https://s3.amazonaws.com/bucket/tmpl.yaml
```

<strong>7. AWS::Serverless::StateMachine</strong>

For creating step functions.

```yaml
MySampleStateMachine:
  Type: AWS::Serverless::StateMachine
  Properties:
    DefinitionUri: sfdn/MyStateMachine.asl.json
    Role: arn:aws:iam:::role/service-role/MyRole
    DefinitionSubstitutions:
      MyFunctionArn: !GetAtt MyFuction.Arn
      MyDDBTable: !Ref TransactionTable
```

<strong><u>Globals</u></strong>

We can define Global parameters for multiple lambda functions to share. It makes SAM templates smaller by re-using attributes throughout.

```yaml
Globals:
  Function:
    Runtime: nodejs12.x
    Handler: index.handler
    Timeout: 15
    MemorySize: 1024
    Layers: 
      - arn:aws::2222222:layer:aws-sdk:6
    Environment:
      Variables:
        TABLE_NAME: MyTable
```

<strong><u>Building Reusable Templates</u></strong>

<strong>1. Parameters</strong>

To deploy to different environments, we can use parameters that can be passed through SAM CLI.

Parameter types support the below:

- Sting
- Number
- List<Number>
- CommaDelimitedList
- AWS-Specific parameter types
- AWS SSM or AWS Secrets Manager parameter types

```yaml
Parameters:
  DomainName:
    Type: String
    Description: Domain name for API
  ZoneId:
    Type: String
    Description: Zone ID if exists.
    Default: none
  CertArn:
    Type: String
    Description: Certificate ARN if exists.
    Default: none
  DbEngine:
    # Example of using AWS parameter values
    Type: AWS::SSM::Parameter::Value<String>
    Default: /myApp/DbEngine
```

Now we can use parameters inline.

```yaml
LambdaFunction:
  Type: AWS::Serverless::Function
  Properties:
    CodeUri: src/
    Runtime: nodejs12.x
    Handler: app.lambdaHandler
    Environment:
      Variables:
        DB_ENGINE: !Ref DbEngine
        # We can use inline resolver to get the value
        DB_VERSION: '{{resolve:ssm:/myApp/DbVersion:1}}'
        DB_NAME: '{{resolve:secretsmanager:/myApp/DbName}}'
        ...
```

<strong>2. Pseudo parameters and intrinsic functions</strong>

These are in-built, most commonly used parameters that come with SAM template.

- AWS::AccountId
- AWS::NotificationARNs
- AWS::NoValue
- AWS::Partition
- AWS::Region
- AWS::StackId
- AWS::StackName
- AWS::URLSuffix

Of course, we can use CloudFormation's intrinsic functions

- Fn::Base64
- Fn::Cidr
- Fn::And
- Fn::Equals
- Fn::If
and so on...

Then, we can combine these.

```yaml
HttpApiGateway:
  Type: AWS::Serverless::HttpApi
  Properties:
    Domain:
      DomainName: !Ref DomainName
      CertificateArn: !If [CreateCert, !Ref GeneratedCert, !Ref CertArn]
      Route53:
        HostedZoneId: !If [CreateZone, !Ref GeneratedZone, !Ref ZoneId]
```

## SAM CLI Gotcha

<strong>1. About template.yaml</strong>

Once SAM builds the function, `template.yaml` in the `.aws-sam/build` folder takes precedence.

<strong>2. Debugging locally with SAM CLI in VS Code</strong>

With Node.js

1. Create a debug configuration in VSCode
2. Add a breakpoint in the code
3. Invoke lambda function using start-api, start-lambda or invoke lambda.
4. Attach the debuggerc
