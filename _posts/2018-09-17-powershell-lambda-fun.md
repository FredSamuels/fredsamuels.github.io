---
layout: post
title: Creating a PowerShell Lambda-backed Custom Resource for AWS CloudFormation
description: Fun with PowerShell, Lambda, Secrets Manager and VaporShell
tags: [powershell, aws, cloudformation, lambda, vaporshell]
comments: true
excerpt_separator: <!--more-->
---

> "Changes call for innovation, and innovation leads to progress." - Li Keqiang

<!--more-->

```powershell
Write-Host "Sending response back to CloudFormation"
Invoke-WebRequest -Uri $([Uri]$CFNEvent.ResponseURL) -Method Put -Body $($body|ConvertTo-Json -Depth 5)
```

[Recently](https://aws.amazon.com/blogs/developer/announcing-lambda-support-for-powershell-core/), AWS announced Lambda support for PowerShell Core. Being primarily a PowerShell developer myself, this was incredibly welcomed (and long awaited) news!

Lambda-backed custom resources have been a huge help for performing tasks that aren't accomplishable with CloudFormation alone. The documentation around using PowerShell Lambdas as custom resources with CloudFormation specifically doesn't exist (yet), so I wanted to see if I could get it working.

***

* [excerpt_separator: ](#excerptseparator)
* [What are we building?](#what-are-we-building)
* [Prerequisites](#prerequisites)
* [The Lambda](#the-lambda)
  * [Creating the PowerShell Lambda script from template](#creating-the-powershell-lambda-script-from-template)
  * [Adding our Lambda code](#adding-our-lambda-code)
  * [Creating an IAM Role for our Lambda](#creating-an-iam-role-for-our-lambda)
  * [Publishing the Lambda to AWS](#publishing-the-lambda-to-aws)
* [Adding our secret to Secrets Manager](#adding-our-secret-to-secrets-manager)
* [The CloudFormation stack](#the-cloudformation-stack)
  * [Resulting JSON](#resulting-json)
  * [Resulting YAML](#resulting-yaml)
* [Wrapping up](#wrapping-up)
{:toc}

## What are we building?

In this post, we'll be building the following in AWS:

* PowerShell Core Lambda to get secrets from AWS Secrets Manager
* Secret in AWS Secrets Manager containing our RDSMaster Password
* CloudFormation stack containing a custom resource to call the Lambda and SQL Server Express RDS instance that uses the RDSMaster Password returned from the Lambda

We'll be doing this completely from PowerShell as well!

## Prerequisites

* An AWS account with CLI credentials
* The [.NET Core 2.1 SDK](https://www.microsoft.com/net/download)
* If you are on Windows, you will need [PowerShell Core installed](https://docs.microsoft.com/en-us/powershell/scripting/setup/installing-powershell-core-on-windows?view=powershell-6)
* You will need the following PowerShell modules installed and available from PowerShell Core:
    * AWSPowerShell.NetCore
    * AWSLambdaPSCore
    * VaporShell

## The Lambda

### Creating the PowerShell Lambda script from template

The first thing we'll need to do is create the PowerShell script for our Lambda from one of their provided templates. You technically do not need to use one of the provided templates before publishing, but it's helpful to gain ideas around how to use PowerShell with Lambda.

`Set-Location` to your preferred working directory and run the following commands from PowerShell to get started:

```powershell
Import-Module AWSLambdaPSCore
New-AWSPowerShellLambda -ScriptName "SecretsManagerCustomResource" -Template Basic
```

The commands above will...

1. Import the AWSLambdaPSCore module into our session
2. Create a new folder in your working directory named `SecretsManagerCustomResource` with a `readme.txt` file and a barebones PowerShell script named `SecretsManagerCustomResource.ps1` containing the following base info:

```powershell
# PowerShell script file to be executed as a AWS Lambda function.
#
# When executing in Lambda the following variables will be predefined.
#   $LambdaInput - A PSObject that contains the Lambda function input data.
#   $LambdaContext - An Amazon.Lambda.Core.ILambdaContext object that contains information about the currently running Lambda environment.
#
# The last item in the PowerShell pipeline will be returned as the result of the Lambda function.
#
# To include PowerShell modules with your Lambda function, like the AWSPowerShell.NetCore module, add a "#Requires" statement
# indicating the module and version.

#Requires -Modules @{ModuleName='AWSPowerShell.NetCore';ModuleVersion='3.3.343.0'}

# Uncomment to send the input event to CloudWatch Logs
# Write-Host (ConvertTo-Json -InputObject $LambdaInput -Compress -Depth 5)
```

The comments in the template immediately give us some very valuable key information that we'll need for this task:

* The event details triggering the Lambda are available in the `$LambdaInput` variable and the event context is available in the `$LambdaContext` variable.
* We can write logs to CloudWatch simply by calling one of the `Write-*` cmdlets
    * The example only shows using `Write-Host`, but I can confirm that `Write-Verbose` and `Write-Error` also work great!

### Adding our Lambda code

Open up the new `SecretsManagerCustomResource.ps1` file in your favorite editor and start adding in the code. If you'd like to skip ahead, the full Lambda code is at the bottom of this section, but we'll dive through each piece along the way.

1. We will need to use the `AWSPowerShell.NetCore` module within our Lambda to get secrets from Secrets Manager, so let's include the `#Requires` section on top:

    ```powershell
    #Requires -Modules @{ModuleName='AWSPowerShell.NetCore';ModuleVersion='3.3.343.0'}
    ```
2. We may be sending CloudFormation events across accounts via SNS, so let's get the actual CloudFormation event details if the source is SNS and store it in the `$CFNEvent` variable:

    ```powershell
    $CFNEvent = if ($null -ne $LambdaInput.Records) {
        Write-Host 'Parsing CloudFormation event from SNS message'
        $LambdaInput.Records[0].Sns.Message
    }
    else {
        $LambdaInput
    }
    ```
3. We need to send the response back to CloudFormation via web request (`Invoke-WebRequest`/`Invoke-RestMethod`), so we'll add a request body base and store that in the `$body` variable. We'll assume the request was successful and overwrite it if a failure does occur:

    ```powershell
    $body = @{
        Status             = "SUCCESS"
        Reason             = "See the details in CloudWatch Log Stream:`n[Group] $($LambdaContext.LogGroupName)`n[Stream] $($LambdaContext.LogStreamName)"
        PhysicalResourceId = $LambdaContext.LogStreamName
        StackId            = $CFNEvent.StackId
        RequestId          = $CFNEvent.RequestId
        LogicalResourceId  = $CFNEvent.LogicalResourceId
    }
    ```
4. Next, we'll take action based on the RequestType `[Delete|Update|Create]`. For this Lambda's use-case, we'll skip action if the RequestType is `Delete` to signal success immediately, since this Lambda is only for retrieving secrets during `Create` or `Update` requests. We'll update our `$body` contents with the results and set the status to `FAILED` if we hit any errors during secret retrieval (i.e. Secret or Key does not exist). We'll also wrap this in a `try/catch` statement for error handling:

    ```powershell
    try {
        switch ($CFNEvent.RequestType) {
            Delete {
            }
            default {
                $secretString = ConvertFrom-Json (Get-SECSecretValue -SecretId $CFNEvent.ResourceProperties.SecretId -ErrorAction Stop -Verbose).SecretString -ErrorAction Stop
                if ($secret = $secretString."$($CFNEvent.ResourceProperties.SecretKey)") {
                    $body.Data = @{Secret = $secret}
                }
                else {
                    Write-Error "Key [$($CFNEvent.ResourceProperties.SecretKey)] not found on secret [$($CFNEvent.ResourceProperties.SecretId)]"
                    $body.Status = "FAILED"
                    $body.Data = @{Secret = $null}
                }
            }
        }
    }
    catch {
        Write-Error $_
        $body.Status = "FAILED"
    }
    ```
5. Finally, we'll signal back to CloudFormation with the results using `Invoke-WebRequest`. The body needs to be a JSON string and the request method must be `Put` in order for this to work as needed. We wrap this in a `finally` statement so that the response will be sent to CloudFormation even if there is a terminating error when retrieving the Secret, preventing the stack creation or update from hanging due to no response from Lambda:

    ```powershell
    finally {
        try {
            Write-Host "Sending response back to CloudFormation"
            Invoke-WebRequest -Uri $([Uri]$CFNEvent.ResponseURL) -Method Put -Body $($body|ConvertTo-Json -Depth 5)
        }
        catch {
            Write-Error $_
        }
    }
    ```

You can find the full Lambda code here for brevity:

```powershell
#Requires -Modules @{ModuleName='AWSPowerShell.NetCore';ModuleVersion='3.3.343.0'}
$CFNEvent = if ($null -ne $LambdaInput.Records) {
    Write-Host 'Parsing CloudFormation event from SNS message'
    $LambdaInput.Records[0].Sns.Message
}
else {
    $LambdaInput
}
$body = @{
    Status             = "SUCCESS"
    Reason             = "See the details in CloudWatch Log Stream:`n[Group] $($LambdaContext.LogGroupName)`n[Stream] $($LambdaContext.LogStreamName)"
    PhysicalResourceId = $LambdaContext.LogStreamName
    StackId            = $CFNEvent.StackId
    RequestId          = $CFNEvent.RequestId
    LogicalResourceId  = $CFNEvent.LogicalResourceId
}
Write-Host "Processing RequestType [$($CFNEvent.RequestType)]"
try {
    switch ($CFNEvent.RequestType) {
        Delete {
        }
        default {
            $secretString = ConvertFrom-Json (Get-SECSecretValue -SecretId $CFNEvent.ResourceProperties.SecretId -ErrorAction Stop -Verbose).SecretString -ErrorAction Stop
            if ($secret = $secretString."$($CFNEvent.ResourceProperties.SecretKey)") {
                $body.Data = @{Secret = $secret}
            }
            else {
                Write-Error "Key [$($CFNEvent.ResourceProperties.SecretKey)] not found on secret [$($CFNEvent.ResourceProperties.SecretId)]"
                $body.Status = "FAILED"
                $body.Data = @{Secret = $null}
            }
        }
    }
}
catch {
    Write-Error $_
    $body.Status = "FAILED"
}
finally {
    try {
        Write-Host "Sending response back to CloudFormation"
        Invoke-WebRequest -Uri $([Uri]$CFNEvent.ResponseURL) -Method Put -Body $($body|ConvertTo-Json -Depth 5)
    }
    catch {
        Write-Error $_
    }
}
```

### Creating an IAM Role for our Lambda

In order for this Lambda to work as designed, it's going to need a few permissions not available within the default role options provided. Let's create a new IAM Role for this Lambda before deploying with the following policy containing these permissions:

* KMS key decryption (so we'll be able to decrypt the secret from Secrets Manager)
* CloudWatch log creation (because logging is great)
* X-Ray tracing (in case we want transaction tracing for our code)
* `GetSecretValue` permission for `secretsmanager` so we can retrieve secrets

To create this role and policy, we'll run the following commands from the `AWSPowerShell.NetCore` module

```powershell
$policyDoc = @'
{
    "Version": "2012-10-17",
    "Statement": [
        {
        "Action": [
            "kms:Decrypt"
        ],
        "Resource": [
            "arn:aws:kms:*:*:key/*"
        ],
        "Effect": "Allow"
        },
        {
        "Action": [
            "logs:CreateLogGroup",
            "logs:CreateLogStream",
            "logs:PutLogEvents"
        ],
        "Resource": [
            "arn:aws:logs:*:*:log-group:/aws/lambda/*"
        ],
        "Effect": "Allow"
        },
        {
        "Action": [
            "xray:PutTraceSegments",
            "xray:PutTelemetryRecords"
        ],
        "Resource": [
            "*"
        ],
        "Effect": "Allow"
        },
        {
        "Action": [
            "secretsmanager:GetSecretValue"
        ],
        "Resource": [
            "arn:aws:secretsmanager:*:*:secret:*"
        ],
        "Effect": "Allow"
        }
    ]
}
'@
$role = New-IAMRole -RoleName 'SecretsManagerLambdaRole' -AssumeRolePolicyDocument '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
Write-IAMRolePolicy -PolicyDocument $policyDoc -PolicyName SecretsManagerLambdaPolicy -RoleName SecretsManagerLambdaRole

Write-Host "The new role's ARN is:`n`n$($role.Arn)`n"
```

Once you have the new role created and inline policy attached, copy the Role's ARN then continue on to the next section.

### Publishing the Lambda to AWS

We're now ready to publish the Lambda to our AWS account using the `Publish-AWSPowerShellLambda` cmdlet! Run the following commands to publish your Lambda to your AWS account, replacing the example ARN with your own role's ARN first. If you are in the same session you created the role in, you can simply use the returned `$role.Arn` value.

_FYI: I make it a habit to include the `-PublishNewVersion` parameter as well when using this cmdlet, as it will publish it new the first time or update the existing code without needing to change the command._

```powershell
$roleARN = if ($role.Arn){
    $role.Arn
}
else {
    'arn:aws:iam::ACCOUNTID:role/SecretsManagerCustomResourceLambdaRole'
}
Publish-AWSPowerShellLambda -Name 'SecretsManagerCustomResource' -ScriptPath '.\SecretsManagerCustomResource\SecretsManagerCustomResource.ps1' -PublishNewVersion -IAMRoleArn $roleARN
```

You should see some output from the command as it packages and deploys the Lambda. Continue on to the next section once successfully published.

## Adding our secret to Secrets Manager

Before we create our RDS stack, we need to make sure we have a secret to retrieve! If you already have secrets in Secrets Manager you would like to use, continue on to the next section. Otherwise, we can create our desired secret using the `New-SECSecret` command. If you do not specify a customer-provided key, the default Secrets Manager KMS key will be used.

```powershell
$secretString = ConvertTo-Json -Compress @{RDSMasterPassword = 'Pa$$word!'} # Replace 'Pa$$word!' with the value you'd like to store
New-SECSecret -Name "development/RDS" -SecretString $secretString
```

## The CloudFormation stack

Now that our Lambda is deployed and our secret is stored in Secrets Manager, we can deploy our stack! I'll be building and deploying the template using [VaporShell](https://github.com/scrthq/Vaporshell), but the resulting JSON and YAML can be found below for reference.

1. Initialize the template:

    ```powershell
    $template = Initialize-Vaporshell -Description "My SQL Server RDS stack"
    ```
2. Add the custom resource and store the call to `GetAtt` in a variable for re-use:

    ```powershell
    $customResource = New-VaporResource -LogicalId "SecretsManagerCustomResource" -Type "Custom::SecretsManager" -Properties @{
        ServiceToken = (Add-FnJoin -Delimiter "" -ListOfValues 'arn:aws:lambda:',(Add-FnRef $_AWSRegion),':',(Add-FnRef $_AWSAccountId),':function:SecretsManagerCustomResource')
        SecretId = 'development/RDS'
        SecretKey = 'RDSMasterPassword'
        UpdateTrigger = $true
    }
    $secretValue = Add-FnGetAtt $customResource -AttributeName 'Secret'
    ```
3. Add the security group and its ingress rules. We'll use a handy call to `http://ipinfo.io/json` to get our current public IP so the instance is accessible from your local host once launched:

    ```powershell
    $securityGroupIngress = Add-VSEC2SecurityGroupIngress -CidrIp "$(Invoke-RestMethod http://ipinfo.io/json | Select-Object -ExpandProperty IP)/32" -FromPort '1433' -ToPort '1433' -IpProtocol 'tcp'
    $ec2SecurityGroup = New-VSEC2SecurityGroup -LogicalId 'RDSSecurityGroup' -GroupDescription 'Port 1433 access to RDS from local only' -SecurityGroupIngress $securityGroupIngress
    ```
4. Add the RDS instance. We'll want to use `DependsOn` to ensure the security group is created before the RDS instance, otherwise the RDS instance will fail to create. Since I'll be accessing this instance over public internet, I set `-PubliclyAccessible` to `$true`; if you are only accessing your instance from your own VPC/LAN, please set this to `$false` to keep your RDS instance secure:

    ```powershell
    $rdsInstance = New-VSRDSDBInstance -LogicalId "SqlServerExpress" -MasterUsername 'rdsmaster' -MasterUserPassword $secretValue -DBInstanceClass 'db.t2.micro' -PubliclyAccessible $true -Engine 'sqlserver-ex' -MultiAZ $false -StorageType 'gp2' -EngineVersion "13.00.4451.0.v1" -DBInstanceIdentifier 'cf-sqlserver-ex-1' -AllocatedStorage '25' -AvailabilityZone 'us-west-2a' -VPCSecurityGroups (Add-FnGetAtt $ec2SecurityGroup 'GroupId') -DependsOn $ec2SecurityGroup
    ```
5. Add the resource objects to the template:

    ```powershell
    $template.AddResource($customResource,$ec2SecurityGroup,$rdsInstance)
    ```
6. Lastly, deploy the template as a new stack:

    ```powershell
    New-VSStack -TemplateBody $template -StackName "my-sql-express-stack" -Confirm:$false
    ```

Full VaporShell script to create the template and deploy it as a new stack:

```powershell
$template = Initialize-Vaporshell -Description "My SQL Server RDS stack"
$customResource = New-VaporResource -LogicalId "SecretsManagerCustomResource" -Type "Custom::SecretsManager" -Properties @{
    ServiceToken = (Add-FnJoin -Delimiter "" -ListOfValues 'arn:aws:lambda:',(Add-FnRef $_AWSRegion),':',(Add-FnRef $_AWSAccountId),':function:SecretsManagerCustomResource')
    SecretId = 'development/RDS'
    SecretKey = 'RDSMasterPassword'
}
$secretValue = Add-FnGetAtt $customResource -AttributeName 'Secret'
$securityGroupIngress = Add-VSEC2SecurityGroupIngress -CidrIp "$(Invoke-RestMethod http://ipinfo.io/json | Select-Object -ExpandProperty IP)/32" -FromPort '1433' -ToPort '1433' -IpProtocol 'tcp'
$ec2SecurityGroup = New-VSEC2SecurityGroup -LogicalId 'RDSSecurityGroup' -GroupDescription 'Port 1433 access to RDS from local only' -SecurityGroupIngress $securityGroupIngress
$rdsInstance = New-VSRDSDBInstance -LogicalId "SqlServerExpress" -MasterUsername 'rdsmaster' -MasterUserPassword $secretValue -DBInstanceClass 'db.t2.micro' -PubliclyAccessible $true -Engine 'sqlserver-ex' -MultiAZ $false -StorageType 'gp2' -EngineVersion "13.00.4451.0.v1" -DBInstanceIdentifier 'cf-sqlserver-ex-1' -AllocatedStorage '25' -AvailabilityZone 'us-west-2a' -VPCSecurityGroups (Add-FnGetAtt $ec2SecurityGroup 'GroupId') -DependsOn $ec2SecurityGroup
$template.AddResource($customResource,$ec2SecurityGroup,$rdsInstance)
New-VSStack -TemplateBody $template -StackName "my-sql-express-stack" -Confirm:$false
```

### Resulting JSON

```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Description": "My SQL Server RDS stack",
  "Resources": {
    "SecretsManagerCustomResource": {
      "Type": "Custom::SecretsManager",
      "Properties": {
        "SecretKey": "RDSMasterPassword",
        "ServiceToken": {
          "Fn::Join": [
            "",
            [
              "arn:aws:lambda:",
              {
                "Ref": "AWS::Region"
              },
              ":",
              {
                "Ref": "AWS::AccountId"
              },
              ":function:SecretsManagerCustomResource"
            ]
          ]
        },
        "SecretId": "development/RDS"
      }
    },
    "RDSSecurityGroup": {
      "Type": "AWS::EC2::SecurityGroup",
      "Properties": {
        "GroupDescription": "Port 1433 access to RDS from local only",
        "SecurityGroupIngress": [
          {
            "CidrIp": "xxx.xxx.xxx.xxx/32",
            "FromPort": 1433,
            "ToPort": 1433,
            "IpProtocol": "tcp"
          }
        ]
      }
    },
    "SqlServerExpress": {
      "Type": "AWS::RDS::DBInstance",
      "Properties": {
        "MasterUsername": "rdsmaster",
        "MasterUserPassword": {
          "Fn::GetAtt": [
            "SecretsManagerCustomResource",
            "Secret"
          ]
        },
        "DBInstanceClass": "db.t2.micro",
        "PubliclyAccessible": true,
        "Engine": "sqlserver-ex",
        "MultiAZ": false,
        "StorageType": "gp2",
        "EngineVersion": "13.00.4451.0.v1",
        "DBInstanceIdentifier": "cf-sqlserver-ex-1",
        "AllocatedStorage": "25",
        "AvailabilityZone": "us-west-2a",
        "VPCSecurityGroups": [
          {
            "Fn::GetAtt": [
              "RDSSecurityGroup",
              "GroupId"
            ]
          }
        ]
      },
      "DependsOn": [
        "RDSSecurityGroup"
      ]
    }
  }
}
```

### Resulting YAML

```yml
AWSTemplateFormatVersion: '2010-09-09'
Description: My SQL Server RDS stack
Resources:
  SecretsManagerCustomResource:
    Type: Custom::SecretsManager
    Properties:
      SecretKey: RDSMasterPassword
      ServiceToken: !Join
        - ''
        - - 'arn:aws:lambda:'
          - !Ref 'AWS::Region'
          - ':'
          - !Ref 'AWS::AccountId'
          - :function:SecretsManagerCustomResource
      SecretId: development/RDS
  RDSSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Port 1433 access to RDS from local only
      SecurityGroupIngress:
        - CidrIp: xxx.xxx.xxx.xxx/32
          FromPort: 1433
          ToPort: 1433
          IpProtocol: tcp
  SqlServerExpress:
    Type: AWS::RDS::DBInstance
    Properties:
      MasterUsername: rdsmaster
      MasterUserPassword: !GetAtt 'SecretsManagerCustomResource.Secret'
      DBInstanceClass: db.t2.micro
      PubliclyAccessible: true
      Engine: sqlserver-ex
      MultiAZ: false
      StorageType: gp2
      EngineVersion: 13.00.4451.0.v1
      DBInstanceIdentifier: cf-sqlserver-ex-1
      AllocatedStorage: '25'
      AvailabilityZone: us-west-2a
      VPCSecurityGroups:
        - !GetAtt 'RDSSecurityGroup.GroupId'
    DependsOn:
      - RDSSecurityGroup
```

## Wrapping up

I hope this post has been informative! If you came here solely to learn how to use PowerShell Lambdas as custom resources for CloudFormation, here are the key points to take away from this post:

1. You must send the response back to CloudFormation using either `Invoke-WebRequest` or `Invoke-RestMethod`. Returning objects from Lambda do not signal back to CloudFormation by default
2. The request body sent back to CloudFormation must be converted to a JSON string; errors will be thrown from the CloudFormation side if left as a hashtable or PSObject


**Until next time!**

- Nate
