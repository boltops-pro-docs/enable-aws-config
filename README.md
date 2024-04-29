<!-- note marker start -->
NOTE: This repo contains only the documentation for the private BoltsOps repo code.
Original file: https://github.com/boltops-pro/enable-aws-config/blob/master/README.md
The docs are publish so they are available for interested subscribers.
For access to the source code, you can become a BoltOps subscriber.
See: https://learn.boltops.com

<!-- note marker end -->

# Enable AWS Config CloudFormation Blueprint

[![BoltOps Badge](https://img.boltops.com/boltops/badges/boltops-badge.png)](https://www.boltops.com)

This blueprint can be used to enable AWS Config in a single region or multiple regions. The same infrastructure code is used in both cases. This template is useful for compliance requirements.

AWS also provides a CloudFormation StackSet example template that enables AWS Config in every region. That template creates an S3 Bucket in every region. This can be a little messy if you're only enabling AWS Config for compliance reasons and don't really use the other AWS regions. It can be preferable to only have one s3 bucket in a single region to store AWS Config results. This blueprint provides the flexibility to either use an existing S3 Bucket or create new S3 Buckets in multiple regions.

* The blueprint can be deployed a single CloudFormation stack.
* Or can be deployed via Stack Sets to multiple regions and multiple accounts.

Related Blueprints:

* [boltopspro/aws-config-aggregator](https://github.com/boltopspro/aws-config-aggregator)
* [boltopspro/aws-config-bucket](https://github.com/boltopspro/aws-config-bucket)
* [boltopspro/enable-aws-cloudtrail](https://github.com/boltopspro/enable-aws-cloudtrail)
* [boltopspro/enable-aws-config](https://github.com/boltopspro/enable-aws-config)
* [boltopspro/enable-aws-guardduty](https://github.com/boltopspro/enable-guardduty)

## Usage

1. Add blueprint to Gemfile
2. Configure: configs/enable-aws-config values
3. Deploy

## Add

Add the blueprint to your lono project's `Gemfile`.

```ruby
gem "enable-aws-config", git: "git@github.com:boltopspro/enable-aws-config.git"
```

## Configure

First you want to configure the `configs/enable-aws-config` [config files](https://lono.cloud/docs/core/configs/).  You can use [lono seed](https://lono.cloud/reference/lono-seed/) to configure starter values quickly.

    LONO_ENV=development lono seed enable-aws-config

For additional environments:

    LONO_ENV=production  lono seed enable-aws-config

The generated files in `config/enable-aws-config` folder look something like this:

    configs/enable-aws-config/
    ├── params
    │   ├── development.txt
    │   └── production.txt
    └── variables
        ├── development.rb
        └── production.rb

## Deploy: Single Account and Region

With this blueprint, you can enable AWS Config on an individual account and region basis like with the [lono cfn deploy](https://lono.cloud/reference/lono-cfn-deploy/) command. Example:

    LONO_ENV=development lono cfn deploy enable-aws-config --sure
    LONO_ENV=production  lono cfn deploy enable-aws-config --sure

However, it is common to use StackSets to enable AWS Config in multiple regions and accounts.

## Deploy: Multiple Accounts and Regions

To deploy the stack to multiple accounts, we can use [lono sets](https://lono.cloud/docs/stack-sets/lono-sets/), which is essentially CloudFormation Stack Sets.

### Create the S3 Bucket

If you want to use a single s3 bucket to be shared, instead of lots of regions, first deploy the [boltopspro/aws-config-bucket](https://github.com/boltopspro/aws-config-bucket) blueprint to create a S3 Bucket that will be shared.

    lono cfn deploy aws-config-bucket

You can use that bucket to configure the `BucketName` param:

configs/enable-aws-config/development.txt

    # Parameter Group: S3 Bucket
    BucketName=<%= stack_output("aws-config-bucket.ConfigBucket") %> # using lookup_output helper, but can also just use the bucket name

Configure the `configs/accounts` and `configs/regions` files (with your own values):

configs/accounts/development.txt

    111111111111
    222222222222

configs/regions/development.txt

    us-east-1
    us-west-2

Then you can deploy the template to multiple accounts and regions.

    lono sets deploy enable-aws-config --sure # only creates the StackSet
    lono sets instances sync enable-aws-config --sure # deploys StackSet instances - actual stacks

Deploying Stack Sets take a while as CloudFormation loops through each region one at a time. Note, using the "operation preferences" to increase the parallelism usually results in CloudFormation Rate Limit Errors and it failing. So, it is recommended running StackSets one stack at a time. Generally, StackSets are a nice feature and are helpful, but they can take a long time. Recommend using them only for simple templates.

The output looks something like this:

    $ lono sets instances sync enable-aws-config --sure
    => Running create_stack_instances on:
      accounts: 111111111111,222222222222
      regions: us-east-1,us-west-2
    Stack Instance statuses... (takes a while)
    You can check on the StackSetsole Operations Tab for the operation status.
    Here is also the cli command to check:

        aws cloudformation describe-stack-set-operation --stack-set-name enable-aws-guardduty --operation-id 92089e7a-e9f5-49c2-bef1-0ad883cdd4c1

    2020-03-29 11:18:38PM Stack Instance: account 111111111111 region us-west-2 status OUTDATED reason User initiated operation
    2020-03-29 11:18:38PM Stack Instance: account 222222222222 region us-west-2 status OUTDATED reason User initiated operation
    2020-03-29 11:18:38PM Stack Instance: account 111111111111 region us-east-1 status OUTDATED reason User initiated operation
    2020-03-29 11:18:38PM Stack Instance: account 222222222222 region us-east-1 status OUTDATED reason User initiated operation
    Stack Set Operation Status: SUCCEEDED
    Time took to complete stack set operation: 44s
    Stack Set Operation Summary:
    account 111111111111 region us-east-1 status SUCCEEDED
    account 222222222222 region us-east-1 status SUCCEEDED
    account 111111111111 region us-west-2 status SUCCEEDED
    account 222222222222 region us-west-2 status SUCCEEDED
    $

### ResourceStatusReason:Insufficient Error

If you see a `ResourceStatusReason:Insufficient` error. That means the S3 bucket you have provided does allow permission for AWS Config to deliver logs to it. Here's an example of the error:

    OUTDATED reason ResourceLogicalId:ConfigDeliveryChannel, ResourceType:AWS::Config::DeliveryChannel, ResourceStatusReason:Insufficient delivery policy to s3 bucket: aws-config-bucket-configbucket-lr7n5zsxk0yz, unable to write to bucket, provided s3 key prefix is 'null'. (Service: AmazonConfig; Status Code: 400; Error Code: InsufficientDeliveryPolicyException; Request ID: 6d139e0d-3736-4556-867b-f65611a7fda9).

And example screenshot:

![](https://img.boltops.com/boltopspro/blueprints/enable-aws-config/insufficient-permission.png)

You might have deployed the [boltopspro/aws-config-bucket](https://github.com/boltopspro/aws-config-bucket) blueprint without adjusting the `@accounts`, which adds the permission to the S3 Bucket and allows AWS Config from other accounts to deliver logs to the S3 bucket. Please double check that.

You can check the S3 Bucket Permissions / Policy to see if there is an `s3:PutObject` allowance for all the accounts you're enabling AWS Config in.

![](https://img.boltops.com/boltopspro/blueprints/enable-aws-config/check-bucket-policy.png)

## Confirm AWS Config is Enabled in All Regions

Here's a useful loop to help determine that AWS Config is enabled on all Regions:

    REGIONS=$(aws ec2 describe-regions | jq -r '.Regions[].RegionName')
    for i in $REGIONS ; do echo $i ; aws configservice describe-configuration-recorder-status --region $i ; done

Here's an example checking just 2 regions:

    $ for i in $REGIONS ; do echo $i ; aws configservice describe-configuration-recorder-status --region $i ; done
    us-west-2
    {
        "ConfigurationRecordersStatus": [
            {
                "name": "StackSet-enable-aws-config-177ce307-47c2-41ff-88e1-516bf1e50dde-ConfigRecorder-1QY1HTHZBV65E",
                "lastStartTime": 1580683128.282,
                "recording": true,
                "lastStatus": "SUCCESS",
                "lastStatusChangeTime": 1585496343.32
            }
        ]
    }
    us-east-1
    {
        "ConfigurationRecordersStatus": []
    }
    $

We can see that us-west-2 has AWS Config enabled and us-east-1 does not.

Note: The `eu-north-1` region does currently support StackSets. You can deploy the same code with [lono cfn deploy](https://lono.cloud/reference/lono-cfn-deploy/) separately to that region to enable AWS Config.
