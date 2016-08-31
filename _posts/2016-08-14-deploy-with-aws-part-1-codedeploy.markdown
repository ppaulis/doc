---
layout:         post
title:          "Deploy with AWS - Part 1 : CodeDeploy"
date:           2016-08-14 15:24:11
author:         'oswaldderiemaecker'
image:          '/assets/2015-03-23/pipelines-share.png'
hasDescription: true
categories:     tutorial
---
This tutorial explains how to configure continuousphp and AWS to deploy an PHP application with AWS CodeDeploy.

<!--more-->

- [Set-up the AWS environment accounts](#set-up-the-aws-environment-accounts)
- [Set-up the package S3 bucket](#set-up-the-package-s3-bucket)
- [Set-up the IAM permissions](#set-up-the-iam-permissions)
- [Set-up the PHP EC2 autoscaling](#set-up-the-php-ec2-autoscaling)
  - [Set-up the infrastructure](#set-up-the-infrastructure)
  - [Set-up the AWS CodeDeploy Agent](#set-up-the-aws-codedeploy-agent)
- [Set-up CodeDeploy](#set-up-codedeploy)
- [Set-up continuousphp](#set-up-continuousphp)
  - [Fork the sample application](#fork-the-sample-application)
  - [Application Projet Set-up](#application-project-set-up)
  - [Deployment pipeline Configuration](#deployment-pipeline-configuration)
- [Deploy the apps](#deploy-the-apps)
- [Notes](#notes)

## Set-up the AWS environment accounts

Based on your deployment strategy you may plan to deploy your application in differents environments like testing, preproduction and production.
AWS recommends the separation of responsibilities, for this you should create as many AWS account as environment you might require.

This tutorials explain the deployment in a testing environment. You can use it as a base for any other environments you might need.

So let's start and create an AWS account for your testing environment.

**To set up a new account:**

1. Open [https://aws.amazon.com/](https://aws.amazon.com/), and then choose Create an AWS Account.
2. Follow the online instructions.

## Set-up the package S3 bucket

You are now ready to create the S3 bucket in which the continuousphp package will be uploaded for AWS CodeDeploy.

**To create a bucket**

1. Sign into the AWS Management Console and open the Amazon S3 console at https://console.aws.amazon.com/s3.
2. Click Create Bucket.
3. In the Create a Bucket dialog box, in the Bucket Name box, enter the bucket name, in this example we will use **"mycompany-package-testing"**.
4. In the Region box, select a region from the drop-down list.
5. Click Create.

When Amazon S3 successfully creates your bucket, the console displays your empty bucket **"mycompany-package-testing"** in the Buckets panel. 

For more information, visit the [AWS S3 Bucket documentation](http://docs.aws.amazon.com/AmazonS3/latest/gsg/CreatingABucket.html)

## Set up the IAM permissions

To start we are going to create an IAM policy to grant continuousphp the permission to upload the package to your bucket **"mycompany-package-testing"** and communicate with CodeDeploy to deploy it.

**To create the User policy**

1. Sign in to the IAM console at https://console.aws.amazon.com/iam/ with your user that has administrator permissions.
2. In the navigation pane, choose Policies.
3. In the content pane, choose Create Policy.
4. Next to Create Your Own Policy, choose Select.
5. For Policy Name, type **testingDeployment**.
6. For Policy Document, paste the following policy.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1438004530000",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObjectAcl",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::mycompany-package-testing/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "codedeploy:Batch*",
                "codedeploy:CreateDeployment",
                "codedeploy:Get*",
                "codedeploy:List*",
                "codedeploy:RegisterApplicationRevision"
            ],
            "Resource": "*"
        }
    ]
}
```

7\. Choose Validate Policy and ensure that no errors display in a red box at the top of the screen. Correct any that are reported.

8\. Choose Create Policy.

Now let's create an IAM user with an Access Key and attach the policy we've just created.  

**To create a new User and attach the User policy**

1. Sign in to the Identity and Access Management (IAM) console at https://console.aws.amazon.com/iam/.
2. In the navigation pane, choose Users and then choose Create New Users. 
3. Enter the following user: **deployment.testing**
4. Generate an access key this user at this time with select Generate an access key for each user.
5. Choose Create.
6. Attach the policy **testingDeployment** to our user **deployment.testing**. 

And now let's create the CodeDeploy policy which grant to CodeDeploy the permissions to get informations about the infrastructure EC2 Instance.

**To create the CodeDeploy policy**

1. Sign in to the IAM console at https://console.aws.amazon.com/iam/ with your user that has administrator permissions.
2. In the navigation pane, choose Policies.
3. In the content pane, choose Create Policy.
4. Next to Create Your Own Policy, choose Select.
5. For Policy Name, type **CodeDeploy**.
6. For Policy Document, paste the following policy.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:CompleteLifecycleAction",
        "autoscaling:DeleteLifecycleHook",
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeLifecycleHooks",
        "autoscaling:PutLifecycleHook",
        "autoscaling:RecordLifecycleActionHeartbeat",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "tag:GetTags",
        "tag:GetResources"
      ],
      "Resource": "*"
    }
  ]
}
```

7\. Choose Validate Policy and ensure that no errors display in a red box at the top of the screen. Correct any that are reported.

8\. Choose Create Policy.

And finally let's create the CodeDeploy Role and attach the CodeDeploy policy.

**To create a new Role and attach the CodeDeploy policy**

1. Sign in to the Identity and Access Management (IAM) console at https://console.aws.amazon.com/iam/.
2. In the navigation pane, choose Roles and then choose Create New Roles. 
3. Enter the following user: **CodeDeploy**
5. Choose Create.
6. Attach the policy **CodeDeploy** to our **CodeDeploy** role. 

For more information, visit the [AWS IAM User documentation](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) and the [Tutorial: Create and Attach Your First Customer Managed Policy](http://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_managed-policies.html)

## Set-up the PHP EC2 autoscaling

We have set-up all the IAM permissions needed for continuousphp to put its builds package to the S3 bucket, and trigger a deployment with CodeDeploy. Also CodeDeploy has a Role that grant it access to your EC2 informations like Instance IDs, Tags, Autoscaling.

Let's create our infrastructure, for this we will use this [CloudFormation template](https://github.com/oswaldderiemaecker/continuousphp-aws-cloudformation-template).

### Set-up the infrastructure

Fork this projet and clone it locally. 

**To create a stack on the AWS CloudFormation console**

1. Log in to the AWS Management Console and select CloudFormation in the Services menu.
2. Create a new stack by using one of the following options:
  1. Click Create Stack. This is the only option if you have a currently running stack.
  2. Click Create New Stack in the CloudFormation Stacks main window. This option is visible only if you have no running stacks.
3. On the Select Template page
  1. Choose a template
  2. Upload a template to Amazon S3, select AWS CloudFormation template on your local computer. Choose the [CloudFormation template](https://github.com/oswaldderiemaecker/continuousphp-aws-cloudformation-template) that your just forked.
4. Click Next to accept your settings and proceed with specifying the stack name and parameters.

**To specify the stack parameter values**

1. On the Specify Details page, type a stack name in the Stack name box.
2. In the Parameters section
  * InstanceType: t2.nano
  * KeyName: The EC2 Key Pair to allow SSH access to the instances
  * OperatorEMail: EMail address to notify if there are any scaling operations
  * SSHLocation: The IP address range that can be used to SSH to the EC2 instances
3. When you are satisfied with the parameter values, click Next to proceed with setting options for your stack.
4. Click Next Step to proceed with reviewing your stack.
5. On the Review page, review the details of your stack.
6. After you review the stack launch settings and the estimated cost of your stack, click Create to launch your stack.

Your stack appears in the list of AWS CloudFormation stacks, with a status of **CREATE_IN_PROGRESS**.

After your stack has been successfully created, its status changes to **CREATE_COMPLETE**.
 
### Set-up the AWS CodeDeploy Agent 

If you used the CloudFormation Template above you don't need to install the AWS CodeDeploy Agent, the Amazon Machine Image (AMI) used in the Template are preconfigured by continuousphp and include it already.

If you like to make your own AMI please visit the [Install or Reinstall the AWS CodeDeploy Agent](http://docs.aws.amazon.com/codedeploy/latest/userguide/how-to-run-agent-install.html)

## Set-up CodeDeploy

We are alomost there, Let's set up CodeDeploy with the following steps:

1. Sign in to the AWS Management Console and open the AWS CodeDeploy console at https://console.aws.amazon.com/codedeploy.
2. If the Applications page does not appear, on the AWS CodeDeploy menu, choose Applications.
3. Choose Create New Application.
4. In the Application Name box, type the application's name: **mycompany_app**. (In an AWS account, an AWS CodeDeploy application name can be used only once per region. You can reuse an application name in different regions.)
5. In the Deployment Group Name box, type a name that describes the deployment group: **testing**.
6. In the list of tags, select the tag type and fill in the Key and Value boxes with the value of the key-value pair you will use to tag the instances.
As you begin adding key-value pair information, a new row appears for you to add another key-value pair if desired. You can repeat this step for up to 10 key-value pairs.
7. In the Deployment Config list, choose the deployment configuration: **OneAtATime**
8. In the Service Role ARN box, choose an Amazon Resource Name (ARN): **arn:aws:iam::00000000000:role/CodeDeploy** 

For more information, visit the [AWS CodeDeploy Deployments](http://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps.html) and [Create an Application with AWS CodeDeploy](http://docs.aws.amazon.com/codedeploy/latest/userguide/how-to-create-application.html)

## Set-up continuousphp

### Fork the Sample Application

Finally, let's configure continuousphp to deploy our [Sample Application](https://github.com/oswaldderiemaecker/aws-demo-zf-apigility-phinx).

This application include the following files:

* composer.json with our dependency (Zend Framework 2.x / Apigility / Phinx)
* build.xml with our phing tasks
* phinx.yml.dist with the Phinx Database migration configuration
* appspec.yml with CodeDeploy configuration and hooks
* behat.yml for Behat configuration
* tests/phpunit.xml for PHPUnit configuration

**To Fork our Sample Application**

1. Login to your github account.
2. Visit our [Sample Application](https://github.com/oswaldderiemaecker/aws-demo-zf-apigility-phinx).
3. Click on **Fork* button at the top right
4. Select where you want to fork the repository

### Application Projet Set-up

**To set-up our application project in continuousphp**

1. Type in the omnibar the name of the application: aws-demo
2. The fork should appear in the Project List (if not please wait a little that projects list update)
3. Click on **Setup**
4. Click on **Setup Repository**
5. Click on **+** to add a deployment pipeline
6. Select the **master** branch

### Deployment pipeline Configuration

**To configure your deployment pipeline**
 
1. In the Build Settings (Step 1):
   1. In the **PHP VERSIONS**, select the PHP versions: **5.6** / **7.0**
   2. In the **CREDENTIALS**, click on the **+**, add the AWS IAM Credential **deployment.testing* User Access Key and Secret Key we created in step [Set-up the IAM permissions](#set-up-the-iam-permissions)
      * Name: AWSCodeDeployDemo
      * Region: US West (Oregon)
      * Access Key: XXXXXXXXXXXXXXXXXXXXXXXXXXXXX
      * Secret Key: XXXXXXXXXXXXXXXXXXXXXXXXXXXXX
   3. Click on **Next** to move to the Test Settings
2. In the Test Settings (Step 2):
   1. continuousphp auto discover you have a behat.yml and phpunit.yml in your repository, it automatically setup the testing framework based on them.
   2. Click on the **Behat** configuration panel, in the **PHING** section, select the following Phing Targets: **init-db** and **setup**
   3. Still in the **PHING** section, add the following variables: 
      * db.host: 127.0.0.1
      * db.port: 3306
      * db.dbname: skeleton
      * db.username: root
      * db.password:
   4. Click on **Next** to move to the Package Settings
3. In the Package Settings (Step 3):
   1. Select **AWS Code Deploy**
   2. Click on **Next** to move to the Deployment Settings
4. In the Deployment Settings (Step 4):
   1. Click on **+** on the **DESTINATIONS** panel
   2. Complete the destination:
      * Name: testing
      * Apply to: push
      * IAM Profile: The profile we created in Step 1.2
      * Application: mycompany_app
      * Group: testing
      * S3 Bucket: mycompany-package-testing/testing

## Deploy the apps 

## Notes

Now you can [start using continuousphp](https://continuousphp.com/)  
We do hope you will enjoy using it for your projects. Any question, don’t hesitate to ask us using the chat button!