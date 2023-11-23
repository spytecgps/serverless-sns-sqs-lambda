# ⚠️ DEPRECATED

> [!CAUTION]
> The only reason of this fork was to add the possibility of disable the dead letter queue. The main project already support this feature, so the fork is no longer needed.

</br>
</br>
</br>
</br>
</br>
</br>
</br>

# Serverless Sns Sqs Lambda

[![serverless](http://public.serverless.com/badges/v3.svg)](http://www.serverless.com)
[![MIT License](http://img.shields.io/badge/license-MIT-blue.svg?style=flat)](LICENSE)
![Github Actions Status](https://github.com/agiledigital/serverless-sns-sqs-lambda/workflows/Node.js%20CI/badge.svg?branch=master)
[![Type Coverage](https://img.shields.io/badge/dynamic/json.svg?label=type-coverage&prefix=%E2%89%A5&suffix=%&query=$.typeCoverage.atLeast&uri=https%3A%2F%2Fraw.githubusercontent.com%2Fagiledigital%2Fserverless-sns-sqs-lambda%2Fmaster%2Fpackage.json)](https://github.com/plantain-00/type-coverage)
[![Language grade: JavaScript](https://img.shields.io/lgtm/grade/javascript/g/agiledigital/serverless-sns-sqs-lambda.svg?logo=lgtm&logoWidth=18)](https://lgtm.com/projects/g/agiledigital/serverless-sns-sqs-lambda/context:javascript)
[![semantic-release](https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg)](https://github.com/semantic-release/semantic-release)
![npm](https://img.shields.io/npm/v/@spytecgps/serverless-sns-sqs-lambda)

This is a Serverless Framework plugin for AWS lambda Functions. Currently, it
is possible to subscribe directly to an SNS topic. However, if you want to
provide retry capability and error handling, you need to write a whole lot of
boilerplate to add a Queue and a Dead Letter Queue between the Lambda and the
SNS topic. This plugin allows you to define an sns subscriber with a `batchSize`
and a `maxRetryCount` as simply as subscribing directly to the sns topic.

# Table of Contents

- [Install](#install)
- [Setup](#setup)

## Install

Run `npm install` in your Serverless project.

`$ npm install --save-dev @spytecgps/serverless-sns-sqs-lambda`

Add the plugin to your serverless.yml file

```yml
plugins:
  - "@spytecgps/serverless-sns-sqs-lambda"
```

## Setup

Provide the lambda function with the snsSqs event, the plugin will add the AWS SNS topic and subscription, SQS queue and dead letter queue, and the role need for the lambda.

If no dead letter is configured, messages will be procesed until successed.

```yml
functions:
  processEvent:
    handler: handler.handler
    events:
      - snsSqs:
          name: TestEvent # Required - choose a name prefix for the event queue
          topicArn: !Ref Topic # Required - SNS topic to subscribe to
          omitPhysicalId: true # Optional - default value is false but recommended to be set to true for new deployments (see below)
          batchSize: 2 # Optional - default value is 10
          maximumBatchingWindowInSeconds: 10 # optional - default is 0 (no batch window)
          maxRetryCount: 2 # Optional - default value is 5
          disableDeadLetterQueue: false # Optional - default value is false
          kmsMasterKeyId: !GetAtt SQSQueueKey.Arn # optional - default is none (no encryption) - see Notes on Encryption section below
          kmsDataKeyReusePeriodSeconds: 600 # optional - AWS default is 300 seconds
          deadLetterMessageRetentionPeriodSeconds: 1209600 # optional - AWS default is 345600 secs (4 days)
          visibilityTimeout: 120 # optional (in seconds) - AWS default is 30 secs
          rawMessageDelivery: true # Optional - default value is true
          enabled: true # Optional - default value is true
          filterPolicy: # Optional - filter messages that are handled
            pets:
              - dog
              - cat

            # Overrides for generated CloudFormation templates
            # Mirrors the CloudFormation docs but uses camel case instead of title case
            #
            # https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-sqs-queues.html
            mainQueueOverride:
              maximumMessageSize: 1024
              ...
            deadLetterQueueOverride:
              maximumMessageSize: 1024
              ...
            # https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-eventsourcemapping.html
            eventSourceMappingOverride:
              sourceAccessConfigurations:
                - Type: SASL_SCRAM_256_AUTH
                  URI: arn:aws:secretsmanager:us-east-1:01234567890:secret:MyBrokerSecretName
            # https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-sns-subscription.html
            subscriptionOverride:
              region: ap-southeast-2

resources:
  Resources:
    Topic:
      Type: AWS::SNS::Topic
      Properties:
        TopicName: TestTopic

plugins:
  - "@spytecgps/serverless-sns-sqs-lambda"
```

### What is the omitPhysicalId option?

AWS allows you to omit the physical ID (queue name) for SQS resources.

In this case, the name is automatically generated by AWS based on the logical ID (the CloudFormation resource name).

This provides some benefits such as the ability to perform updates that require replacement, and not having to worry about the 80 character maximum length for queue names.

However, if you need to refer to the queue by name (which should be rare), rather than a CloudFormation reference, you might have to switch this off so that the name is stable.

This would be set to 'true' by default if this was the first version of the plugin. However, since the plugin is already in use and it could break existing stacks and may not be backwards compatible we have set this to 'false' by default. Switching this to 'true' on an existing stack has been tested and it seems to work OK, although it does create new queues and deletes the old ones. Switch the option on an existing stack at your own risk.

See also: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-name.html

### Notes on Encryption

If you choose to encrypt your SQS queue, the SNS topic will not be able to send it any messages if you use a managed key (alias/aws/sqs). This is due to an AWS limitation.

See: https://aws.amazon.com/premiumsupport/knowledge-center/sns-topic-sqs-queue-sse-kms-key-policy/

You will need to create a CMK like the following:

```yaml
# To allow SNS to push messages to an encrypted queue, a CMK must be used
SQSQueueCMK:
  Type: AWS::KMS::Key
  Properties:
    KeyPolicy:
      Version: "2012-10-17"
      Id: key-default-1
      Statement:
        - Sid: Enable IAM User Permissions
          Effect: Allow
          Principal:
            AWS: !Join
              - ""
              - - "arn:aws:iam::"
                - !Ref "AWS::AccountId"
                - ":root"
          Action: "kms:*"
          Resource: "*"
        - Sid: Allow SNS publish to SQS
          Effect: Allow
          Principal:
            Service: sns.amazonaws.com
          Action:
            - kms:GenerateDataKey
            - kms:Decrypt
          Resource: "*"
```

and then reference it in the `snsSqs` config with the `kmsMasterKeyId` attribute.

```yaml
functions:
  processEvent:
    handler: handler.handler
    events:
      - snsSqs:
          # ...
          kmsMasterKeyId: !GetAtt SQSQueueKey.Arn
          # ...
```

`kmsMasterKeyId` can either be a key ID (simple string) or an ARN or reference to to ARN. Using !Ref on a key will return a key ID and is invalid, so you'll need to use GetAtt and reference the Arn property.

### CloudFormation Overrides

If you would like to override a part of the CloudFormation template
that is generated by this plugin, you can pass raw CloudFormation
to the override config options outlined above.

The configuration must be provided with camel case keys,
but apart from that, you can use the CloudFormation config
as specified by AWS.

For example, if you wanted to override the maximumMessageSize for the main queue
you could find the "MaximumMessageSize" config option in the [AWS documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-sqs-queues.html)
make the key camel case ("maximumMessageSize") and pass it into the override section:

```yaml
    events:
      - snsSqs:
          name: Example
          ...
          mainQueueOverride:
            maximumMessageSize: 1024
```
