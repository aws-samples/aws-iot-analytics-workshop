# AWS IoT Analytics Workshop

In this workshop, you will learn about the different components of AWS IoT Analytics. You will configure AWS IoT Core to ingest stream data from the AWS Device Simulator, process batch data using Amazon ECS, build an analytics pipeline using AWS IoT Analytics, visualize the data using Amazon QuickSight, and perform machine learning using Jupyter Notebooks. Join us, and build a solution that helps you perform analytics on appliance energy usage in a smart building and forecast energy utilization to optimize consumption.

## Table of Contents

*   [Prerequisites](#prerequisites)
*   [Solution Architecture Overview](#solution-architecture-overview)
*   [Step 1a: Build the Streaming data workflow](#step-1a-build-the-streaming-data-workflow)
*   [Step 1b: Create Stream Analytics Pipeline](#step-1b-create-stream-analytics-pipeline)
*   [Step 1c: Analyse the data](#step-1c-analyse-the-data)
*   [Step 2a: Build the Batch analytics workflow](#step-2a-build-the-batch-analytics-workflow)
*   [Step 2b: Create Batch Analytics Pipeline](#step-2b-create-batch-analytics-pipeline)
*   [Step 2c: Analyse Stream and Batch data](#step-2c-analyse-stream-and-batch-data)
*   [Recap and Review - So far](#recap-and-review-what-have-we-done-so-far)
*   [Step 3: Visualize your data using Quicksight](#step-3-visualize-your-data-using-quicksight)
*   [Step 4: Machine Learning and Forecasting with Jupyter Notebooks](#step-4-machine-learning-and-forecasting-with-jupyter-notebooks)
*   [Recap and Review - What did we learn in this workshop?](#recap-and-review-what-did-we-learn-in-this-workshop)
*   [Clean up resources in AWS](#clean-up-resources-in-aws)
*   [Troubleshooting](#troubleshooting)

## Prerequisites

To conduct the workshop you will need the following tools/setup/knowledge:

*   AWS Account
*   Laptop
*   Secure shell (ssh) to login into your Docker instance (EC2)
    *   Mac OS/Linux: command lines tools are installed by default
    *   Windows
        *   Putty: ssh client: [http://www.putty.org/](http://www.putty.org/)
        *   Manual connect (ssh) to an EC2 instance from Windows with Putty: [http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html)
*   You need to have an ssh key-pair -
    *   A ssh key-pair can be generated or imported in the AWS console under EC2 -> Key Pairs
    *   Download the .pem file locally to log into the Ec2 docker instance later in the workshop
    *   For MAC / UNIX, change permissions - chmod 400 "paste-your-keypair-filename"
*   You are in one of the following regions:
    * us-east-1 (N Virgina)
    * us-east-2 (Ohio)
    * us-west-2 (Oregon)
    * eu-west-1 (Ireland)
*   You do not have more than 3 VPCs already deployed in the active region. In the workshop you will create 2 VPCs, and the limit for VPCs per region is 5.
*   You do not have more than 3 Elastic IP addresses already deloyed in the active region. In the workshop you will create 2 Elastic IP addresses for the Device Simulator, and the limit of Elastic IPs per region is 5.

## Solution Architecture Overview:

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

## Step 1a: Build the Streaming data workflow

### Launch AWS IoT Device Simulator with CloudFormation

The IoT Device Simulator allows you to simulate real world devices by creating device types and data schemas via a web interface and allowing them to connect to the AWS IoT message broker.

By choosing one of the links below you will be automatically redirected to the CloudFormation section of the AWS Console where your IoT Device Simulator stack will be launched:

*   [Launch CloudFormation stack in us-east-1](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=IoTDeviceSimulator&templateURL=https://s3.amazonaws.com/solutions-reference/iot-device-simulator/latest/iot-device-simulator.template) (N. Virginia)
*   [Launch CloudFormation stack in us-west-2](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/review?stackName=IoTDeviceSimulator&templateURL=https://s3.amazonaws.com/solutions-reference/iot-device-simulator/latest/iot-device-simulator.template) (Oregon)
*   [Launch CloudFormation stack in us-east-2](https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/create/review?stackName=IoTDeviceSimulator&templateURL=https://s3.amazonaws.com/solutions-reference/iot-device-simulator/latest/iot-device-simulator.template) (Ohio)
*   [Launch CloudFormation stack in eu-west-1](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create/review?stackName=IoTDeviceSimulator&templateURL=https://s3.amazonaws.com/solutions-reference/iot-device-simulator/latest/iot-device-simulator.template) (Ireland)

After you have been redirected to the AWS CloudFormation console, take the following steps to launch your stack:

 1. **Parameters** - Input Administrator Name & Email (An ID and password will be emailed to you for the IoT Device Simulator)
 2. **Capabilities** - Check "I acknowledge that AWS CloudFormation might create IAM resources." at the bottom of the page
 3. **Create Stack**
 4. Wait until the stack creation is complete 

The CloudFormation creation may take between **10-25 mins to complete**. In the **Outputs** section of your CloudFormation stack, you will find the Management console URL for the IoT simulator. Please **copy the url** to use in the next section.

\[[Top](#Top)\]

### Connect your Smart Home to AWS IoT Core

You will provision the smart home endpoint to publish telemetric data points to AWS IoT.

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

Please login to the IoT Device Simulator Management console (link copied from the earlier step) with the provided credentials.

Credentials for the Device Simulator will be mailed to the email address provided during CloudFormation stack creation.


#### Create the Simulated Device
 1. Navigate to **Modules** -> **Device Types** -> Click **Add Device Type**
    *  **Device Type Name:** smart-home  
    *  **Data Topic:** smarthome/house1/energy/appliances
    *  **Data Transmission Duration:** 7200000  
    * **Data Transmission Interval:** 3000  
    * **Message Payload:** Click **Add Attribute** and add the following attributes:  
        
|     Attribute Name    |            Data Type           | Float Precision | Integer Minimum Value | Integer Maximum Value |
|:---------------------:|:------------------------------:|:---------------:|:---------------------:|:---------------------:|
|     sub_metering_1    |              float             |               2 |                    10 |                   100 |
|     sub_metering_2    |              float             |               2 |                    10 |                   100 |
|     sub_metering_3    |              float             |               2 |                    10 |                    25 |
|  global_active_power  |              float             |               2 |                     1 |                     8 |
| global_reactive_power |              float             |               2 |                     5 |                    35 |
|        voltage        |              float             |               2 |                    10 |                   250 |
|       timestamp       | UTC Timestamp (Choose Default) |                 |                       |                       |

 3. Once the sample message payload shows all the attributes above, click **Save**
 4. Navigate to **Modules** -> **Widgets** -> **Add Widget** 
    * Select 'smart-home' as the **Device Type**
    * **Number of widgets:** 1 -> **Submit**
    
 We have now created a simulated smart home device which is collecting power usage data and publishing that data to AWS IoT Core on the 'smarthome/house1/energy/appliances' topic.

#### Verify that the data is being published to AWS IoT

**Note**: *You will use the AWS console for the remainder of the workshop. Sign-in to the [AWS console](https://aws.amazon.com/console).*

We will verify that the smart home device is configured and publishing data to the correct topic.
 1. From the AWS console, choose the **IoT Core** service
 2. Navigate to **Test** (On the left pane) 
 3. Under **Subscription** input the following:
     * **Subscription topic:** 'smarthome/house1/energy/appliances'
     * Click **Subscribe to topic**

After a few seconds, you should see your simulated devices's data that is published on the 'smarthome/house1/energy/appliances' MQTT topic. 

\[[Top](#Top)\]

## Step 1b: Create Stream Analytics Pipeline

In this section we will create the IoT Analytics components, analyze data and define different pipeline activities.

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

### Create your IoT Analytics S3 Storage Buckets

First you will need to create 3 S3 buckets, one for your IoT Analytics channel, one for the data store that holds your transformed data, and one for the data set that is resulted from an IoT Analytics SQL query.

1. Navigate to the **S3 Management Console**
2. Choose **Create Bucket**
    * **Bucket name:** Give your bucket a unique name (must be globally unique) and append it with '-channel'. For example: 'my-iot-analytics-channel'.
    * **Region:** The region should be the same as where you launched the Device Simulator Cloud Formation template.
3. Click **Next** and keep all options default. Click on **Create bucket** to finish the creation.
4. Repeat steps 1-3 twice more to finish creating the required buckets. Use the appendices '-datastore' and '-dataset' to differentiate the buckets.

You will also need to give appropriate permissions to IoT Analytics to access your Data Store bucket.
1. Navigate to the **S3 Management Console**
2. Click on your data store bucket ending with '-datastore'.
3. Navigate to the **Permissions** tab
4. Click on **Bucket Policy** and enter the following JSON policy (be sure to edit to include your S3 bucket name):
```
{
    "Version": "2012-10-17",
    "Id": "IoTADataStorePolicy",
    "Statement": [
        {
            "Sid": "IoTADataStorePolicyID",
            "Effect": "Allow",
            "Principal": {
                "Service": "iotanalytics.amazonaws.com"
            },
            "Action": [
                "s3:GetBucketLocation",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:ListBucketMultipartUploads",
                "s3:ListMultipartUploadParts",
                "s3:AbortMultipartUpload",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::<your bucket name here>",
                "arn:aws:s3:::<your bucket name here>/*"
            ]
        }
    ]
}
```
5.  Click **Save**

### Create the IoT Analytics Channel

Next we will create the IoT Analytics channel that will consume data from the IoT Core broker and store the data into your S3 bucket.

1. Navigate to the **AWS IoT Analytics** console.
2. In the left navigation pane, navigate to **Channels**
3. **Create** a new channel
    * **ID:** streamchannel
    * **Choose the Storage Type:** Customer Managed S3 Bucket, and choose your Channel S3 bucket created in the previous step.
    * **IAM Role:** Create New, and give your new IAM Role a name. This will give IoT Analytics the correct IAM policies to access your S3 bucket.
7. Click '**Next**' and input the following.  This step will create an IoT Rule that consumes data on the specified topic.
    * **IoT Core topic filter:** 'smarthome/house1/energy/appliances' 
    * **IAM Role:** Create New, and give your new IAM Role a name. This will give IoT Analytics the correct IAM policies to access your AWS IoT Core topic.
    * Click on **See messages** to see the messages from your smartbuilding device arriving on the topic. Ensure your device is still running in Device Simulator if you do not see any messages.
8. Click **Create Channel**

### Create the IoT Analytics Data Store for your pipeline

1. Navigate to the **AWS IoT Analytics** console.
2. In the left navigation pane, navigate to **Data stores**
3. **Create** a new data store
    * **ID:** iotastore
    * **Choose the Storage Type:** Customer Managed S3 Bucket, and choose your Data Store S3 bucket created in the previous step.
    * **IAM Role:** Create New, and give your new IAM Role a name. This will give IoT Analytics the correct IAM policies to access your S3 bucket.
4. Click 'Next' and then **Create data store**

### Create the IoT Analytics Pipeline

1. Navigate to the **AWS IoT Analytics** console.
2. In the left navigation pane, navigate to **Pipelines**
3. **Create** a new Pipeline:
    * **ID:** streampipeline
    * **Pipeline source**: streamchannel
6. Click **Next**
7. IoT Analytics will automatically parse the data coming from your channel and list the attributes from your simulated device. By default, all messages are selected.
8. Click **Next**
9. Under 'Pipeline activites' you can trasform the data in your pipeline, add, or remove attributes
10. Click **Add Activity** and choose **Calculate a message attribute** as the type.
     * **Attribute Name:** cost
     * **Formula:** ``(sub_metering_1 + sub_metering_2 + sub_metering_3) * 1.5``
13. Test your formula by clicking **Update preview** and the cost attribute will appear in the message payload below.
14. Add a second activity by clicking **Add activity** and **Remove attributes from a message**
     * **Attribute Name:** _id_ and click 'Next'. The _id_ attribute is a unique identifier coming from the Device Simulator, but adds noise to the data set.
16. Click **Update preview** and the _id_ attribute will disappear from the message payload.
17. Click 'Next'
18. **Pipeline output:** Click 'Edit' and choose 'iotastore'
19. Click **Create Pipeline** 

Your IoT Analytics pipeline is now set up.

\[[Top](#Top)\]

## Step 1c: Analyse the data

In this section, you will learn how to use IoT Analytics to extract insights from your data set using SQL over a specified time period.

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")


### Create a data set

1. Navigate to the **AWS IoT Analytics** console.
2. In the left navigation pane, navigate to **Data sets**
3. Choose **Create a data set**
4. Select **Create SQL**
    * **ID:** streamdataset
    * **Select data store source:** iotastore - this is the S3 bucket containing the transformed data created in step 1b.
7. Click **Next**
8. Keep the default SQL statement, which should read ``SELECT * FROM iotastore`` and click **Next**
9. Input the folllowing:
    * **Data selection window:** Delta time
    * **Offset:** -5 Seconds - the 5 second offset is to ensure all 'in-flight' data is included into the data set at time of execution.
    * **Timestamp expression:** ``from_iso8601_timestamp(timestamp)`` - we need to convert the ISO8601 timestamp coming from the streaming data to a standard timestamp.
12. Keep all other options as default and click **Next** until you reach 'Configure the delivery rules of your analytics results'
13. Click **Add rule**
14. Choose **Deliver result to S3**
    * **S3 bucket:** select the S3 bucket that ends with '-dataset'
13. Click **Create data set** to finalise the creation of the data set.

### Execute and save the dataset

1. Navigate to **Data sets** on the lefthand navigation pane of the AWS IoT Analytics console.
2. Click on 'streamdataset'
3. Click on **Actions** and in the dropdown menu choose **Run now**
4. On the left navigation menu, choose **Content** and monitor the status of your data set creation.
5. The results will be shown in the preview pane and saved as a .csv in the '-dataset' S3 bucket.
    

\[[Top](#Top)\]

## Step 2a: Build the Batch analytics workflow

In this section we will create an EC2 instance and docker image to batch a public dataset from S3 to an IoT Analytics data store using containers.

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

### Launch Docker EC2 Instance with CloudFormation

By choosing one of the links below you will be automatically redirected to the CloudFormation section of the AWS Console where your stack will be launched.

Before launching the CloudFormation, you will need an SSH key pair to log into the EC2 instance. If you don't have an SSH key pair you can create one by:
1. Navigate to the **EC2 console**
2. Click on **Key Pairs**
3. Click on **Create Key Pair** and input a name.
4. Save the .pem file in a directory accessible on your computer.
5. If you are running Mac or Linux, set the appropriate permissions on the keypair:
    * ``chmod 400 myec2keypair.pem``
 
 
 To launch the CloudFormation stack, choose one of the following links for your region, and follow the steps below:
 
 *   [Launch CloudFormation stack in us-east-1](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=IoTAnalyticsStack&templateURL=https://s3.amazonaws.com/iotareinvent18/iota-reinvent-cfn-east.json) (N. Virginia)
 *   [Launch CloudFormation stack in us-west-2](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/review?stackName=IoTAnalyticsStack&templateURL=https://s3.amazonaws.com/iotareinvent18/iota-reinvent-cfn-west.json) (Oregon)
 *   [Launch CloudFormation stack in us-east-2](https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/create/review?stackName=IoTAnalyticsStack&templateURL=https://s3.amazonaws.com/iotareinvent18/iota-reinvent-cfn-east-2.json) (Ohio)
 *   [Launch CloudFormation stack in eu-west-2](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create/review?stackName=IoTAnalyticsStack&templateURL=https://s3.amazonaws.com/iotareinvent18/iota-reinvent-cfn-west-2.json) (Ireland)

1. Navigate to **Parameters**
    * **SSHKeyName** - select the SSH key pair you will use to login to the EC2 instance.
2. Check the box "I acknowledge that AWS CloudFormation might create IAM resources."
3. Click **Create stack**

The CloudFormation stack will take approximately 5-7 minutes to complete launching all the necessary resources.

Once the CloudFormation has completed, navigate to the **Outputs** tab, and see the **SSHLogin** parameter. Copy this string to use when SSHing to the EC2 instance.

### Setup the docker image on EC2

1. SSH to the EC2 instance using the SSHLogin string copied from the above step.
    * Example: ``ssh -i Iotaworkshopkeypair.pem ec2-user@ec2-my-ec2-instance.eu-west-1.compute.amazonaws.com``
2. Move to the docker-setup folder
    * ``cd /home/ec2-user/docker-setup``
3. Update your EC2 instance:
    * ``sudo yum update``
4. Build the docker image:
    * ``docker build -t container-app-ia .``
5. Veryify the image is running:
    * ``docker image ls | grep container-app-ia``
    * You should see an output similar to: ``container-app-ia    latest              ad81fed784f1        2 minutes ago      534MB``
6. Create a new repository in Amazon Elastic Container Registry (ECR) using the AWS CLI (pre-built on your EC2 instance):
    * ``aws ecr create-repository --repository-name container-app-ia``
    * The output should include a JSON object which includes the item 'repositoryURI'. Copy this value into a text editor for later use.
7. Login to your Docker environment _(you need the \` for the command to work)_:
    * `` `aws ecr get-login --no-include-email` ``
8. Tag the Docker image with the ECR Repository URI:
    * `docker tag container-app-ia:latest <your repostoryUri here>:latest`
9. Push the image to ECR
    * `docker push <your repositoryUri here>`

\[[Top](#Top)\]

## Step 2b: Create Batch Analytics Pipeline

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

In this section we will create the IoT Analytics pipeline for your public data-set, analyze data and define different pipeline activities.

### Create the IoT Analytics Channel

Next we will create the IoT Analytics channel that will consume data from the IoT Core broker and store the data into your S3 bucket.

1. Navigate to the **AWS IoT Analytics** console.
2. In the left navigation pane, navigate to **Channels**
3. **Create** a new channel
    * **ID:** batchchannel
    * **Choose the Storage Type:** Service-managed store - in this step we will use an IoT Analytics managed S3 bucket, but you may specify a customer-managed bucket as in Step 1b if you wish.
4. **IoT Core topic filter:** Leave this blank, as the data source for this channel will not be from AWS IoT Core.
5. Leave all other options as default and click **Next**.
6. Click **Create Channel**

### Create the IoT Analytics Pipeline

1. Navigate to the **AWS IoT Analytics** console.
2. In the left navigation pane, navigate to **Pipelines**
3. **Create** a new Pipeline:
    * **ID:** batchpipeline
    * **Pipeline source**: batchchannel
4. Click **Next**
5. **See attributes of messages:** You will not see any data on this screen, as the data source has not been fully configured yet.
6. Click **Next**
7. Under 'Pipeline activites' you can trasform the data in your pipeline, add, or remove attributes
8. Click **Add Activity** and choose **Calculate a message attribute** as the type.
     * **Attribute Name:** cost
     * **Formula:** ``(sub_metering_1 + sub_metering_2 + sub_metering_3) * 1.5``
9. Click 'Next'
10. **Pipeline output:** Click 'Edit' and choose 'iotastore'
11. Click **Create Pipeline** 
      
Now we have created the IoT Analytics Pipeline, we can load the batch data.

### Create Container Data Set

A container data set allows you to automatically run your analysis tools and generate results. It brings together a SQL data set as input, a Docker container with your analysis tools and needed library files, input and output variables, and an optional schedule trigger. The input and output variables tell the executable image where to get the data and store the results.

1. Navigate to the **IoT Analytics Console**
2. Click on **Data sets** on the left-hand navigation pane
3. **Create** a new data set.
4. Choose **Create Container** next to **Container data sets**.
    * **ID**: container_dataset
5. Click **Next**
6. Click **Create** next to **Create an analysis without a data set**
    * **Frequency**: Not scheduled
    * Click **Next**
7. **Select analysis container and map variables**
    * Choose **Select from your Elastic Container Registry repository**
    * **Select your custom analysis container from Elastic Container Registry:** container-app-ia
    * **Select your image:** latest
8. Under **Configure the input variables of your container** add the following variables which will be passed to your Docker container and the Python script running in the instance:

| Name                  | Type   | Value          |
|-----------------------|--------|----------------|
| inputDataS3BucketName | String | iotareinvent18 |
| inputDataS3Key        | String | inputdata.csv  |
| iotchannel            | String | batchchannel   |

9. Click **Next**
10. Under **IAM Role**, click **Edit** and add the role iotAContainerRole. This role was created as part of the CloudFormation stack.
11. Configure the resources for container execution:
     * **Compute Resource**: 4 vCPUs and 16 GiB Memory
     * **Volume Size (GB)**: 2
12. Click **Next**
13. Under **Configure the results of your analytics**, keep all options as default.
14. Click **Next**
15. Leave **Configure delivery rules for analysis results** as default and click **Next**.
     * On this menu you can configure the data set to be delivered to an S3 bucket if you wish.
16. Finalize the creation of the data-set by clicking **Create data set**

### Execute the data set
1. Navigate to the **AWS IoT Analytics console**
2. In the left navigation pane, navigate to **Data sets**
3. Click on **container_dataset**
4. Choose **Actions** and **Run Now**
5. The container dataset can take up to 10 minutes to run. If there are no errors, you will see a **SUCCEEDED** message. If it fails with an error, please see the Troubleshooting section below to enable logging.

\[[Top](#Top)\]

## Step 2c: Analyse Stream and Batch data 

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

Now that we have two data sets into the same data store, we can analyse the result of combining both the container data and simulated device data.

### Create and execute the combined data set

1. Navigate to the **AWS IoT Analytics console**
2. In the left navigation pane, navigate to **Data sets**
    * **ID**: batchdataset
    * **Select data store source**: iotastore
3. Click **Next**
4. **SQL Query**: ``SELECT * FROM iotastore limit 5000``
5. Click **Next**
6. Leave the rest of the options as default and click **Next**
7. Optionally, you can choose to have your dataset be placed into an S3 bucket on the **Configure delivery rules for analysis results** pane.
8. Click **Create data set**
9. Click on your newly created **batchdataset**
10. Click on **Actions** and then **Run now**
11. The data set will take a few minutes to execute. You should see the results of the executed data set in the result preview, as well as an outputted .csv file which includes the complete data set.

\[[Top](#Top)\]

## Recap and Review: What have we done so far?

In the workshop so far you have accomplished the following:
 * Launched an IoT Device Simulator using CloudFormation
 * Used the IoT Device Simulator to simulate data coming from a home energy monitoring device.
 * Created an IoT Analytics pipeline that consumes, cleans and transforms that data.
 * Created a custom data set with SQL that can be used for further analytics.
 * Used a second CloudFormation template to launch an EC2 instance with a script to simulate a public data set.
 * Used Amazon Elastic Container Registry to host your docker image and IoT Analytics to launch that docker container to simulate a custom data anlytics workload.
 * Combined that simulated public data analytics workload with the simulated device data to make a meaningful data set.
 
The purpose of the workshop has been to show you how you can use IoT Analytics for your various data analytics workloads, whether that be from external data sets, a custom analytics workload using external tools, or consuming data directly from IoT devices in real time.

\[[Top](#Top)\]

## Step 3: Visualize your data using Quicksight

In this section we will visualize the time series data captured from your smart home.

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

### Sign up for AWS Quicksight

If you have used AWS Quicksight in the past, you can skip these steps.

1. Navigate to the **AWS Quicksight** console
2. If you have never used AWS Quicksight before, you will be asked to sign up. Be sure to choose the 'Standard' tier, and choose the correct region for your locale.
3. During the sign up phase, give Quicksight access to your Amazon S3 buckets and AWS IoT Analytics via the sign up page.

### Visualise your data set with AWS Quicksight

1. Navige to the **AWS Quicksight** console.
2. Click on **New analysis**
3. Click on **New data set**
4. Choose **AWS IoT Analytics** under FROM NEW DATA SOURCES.
5. Configure your AWS IoT Analytics data source.
    * **Data source name**: smarthome-dashboard
    * **Select an AWS IoT Analytics data set to import**: batchdataset
6. Click on **Create data source** to finalise the Quicksight dashboard data source configuration. You should see the data has also been imported into SPICE, which is the analytics engine driving AWS Quicksight.
7. Click on **Visualize** and wait a few moments until you see 'Import complete' in the upper right hand of the console. You should see all 5000 rows have been imported into SPICE.
8. Under **Fields list** choose 'timestamp' to set the X  axis for the graph.
9. Click on 'sub_metering_1', 'sub_metering_2' and 'sub_metering_3' to add them to the **Value** column. 
10. For each sub_metering_value, choose the drop-down menu and set **Aggregate** to **Average**.

The graphs will look similar to below. 
    
![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/quicksight.png "Quicksight")

You can setwith different fields or visual types for visualizing other smart home related information. 

\[[Top](#Top)\]

## Step 4: Machine Learning and Forecasting with Jupyter Notebooks.

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

In this section we will configure an Amazon Sagemaker instance to forecast energy utilisation in the home.

1. Navigate to the **AWS IoT Analytics** console.
2. Select **Notebooks** from the left-hand navigation pane.
3. Click **Create** to begin configuring a new Notebook.
4. Click **Blank Notebook** and input the following: 
    * **Name**: smarthome_notebook
    * **Select data set sources**: batchdataset
    * **Select a Notebook instance**: IoTAWorkshopSagemaker - this is the name of the notebook created with CloudFormation 
5. **Create Notebook**
6. Click on **IoTAWorkshopSagemaker**
    * Choose **IoT Analytics** in the drop down menu
    * Next to smarthome_notebook.ipynb, choose **Open in Jupyter**
7. A new Amazon Sagemaker window should pop up and a message that says "Kernel not found". Click **Continue Without Kernel**
8. Download the following Jupyter notebook: https://s3.amazonaws.com/iotareinvent18/SmartHomeNotebook.ipynb
9. In the Jupyter Notebook console, click **File** -> **Open**
10. Choose **Upload** and select the SmartHomeNotebook.ipynb notebook downloaded in step 8.
11. Click on the SmartHomeNotebook.ipynb to be taken back to the Jupyter Notebook.
12. Select **conda_mxnet_p36** as the kernel and click **Set kernel**.
13. You should see some pre-filled Python code steps in the Jupyter notebook.
14. Click **Run** for each step. Follow the documentation presented in the Notebook. The Notebook includes information about how the machine learning process works for each step.
15. Run through all the steps in the SmartHomeNotebook to understand how the machine learning training process works with Jupyter Notebooks. _**Note**: Wait until the asterisk * disappears after running each cell. If you click 'Run' before the previous step is completed, this could cause some code to fail to complete and the algorithm will fail._

\[[Top](#Top)\]

## Recap and Review: What did we learn in this workshop?

In Steps 3 and 4, we used AWS Quicksight, and Jupyter Notebooks to visualize and use machhine learning to gather additional insights into our data.

During the workshop, we saw how you can combine raw IoT data coming in real time from devices and public data sets analysed by third party tools. Using Pipelines, you can clean and standardise this data, so you can then perform advanced visualisations and analytics on the data. IoT Analytics is designed to solve the challenge of cleaning, organising, and making usable data out of hundreds, thousands, or even millions of data points.


## Clean up resources in AWS

In order to prevent incurring additonal charges, please clean up the resources created in the workshop.

1. SSH to the EC2 docker instance from step 2c and execute clean-up.sh. This script will delete the IoT Analytics channels, pipelines, and datasets.
    * Example: ``ssh -i Iotaworkshopkeypair.pem ec2-user@ec2-my-ec2-instance.eu-west-1.compute.amazonaws.com``
    * `cd /home/ec2-user/clean-up`
    * `sh clean-up.sh`
2. Navigate to the **AWS CloudFormation** console to delete the CloudFormation stacks and their associated resources.
    * Click on **IoTAnalyticsStack** and click **Delete**
    * Click on **IoTDeviceSimulator** and click **Delete**
    * _Note: Deleting the CloudFormation stacks can take several minutes._
3. Navigate to the **AWS Quicksight** console
    * Click on **Manage Data**
    * Click on **batchdataset** and then **Delete data set** then **Delete**
4. Navigate to the **Amazon ECS** console
    * Click on **Repositories** under **Amazon ECR**
    * Select **container-app-ia** and click **Delete**
5. Navigate to the **AWS EC2** console
    * Click on **Key Pairs** in the left navigation pane.
    * Choose the EC2 keypair you created to SSH to your docker instance and click **Delete**.
6. Navigate to the **S3 Management Console** and delete the following:
    * The 3 buckets you created in Step 1b ending with '-channel', '-dataset', and '-datastore'
    * Each bucket with the prefix 'iotdevicesimulator'.
    * Each bucket with the prefix 'sagemaker-<your region>'
7. Navigate to the **AWS IoT Analytics** console
   * Check that all of your pipelines, data sets, and channels have been deleted. If not, you can manually delete them from the console.

\[[Top](#Top)\]

## Troubleshooting

To aide in troubleshooting, you can enable logs for IoT Core and IoT Analytics that can be viewed in **AWS CloudWatch**.

### AWS IoT Core

1. Navigate to the **AWS IoT Core** console
2. Click on **Settings** in the left-hand pane
3. Under **Logs**, click on **Edit**
   * **Level of verbosity**: Debug (most verbose)
   * **Set role**: Click **Create New**
   * **Name:** iotcoreloggingrole
   
The log files from AWS IoT are send to **Amazon CloudWatch**. The AWS console can be used to look at these logs.

For additional troubleshooting, refer to here [IoT Core Troubleshooting](https://docs.aws.amazon.com/iot/latest/developerguide/iot_troubleshooting.html)

\[[Top](#Top)\]

### AWS IoT Analytics

1. Navigate to the **AWS IoT Analytics** console
2. Click on **Settings** in the left-hand pane
3. Under **Logs**, click on **Edit**
   * **Level of verbosity**: Debug (most verbose)
   * **Set role**: Click **Create New**
   * **Name:** iotanalyticsloggingrole
   * **Create role**
4. Click on **Update**

The log files from AWS IoT Analytics will be send to **Amazon CloudWatch**. The AWS console can be used to look at these logs.

For additional troubleshooting, refer to here [IoT Analytics Troubleshooting](https://docs.aws.amazon.com/iotanalytics/latest/userguide/troubleshoot.html)

\[[Top](#Top)\]
