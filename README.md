# Enable REST API for Amazon EMR Spark jobs using Amazon API Gateway and Amazon Cognito User Pools as authorizer

For many large enterprise organizations, providing access to a refined dataset to various business divisions can be a challenge. Oftentimes, AWS customers want to provide a secure way to other business units to run ad-hoc Amazon EMR batch jobs on a raw dataset and provide a transformed version of the data. Usually, there will be a business unit that owns the actual raw dataset and provides the transformed, ready to use dataset to other business units and charge back them based on how much data they are accessing and how much processing is required to provide the transformed data. The requesting business units do not need to know how the data is processed or what is happening behind the scenes to get this ready-to-use data set. All they care about is to provide how much data and what parameters are needed from the actual data set. At the same time, these requests need to be authenticated and authorized first by the providing business unit and provide an easy REST-compliant API service to the requested business units.

In this blog post, we will walk you through an approach that leverages Amazon Cognito and Amazon API Gateway for authentication and authorization of requests to initiate an AWS Step Functions via AWS Lambda service. Once invoked, AWS Step Functions executes series of steps, like creating an Amazon EMR cluster and running the Spark job which transforms the original data into a refined, ready-to-use data set and store the resultant, ready-to-use data set in an Amazon S3 bucket and record the status of the job in an Amazon DynamoDB table.

Here is the architecture diagram that illustrate the above-mentioned process.

