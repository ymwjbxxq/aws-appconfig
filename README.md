# AWS AppConfig #

[AWS AppConfig](https://docs.aws.amazon.com/appconfig/latest/userguide/what-is-appconfig.html) helps simplify the following tasks:

**Configure**

Source your configurations from Amazon Simple Storage Service (Amazon S3), AWS AppConfig hosted configurations, Parameter Store, Systems Manager Document Store. Use AWS CodePipeline integration to source your configurations from Bitbucket Pipelines, GitHub, and AWS CodeCommit.

**Validate**

While deploying application configurations, a simple typo could cause an unexpected outage. Prevent errors in production systems using AWS AppConfig validators. AWS AppConfig validators provide a syntactic check using a JSON schema or a semantic check using an AWS Lambda function to ensure that your configurations deploy as intended. Configuration deployments only proceed when the configuration data is valid.

**Deploy and monitor**

Define deployment criteria and rate controls to determine how your targets receive the new configuration. Use AWS AppConfig deployment strategies to set deployment velocity, deployment time, and bake time. Monitor each deployment to proactively catch any errors using AWS AppConfig integration with Amazon CloudWatch Events. If AWS AppConfig encounters an error, the system rolls back the deployment to minimize impact on your application users.

### S3 vs Parameter Store vs Lambda Layers vs AppConfig  ###

| Service | Lambda/ EC2 | Max size | Cost  | Extra |
| --------|---------|-------|----------|-------|
| S3 | yes | - | S3 pricing |     | 
| Parameters Store | yes | 4K (free tier) 8K | See System Manager pricing | |
| Lambda Layer | Only lambda | - | - | |
| AppConfig hosted configuration | yes | 64K | Free | Deployment Strategies, Validation, Rollback features |

### AppConfig snipt ###

```
AWSTemplateFormatVersion: '2010-09-09' 
Transform: 'AWS::Serverless-2016-10-31' 
Description: 'Super project' 

Parameters: 
  jsonS3Location: 
    Type: String 
    Description: "For example: s3://bucket/file.json" 

Resources: 
  MyTestApplication: 
    Type: AWS::AppConfig::Application 
    Properties:  
      Name: applicationTest 

  ConfigurationProfileTest: 
    Type: AWS::AppConfig::ConfigurationProfile 
    Properties:  
      ApplicationId: !Ref MyTestApplication 
      LocationUri: hosted 
      Name: myConfigurationProfile 

  BasicHostedConfigurationVersion: 
    Type: AWS::AppConfig::HostedConfigurationVersion 
    Properties: 
      ApplicationId: !Ref MyTestApplication 
      ConfigurationProfileId: !Ref ConfigurationProfileTest 
      Description: 'A sample hosted configuration version' 
      Content: '{ 
        "myConfig": { 
          "prop1": true, 
          "prop2": "ciao", 
          "prop3": 100000 
        } 
      }' 
      ContentType: application/json 

  MyTestDeploymentStrategy: 
    Type: AWS::AppConfig::DeploymentStrategy 
    Properties: 
      Name: "AllAtOnce" 
      DeploymentDurationInMinutes: 0 
      FinalBakeTimeInMinutes: 0 
      GrowthFactor: 100 
      GrowthType: LINEAR 
      ReplicateTo: NONE 

  MyTestEnvironment: 
    Type: AWS::AppConfig::Environment 
    Properties: 
      ApplicationId: !Ref MyTestApplication 
      Name: myTestEnvironment 

  BasicDeployment: 
    Type: AWS::AppConfig::Deployment 
    Properties: 
      ApplicationId: !Ref MyTestApplication 
      EnvironmentId: !Ref MyTestEnvironment 
      DeploymentStrategyId: !Ref MyTestDeploymentStrategy 
      ConfigurationProfileId: !Ref ConfigurationProfileTest 
      ConfigurationVersion: !Ref BasicHostedConfigurationVersion 
    DependsOn:  
      - BasicHostedConfigurationVersion 
```

Some explanation: 

* AWS::AppConfig::Application defines the “application”, the component where we group all our configurations 
* AWS::AppConfig::ConfigurationProfile defines the configuration, what in the end we will refer from the lambda to get the json 
* AWS::AppConfig::HostedConfigurationVersion is the “glue” that connects a configuration, an application the JSON that defines the configuration. As we mentioned above it is possible to avoid hardcoding values, by using transformations and S3 for the uploading phase only. We will show this in section 5. 
* AWS::AppConfig::DeploymentStrategy a deployment strategy defines important criteria for rolling out your configuration to the designated targets. A deployment strategy includes: the overall duration required, a percentage of targets to receive the deployment during each interval, an algorithm that defines how percentage grows, and bake time. Can be linear or exponential. 
* AWS::AppConfig::Environment defines a logical deployment group of AppConfig targets (for example you can define a logical structure like Web, Mobile, Back-end etc., to separate environments configurations). 
* AWS::AppConfig::Deployment starts the deployment of the configuration. 

Very important is to highlight that the limitations that CloudFormation has: 

* The documented way to pass the configuration content (in our case JSON) is to hardcode it in the template  
* It does not support loading of JSON files from your local pipeline or filesystem, but only hardcoded texts of max 4K.

**Workaround**

The workaround is by using CloudFormation Transformations and using an S3 Bucket for the loading phase:

* Add an extra string in the JSON "Content: |", to be processed by the transformation (see next subsection) 
* Upload the JSON to S3
* Provide the link to the CloudFormation to upload this file in the transformation 
* After the deployment is done, the bucket can be removed 

so now you can have 

```
BasicHostedConfigurationVersion: 
    Type: AWS::AppConfig::HostedConfigurationVersion 
    Properties: 
      ApplicationId: !Ref MyTestApplication 
      ConfigurationProfileId: !Ref ConfigurationProfileTest 
      Description: 'A sample hosted configuration version' 
      Fn::Transform: 
          Name: AWS::Include 
          Parameters: 
            Location: !Sub s3://${configBucketName}/${configurationName}.json 
      ContentType: application/json 

```

### Lambda snipt ###

The integration with the AWS Lamba is instead extremely easy. AWS AppConfig works as a Lamba Integration, running as an HTTP Server in localhost. Each request from a Lambda directed to AppConfig goes directly to the HTTP server, which is forwarding the message to AppConfig, retrieving the information and send it back. Caching is also possible, to improve the performances

```javascript
const AWS = require('aws-sdk'); 
const appconfig = new AWS.AppConfig(); 
const http = require('http'); 

exports.handler = async (event) => { 
  const result = await new Promise((resolve, reject) => { 
    http.get("http://localhost:2772/applications/applicationTest/environments/ myTestEnvironment/configurations/myConfigurationProfile", resolve); 
  }); 

  const configData = await new Promise((resolve, reject) => { 
    let data = ''; 
    result.on('data', chunk => data += chunk); 
    result.on('error', err => reject(err)); 
    result.on('end', () => resolve(data)); 
  }); 
 
  console.log(configData); 
  return JSON.parse(configData); 
}; 
```

### Is it worth? ###

YES

### Credits ###

Big thank to Fabio Fava my co-worker for the collaboration of this PoC