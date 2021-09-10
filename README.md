# Cloud Custodian - Responsive and Detective Controls for AWS services

## Table of Contents

1. What is Cloud Custodian?
2. Cloud Custodian Learning Resources
3. Solution Overview
4. What does each control do?
5. What does a Cloud Custodian policy contain?
6. How do I deploy a cloud custodian policy?

---

## 1. What is Cloud Custodian?

Cloud Custodian enables users to be well managed in the cloud. The simple YAML DSL allows you to easily define rules to enable a well-managed cloud infrastructure, that's both secure and cost optimized. It consolidates many of the ad-hoc scripts organizations have into a lightweight and flexible tool, with unified metrics and reporting.

Cloud Custodian can provide real-time compliance by actively enforcing the security policies you define.

Cloud Custodian is open source.

## 2. Cloud Custodian Learning Resources

- [Cloud Custodian Overview](https://gitter.im/capitalone/cloud-custodian)
- [Cloud Custodian Documentation](https://cloudcustodian.io/docs/index.html)
- [Cloud Custodian GitHub](https://github.com/cloud-custodian/cloud-custodian)
- [Cloud Custodian Gitter](https://gitter.im/capitalone/cloud-custodian)
- [Cloud Custodian AWS Reference](https://cloudcustodian.io/docs/aws/resources/index.html)

## 3. Solution Overview

This solution utilizes Cloud Custodian to create Responsive Controls and Detective Controls.  Each responsive control consists of an AWS Lambda Function, CloudWatch Event Rule, and an AWS Lambda Permission.  Together, they respond to events in the AWS Cloud account and **ACT** to uphold compliance standards.

## 4. What does a each control do?

### Responsive Controls

Each responsive control is unique.  Depending on the AWS Service being monitored and the CloudWatch Event Rule, different actions will need to occur. Here's an example workflow of a responsive control:

1. <CreateBucket> API Call is made in the account.
2. Responsive Control (CloudWatch Event Rule) is triggered and Lambda Function is initiated
3. Lambda Function Code determines resource compliance
4. If Compliance check fails, Lambda Function <deletes the S3 Bucket>.
5. Sends the result to the SNS Topic

### Detective Controls

Each detective control is unique.  Depending on the AWS Service being monitored and the CloudWatch Event Rule, different actions will need to occur. Here's an example workflow of a responsive control:

1. <CreateBucket> API Call is made in the account.
2. Detective Control (CloudWatch Event Rule) is triggered and Lambda Function is initiated
3. Lambda Function Code determines resource compliance
4. When Compliance check fails/succeeds, Lambda Function forwards results to Config Rule.
5. User can view the AWS Config Rule console to determine compliance of their service resources.

## 5. What does a Cloud Custodian policy contain?

### 5a. Name, resource, description.

The overview for the cloud custodian policy. resource defines which AWS service will be monitored/reported on

Sample:

        name: s3-bucket-encryption
            resource: aws.s3
            description: >
                Event: CreateBucket. Compliance: Bucket Encryption. Remediation: Delete

### 5b. [Mode](https://cloudcustodian.io/docs/aws/resources/aws-modes.html?highlight=mode)

The "Mode" defines how cloud custodian will deploy/run the policy.  For responsive controls, we'll use **type: cloudtrail**. This mode creates a CloudWatch Event Rule and a Lambda Function. the CloudWatch event rule that listens in on CloudTrail API Calls (the "events" you specify in the policy). The CloudWatch Event Rule will trigger the lambda function. Lambda function properties (timeout, delay, tags) can be defined here.

Sample:

        mode:
            type: cloudtrail
            events:
            - CreateBucket
            role: "arn:aws:iam::{account_id}:role/\
                    custodian-responsivecontrol-role"
            timeout: 200
            delay: 20
            tags:
                CostCenter: abc
                __Exception-LambdaEncryption: "0123456789"

### 5c. [Filters](https://cloudcustodian.io/docs/aws/resources/aws-common-filters.html?highlight=filter)

Filters determine whether the monitored resource is "compliant."  Each Cloud Custodian AWS Resource will have different filters to utilize.

Sample:

        filters:
        - and:
            - type: bucket-encryption
                state: false
            - "tag:__Exception-S3Encryption": absent

### 5d. [Actions](https://cloudcustodian.io/docs/aws/resources/aws-common-actions.html?highlight=actions)

The actions available to you are dependent on the AWS resource being monitored. For Responsive Controls, we'll usually **delete** and **notify**. In the sample below, we are sending a report to an SNS topic.

Sample:

        actions:
        - type: delete
            remove-contents: false
        - type: notify
            to:
            - "arn:aws:sns:us-east-1:{account_id}:{topic-name}"
            Subject: "Bucket NONCOMPLIANT"
            Message: "encryption not enabled"
            transport:
            type: sns
            topic: {topic-name}


## 6. How do I deploy a cloud custodian policy?

This solution is under the assumption that you already have an IAM Role created ex.custodian_detective_role and/or custodian_responsive_role with permissions to all resources or fine-grained for better security.

It also assumes that you have an SNS topic created for notifications from responsive controls. This topic arn and topic name will need to be inserted into the custodian policies.

Here is the quickest and simplest way to deploy a cloud custodian policy:

1. Create a python virtual environment
2. Activate virtual environment
3. Install c7n

```bash
python3 -m venv .env
source .env/bin/activate 
python3 -m pip install c7n
```

4. Choose a control. example: d_dms_public_access.yaml
5. Update the custodian policy for that control to include your {account id}, {role name}, {topic name}
6. Run the following CLI command:

```bash
custodian run detective_controls/d_dms_public_access.yaml --output-dir .
```

This will create the AWS resources required for the detective control (lambda, cloudwatch event rule, config rule).