![Picture1](https://user-images.githubusercontent.com/16944344/125143220-96f68800-e0df-11eb-9a6b-8ad79f0a5f4c.png)

**Solution Overview**

The steps we will follow in this blog post are:

1. Create a Virtual Private Cloud (VPC), an Amazon S3 bucket and an Amazon DynamoDB table.
2. Provision an App Client in Amazon Cognito service and create a scope for an application.
3. Create an API gateway which would invoke the AWS Step Functions via an AWS Lambda service.
4. The AWS Step Functions will create an Amazon EMR cluster and run an Amazon EMR step to execute a simple Spark program to read the predefined dataset and process it.
5. Create an AWS Lambda function which will update the DynamoDB table based on the status of the EMR spark job.
6. Create an AWS Lambda function which gets the status of submitted EMR Spark jobs.

# **Prerequisites and assumptions**

To follow the steps outlined in this blog post, you need the following:

- An AWS account that provides access to AWS services.
- The templates and code are intended to work in the US-EAST-1 region only and they are only for demonstration purpose only and not for production use.

Additionally, be aware of the following:

- We configure all services in the same VPC to simplify networking considerations.
- **Important** : The[AWS CloudFormation](https://aws.amazon.com/cloudformation/) templates and the sample code that we provide use hard-coded user names and passwords and open security groups. These are just for testing purposes and aren&#39;t intended for production use without any modifications.

# **Implementing the solution**

Complete end-to-end code implementation for the this use case is available from GitHub repo.

**1. Use the AWS CloudFormation template to configure Amazon VPC, DynamoDB and S3**

In this step, we set up a VPC, public subnet, internet gateway, route table, and a security group. The security group has one inbound access rule. The inbound rule allows access to any TCP port from any host within the same security group. We use this VPC and subnet for all other services that are created in the next steps. After creating VPC, we will create a DynamoDB table, this table will be used to keep track of the users, jobs created by those users and status of those jobs. This template also creates a standard Amazon S3 bucket with a provided bucket name to store the input data and processed data. In addition to this, we also use this bucket to store the code artifacts for Amazon Lambda functions and Spark jobs.

You can use this [downloadable](https://s3.amazonaws.com/aws-bigdata-blog/artifacts/awsblog-makeshift/cloudformations/step-1-vpc-dynamo-s3.yaml) AWS CloudFormation template to set up the previous components. To launch directly through the console, choose Launch Stack.

[![](RackMultipart20210709-4-1ktie86_html_5c142c6a9d4f35f.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=CF-Root-Makeshift-blog-Step1&amp;templateURL=https://s3.amazonaws.com/aws-bigdata-blog/artifacts/awsblog-makeshift/cloudformations/step-1-vpc-dynamo-s3.yaml)

This template takes the following parameters. The following table provides additional details.

| Parameter | Take this action |
| --- | --- |
| StackName | Provide any custom stackName -ex: aws-blog-vpc |

After you specify the template details, choose Next. On the Review page, choose Create.

When the stack launch is complete, it should return outputs similar to the following.

| Key | Value |
| --- | --- |
| StackName | Name |
| VPCID | vpc-xxxxxxxx |
| SubnetID | subnet-xxxxxxxx |
| SecurityGroup | sg-xxxxxxxxxxx |
| S3BucketName | makeshift-demo-${AWS::Region}-${AWS::AccountId} |
| S3BucketARN | arn:aws:s3:::\&lt;S3\_BUCKET\_NAME\&gt; |
| DynamodbTableArn | arn:aws:dynamodb:${AWS::Region}-${AWS::AccountId}:table/makeshift-jobstatus-table |

Make a note of the output, because you use this information in the next step. You can [view the stack outputs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-view-stack-data-resources.html) on the AWS Management Console or by using the following AWS CLI command:

$ aws cloudformation describe-stacks --stack-name _\&lt;stack\_name\&gt;_ --region us-east-1 --query &#39;Stacks[0].Outputs&#39;

**Upload the blog artifacts/Jar files and the input data file to s3 bucket**

Before running the next steps, copy the artifacts/jar files and sample data to s3 bucket. There will be total three files as following which needs to be copied from AWS public s3 bucket to your bucket created in above step -

A. makeshift-lambdas.jar (This file would have code for three AWS Lambda functions)

B. example-spark-job-1.0-SNAPSHOT-jar-with-dependencies.jar (This file would have code for spark job)

C. monroe-county-crash-data2003-to-2015.csv (This file will contain the input data for spark job)

**Command to download all artifacts from s3 bucket:**

aws s3 sync s3://aws-bigdata-blog/artifacts/awsblog-makeshift/jars/ s3://YOUR\_BUCKET\_NAME/

aws s3 sync s3://aws-bigdata-blog/artifacts/awsblog-makeshift/sample-dataset/ s3:// YOUR\_BUCKET\_NAME/

**NOTE:** Make sure you replaced the &quot;YOUR\_BUCKET\_NAME&quot; with the bucket that was created in the previous step.

**2. Use the AWS CloudFormation template to create &quot;Update Status Lambda&quot; and &quot;Get Status Lambda&quot; functions**

In this step, we will create the &quot;update status lambda function&quot; and &quot;Get Status Lambda function&quot;. The &quot;Update status Lambda&quot; function will be used to update the status of the EMR spark job in the Amazon DynamoDB table. This lambda function will be used by the AWS Step Function&#39;s state machine to periodically check the status of the running spark job and update the status accordingly in the Amazon Dynamo DB table.

The &quot;Get Status Lambda&quot; function will be used to fetch the status of the running/completed Spark jobs. This Lambda will be used by the API gateway when the user calls the &quot;getStatus&quot; API.

 You can use this [downloadable](https://s3.amazonaws.com/aws-bigdata-blog/artifacts/awsblog-makeshift/cloudformations/step-2-update-and-get-status-lambda.yaml) AWS CloudFormation template to set up the previous components. To launch directly through the console, choose Launch Stack.

[![](RackMultipart20210709-4-1ktie86_html_5c142c6a9d4f35f.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=CF-Root-Makeshift-blog-Step2&amp;templateURL=https://s3.amazonaws.com/aws-bigdata-blog/artifacts/awsblog-makeshift/cloudformations/step-2-update-and-get-status-lambda.yaml)

| Parameter | Take this action |
| --- | --- |
| StackName | Provide any custom stackName |
| LambdaFunctionName | Select the default name |

After you specify the template details, choose Next. On the options page, choose Next again. On the Review page select &quot;I acknowledge that AWS CloudFormation might create IAM resources with custom names&quot; option and click the Create button.

When the stack launch is complete, it should return outputs similar to the following.

| Key | Value |
| --- | --- |
| LambdaExecutionRoleArn | arn:aws:iam::${AWS::AccountId}:role/makeshift-aws-blog-iam-role |
| UpdateStatusLambdaArn | arn:aws:lambda:${AWS::Region}-${AWS::AccountId}:function:makeshift-aws-blog-check-emr-status-lambda |
| UpdateStatusLambdaName | makeshift-aws-blog-check-emr-status-lambda |
| GetJobStatusLambdaArn | arn:aws:lambda:${AWS::Region}-${AWS::AccountId}:function:makeshift-aws-blog-get-job-status-lambda |
| GetJobStatusLambdaName | makeshift-aws-blog-get-job-status-lambda |

**3. Use the AWS CloudFormation template to create the AWS Step Function State Machine**

In this step, we will create the step function state machine. This step function will be invoked by the &quot;Invoke Job lambda function&quot;, which will be created in the next step.

When the user invokes an API request to submit a job, first it will be authenticated by Cognito authentication service. Once the it is authenticated, then &quot;Invoke Job Lambda function&quot; will execute the Step function&#39;s state machine. Then the step function will create the EMR cluster and will run the Spark step on the Amazon EMR cluster. It will also take care of periodically invoking the &quot;Update Status lambda&quot; function to update the Spark job&#39;s status in the Amazon DynamoDB table. Once the Spark job is complete, the step function will terminate the Amazon EMR cluster and will update the final status in the Amazon Dynamo DB table before exiting. You can use this [downloadable](https://s3.amazonaws.com/aws-bigdata-blog/artifacts/awsblog-makeshift/cloudformations/step-3-state-machine.yaml) AWS Cloud Formation template to set up the previous components. To launch directly through the console, choose Launch Stack.

[![](RackMultipart20210709-4-1ktie86_html_5c142c6a9d4f35f.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=CF-Root-Makeshift-blog-Step3&amp;templateURL=https://s3.amazonaws.com/aws-bigdata-blog/artifacts/awsblog-makeshift/cloudformations/step-3-state-machine.yaml)

| Parameter | Take this action |
| --- | --- |
| StackName | Provide any custom stackName |

After you specify the template details, choose Next. On the options page, choose Next again. On the Review page select &quot;I acknowledge that AWS CloudFormation might create IAM resources with custom names&quot; option and click the Create button.

When the stack launch is complete, it should return outputs similar to the following.

| Key | Value |
| --- | --- |
| StateMachineArn | arn:aws:states:${AWS::Region}-${AWS::AccountId}:stateMachine:makeshift-demo-state-machine |

**4. Use the AWS CloudFormation template to create an App Client in Amazon Cognito service**

In this step, we will be creating the Amazon Cognito user pool and an app client in that user pool. We will also create a scope for the application and this will be assigned to the app client. For more information on how to create a Cognito app client, please check this [link](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-idp-settings.html). You can use this [downloadable](https://s3.amazonaws.com/aws-bigdata-blog/artifacts/awsblog-makeshift/cloudformations/step-4-cognito.yaml) AWS CloudFormation template to set up the previous components. To launch directly through the console, choose Launch Stack.

[![](RackMultipart20210709-4-1ktie86_html_5c142c6a9d4f35f.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=CF-Root-Makeshift-blog-Step4&amp;templateURL=https://s3.amazonaws.com/aws-bigdata-blog/artifacts/awsblog-makeshift/cloudformations/step-4-cognito.yaml)

| Parameter | Take this action | Default value |
| --- | --- | --- |
| StackName | Provide any custom stackName | NA |
| CognitoUserPoolName | Provide any custom name | my-makeshift-demo-pool |
| ResourceIdentifier | Select the name from the dropdown | myspringboot-ApiUserPoolResourceServer |
| ResourceScopeName | Select the name from the dropdown | myspringboot-AdhocRequestsScope |

After you specify the template details, choose Next. On the Review page, choose Create.

When the stack launch is complete, it should return outputs similar to the following.

| Key | Value |
| --- | --- |
| ResourceIdentifier | myspringboot-ApiUserPoolResourceServer/myspringboot-AdhocRequestsScope |
| UserPoolArn | arn:aws:cognito-idp:${AWS::Region}-${AWS::AccountId}:userpool/us-east-1\_xxxxxxxxx |
| UserPoolClientId | xxxxxxxxxxxxxxxxxxx |
| UserPoolEndpoint | https://cognito-idp.us-east-1.amazonaws.com/\&lt;UserPoolId\&gt; |
| UserPoolId | us-east-1\_xxxxxxxxxx |
| UserPoolName | us-east-1\_xxxxxxxxxx |

**5. Use the AWS CloudFormation template to create the &quot;Invoke Job Lambda&quot; Function.**

In this step, we would be creating a AWS Lambda function which would be used by the Amazon API gateway to invoke the step function and it creates the EMR cluster. After that it will also submit a Spark job.

You can use this [downloadable](https://s3.amazonaws.com/aws-bigdata-blog/artifacts/awsblog-makeshift/cloudformations/step-5-invoke-job-lambda.yaml) AWS CloudFormation template to set up the previous components. To launch directly through the console, choose Launch Stack.

[![](RackMultipart20210709-4-1ktie86_html_5c142c6a9d4f35f.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=CF-Root-Makeshift-blog-Step5&amp;templateURL=https://s3.amazonaws.com/aws-bigdata-blog/artifacts/awsblog-makeshift/cloudformations/step-5-invoke-job-lambda.yaml)

| Parameter | Take this action |
| --- | --- |
| StackName | Provide any custom stackName |
| LambdaFunctionName | Select the name from the dropdown |

After you specify the template details, choose Next. On the options page, choose Next again. On the Review page select &quot;I acknowledge that AWS CloudFormation might create IAM resources with custom names&quot; option and click the Create button.

When the stack launch is complete, it should return outputs similar to the following.

| Key | Value |
| --- | --- |
| LambdaExecutionRoleArn | arn:aws:iam::${AWS::AccountId}:role/makeshift-aws-blog-iam-role-api |
| StartAnalyticsJobLambdaArn | arn:aws:lambda:${AWS::Region}-${AWS::AccountId}:function:makeshift-aws-blog-invoke-step-functions-lambda |
| StartAnalyticsJobLambdaName | makeshift-aws-blog-invoke-step-functions-lambda |

**6. Use the AWS CloudFormation template to create the API gateway**

In this step, we create the Amazon API gateway which has two API functions: StartAnalyticsJob and getJobStatus. The first one will be used to invoke a step function which in turn executes the EMR spark job. And the second API will be used to check the status of any current or previous jobs.

You can use this [downloadable](https://s3.amazonaws.com/aws-bigdata-blog/artifacts/awsblog-makeshift/cloudformations/step-6-api-gateway.yaml) AWS CloudFormation template to set up the previous components. To launch directly through the console, choose Launch Stack.

[![](RackMultipart20210709-4-1ktie86_html_5c142c6a9d4f35f.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=CF-Root-Makeshift-blog-Step6&amp;templateURL=https://s3.amazonaws.com/aws-bigdata-blog/artifacts/awsblog-makeshift/cloudformations/step-6-api-gateway.yaml)

| Parameter | Take this action |
| --- | --- |
| StackName | Provide any custom stackName |

After you specify the template details, choose Next. On the options page, choose Next again. On the Review page select &quot;I acknowledge that AWS CloudFormation might create IAM resources with custom names&quot; option and click the Create button.

When the stack launch is complete, it should return outputs similar to the following.

| Key | Value |
| --- | --- |
| ApiGatewayExecutionRole | arn:aws:iam::${AWS::AccountId}:role/ms-api-ApiGatewayExecutionRole-xxxxxxxxxxx |
| ApiGatewayEndPoint |
 |

Make a note of the output, because you use this information in the next step. You can [view the stack outputs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-view-stack-data-resources.html) on the AWS Management Console or by using the following AWS CLI command:

$ aws cloudformation describe-stacks --stack-name _\&lt;stack\_name\&gt;_ --region us-east-1 --query &#39;Stacks[0].Outputs&#39;

# **Executing the solution**

Now that all the resources are created and ready, we can execute the solution with following steps-

**Get the Amazon Cognito App client Id, App client secret and Amazon Cognito Domain Name**

When we first created the Amazon Cognito User pool using the CloudFormation template, we are storing the Cognito id and corresponding Cognito pool name in the Amazon SSM parameter store. This is needed to tag the resources with Cognito name instead of Cognito Id. When the request is authorized by the Cognito authorizer using a valid token, we can only get the Cognito Id but we cannot use this Cognito Id for tagging the EMR resources as this does not have any meaning as it is a system generated random alphanumerical text. By getting the proper Cognito user pool name, we can assign this as a tag to the EMR cluster.

The Amazon Cognito token is needed to for the API gateway authentication. You can generate the Amazon Cognito token by using the &quot;[postman](https://www.postman.com/)&quot; tool.

Before launching the [postman](https://www.postman.com/) tool, get the Amazon Cognito App client Id and App client secret.

First login to Amazon Cognito service in your AWS account and click on &quot;Manage User Pools&quot; button and then select the user pool that was created in &quot;step 4&quot;. By default, the user pool name will be &quot;my-makeshift-demo-pool&quot;.

Once you click on the correct pool name, it will show something like below.

![Picture2](https://user-images.githubusercontent.com/16944344/125143730-3405f080-e0e1-11eb-9744-68fda70d8991.png)

Click on the App clients on the left-hand side menu as highlighted in the above picture.

![Picture3](https://user-images.githubusercontent.com/16944344/125143852-a37be000-e0e1-11eb-8925-a917e802c9f5.png)

Now get the Amazon Cognito domain name from the &quot;Domain name&quot; section as shown below.

![Picture4](https://user-images.githubusercontent.com/16944344/125143885-babacd80-e0e1-11eb-80b7-773099e88592.png)

**Getting the Amazon Cognito token:**

Open the Postman app and click on the New button. And in the &quot;Create New&quot; window, select the &quot;Collection&quot; tab as shown below.
 
 ![Picture5](https://user-images.githubusercontent.com/16944344/125143958-ef2e8980-e0e1-11eb-9836-c3c6741d854d.png)

Provide a name to the collection and under &quot;Authorization&quot;, select &quot;Basic Auth&quot; option. And provide the Username and Password. Username will be Cognito &quot;App client Id&quot; and Password will be Cognito &quot;App client secret&quot;.

![Picture6](https://user-images.githubusercontent.com/16944344/125143993-0f5e4880-e0e2-11eb-813b-f5a3558cbe70.png)

Next we need to create a &quot;Request&quot; under this &quot;Collection&quot;. Click on the three dots next to the Collection name and select &quot;Add Request&quot; as shown in the following screen shot.

![Picture7](https://user-images.githubusercontent.com/16944344/125144018-24d37280-e0e2-11eb-9a26-89f132c8e396.png)

In the &quot;Save Request&quot; window, provide the &quot;Request name&quot; and click on &quot;Save&quot; button.

In the &quot;Request&quot; window, select &quot;POST&quot; as the request type, and provide the URL. Copy the Cognito domain prefix URL and add &quot;oauth2/token&quot; at the end of the URL. The URL will be in this format.

&quot;\&lt;COGNITO\_DOMAIN\_PREFIX\_URL\&gt;/oauth2/token&quot;.

And in &quot;Authorization&quot; select &quot;Basic Auth&quot; and provide the &quot;Username&quot; and &quot;Password&quot;.

![Picture8](https://user-images.githubusercontent.com/16944344/125144056-43d20480-e0e2-11eb-88d2-1f1c5e592c66.png)

And in the &quot;Headers&quot; tab, add a new key with the below values.

&quot;key&quot; = Content-Type

&quot;value&quot; = &quot;application/x-www-form-urlencoded&quot;

In the body tab, click on &quot;raw&quot; and add &quot;grant\_type=client\_credentials&quot;. The client credentials grant will be used in this case as the request uses access token to access the resources.

Now save this configuration and click on &quot;Send&quot; button. It will display the &quot;access\_token&quot; information as shown in the below screen shot.

![Picture9](https://user-images.githubusercontent.com/16944344/125144084-5a785b80-e0e2-11eb-8b08-5ddc501d12cb.png)

**Invoking the API calls using curl command:**

As mentioned before, we have created two APIs: one to invoke a predefined EMR spark job, and the other to get the status of the job that was invoked. If you login to the Amazon API gateway web UI, you will see the below resources that are created for these two jobs.

**Resources for &quot;startAnalyticsJob&quot;:**

![Picture10](https://user-images.githubusercontent.com/16944344/125144401-644e8e80-e0e3-11eb-91e9-9ecb944276bf.png)

**Resources for &quot;getJobStatus&quot;:**

![Picture12](https://user-images.githubusercontent.com/16944344/125150125-e26e5d80-e102-11eb-8018-33e2ca7f9e39.png)

Before we invoke APIs, first let&#39;s get the API Gateway URLs. After executing the step 6 cloud formation template, the output will contain 2 endpoints URLS. One for invoking the batch job(StartAnalyticsJobInvokeURL) and the other one is to check the status(GetJobStatusInvokeURL) of the job.

First to invoke a batch job, use the below curl command. Change the token value and provide correct API gateway end point URL.

export TOKEN=&quot;\&lt;VALUE\_FROM\_THE\_POSTMAN\_OUTPUT\&gt;&quot;

curl -H &quot;Authorization: $TOKEN&quot; -H &quot;content-type: application/json&quot; -XPOST https://xxxxxxxxx.execute-api.{AWS\_REGION}.amazonaws.com/invoke/startAnalyticsJob

**Note:** Make sure you append &quot;startAnalyticsJob&quot; at the end of the URL.

Once you have executed the above curl command, it will return a unique job id and the output of the command will look like this.

&quot;{request\_id :2f3537cc-fc0f-4f2a-8986-db60917ad391}&quot;%

When the user invokes the startAnalyticsJob API function, first it authorizes the token, then invokes the backend Amazon Lambda function. Before the lambda function invokes the Step function, it does the following.

From the Amazon Cognito token, we can get the Cognito Id, and we use this Cognito Id to get the Cognito Pool name from the Amazon SSM Parameter Store. We use this Cognito Pool name to set the EMR cluster tags. This way we can identify who invoked the API function. These tags can be used to calculate the cost incurred by running the Spark jobs. This way we can charge back the corresponding users/departments within the organization.

If you login to Step functions web UI, you will see the below state machine graph definition. When the user invokes the &quot;startAnalyticsJob&quot; API call, first it creates an Amazon EMR cluster and at the same time, it stores the job information in the Dynamo DB table. Once the EMR cluster creation is complete, it will start the EMR step function which executes the EMR Spark job. At the same time, for every 60 seconds, the state machine checks the status of the EMR Spark job execution. When the job is completed, it updates the status in the dynamo DB table with &quot;Failed&quot; or &quot;Success&quot; as status depending on the result.

![Picture13](https://user-images.githubusercontent.com/16944344/125150139-0cc01b00-e103-11eb-9911-e6093e0e25f4.png)

To get the status of the job, get the job id from the above and run the below curl command.

export TOKEN=&quot;\&lt;VALUE\_FROM\_THE\_POSTMAN\_OUTPUT\&gt;&quot;

curl -H &quot;Authorization: $TOKEN&quot; -H &quot;content-type: application/json&quot; -XPOST [https://xxxxxxxxxxx.execute-api.us-east-1.amazonaws.com/status/getJobStatus\?jobid\=\&lt;JOB\_ID\_FROM\_THE\_ABOVE\_INVOKE\_API](https://xxxxxxxxxxx.execute-api.us-east-1.amazonaws.com/status/getJobStatus%5C?jobid%5C=%3CJOB_ID_FROM_THE_ABOVE_INVOKE_API)\&gt;

The curl command shows the below output with the details about EMR cluster ID, Cognito user id that was used to invoke this job, and the status of the job.

&quot;JobStatus StepID {S: s-HA3HVHWQ1GV,} ClusterID {S: j-1GZ8DIG6VCP57,} CognitoID {S: 17d4nb2p4n3kurv3r4sc0d5ev9,} StepStatusState {S: SUCCESS,} JobID {S: 2f3537cc-fc0f-4f2a-8986-db60917ad391,} &quot;%

If you want to find the status of all the jobs that were completed before, you can invoke the same API function without any JOB ID at the end. This will list all the completed jobs and their statuses so far.

curl -H &quot;Authorization: $TOKEN&quot; -H &quot;content-type: application/json&quot; -XPOST [https://xxxxxxxxxxx.execute-api.us-east-1.amazonaws.com/status/getJobStatus](https://xxxxxxxxxxx.execute-api.us-east-1.amazonaws.com/status/getJobStatus%5C?jobid%5C=%3CJOB_ID_FROM_THE_ABOVE_INVOKE_API)

**Conclusion:**

In this blog post, we went through an overview of invoking a predefined Amazon EMR spark job using an API function call. We also created another API to give the status of the running or completed jobs. This blog also explains how we can authenticate and authorize the requests that will execute predefined EMR Spark batch jobs. The same procedure can be used to invoke Amazon Athena or Amazon Glue jobs as well. We also went through the process of assigning the tags to EMR resources based on the user who requested the job and this can be used to calculate the cost incurred by the user/department.
