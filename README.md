# AWS IoT Analytics Workshop

In this workshop, you will learn about the different components of AWS IoT Analytics. You will configure AWS IoT Core to ingest stream data from the AWS Device Simulator, process batch data using Amazon ECS, build an analytics pipeline using AWS IoT Analytics, visualize the data using Amazon QuickSight, and perform machine learning using Jupyter Notebooks. Join us, and build a solution that helps you perform analytics on appliance energy usage in a smart building and forecast energy utilization to optimize consumption.

## Workshop Agenda

*   [Prerequisites](#prerequisites)
*   [Let's get started](#lets-get-started)
*   [Build the Streaming workflow](#build-the-streaming-workflow)
*   [Connect your Smart Home to AWS IoT Core](#connect-your-smart-home-to-aws-iot-core)
*   [Create Stream Analytics Pipeline](#create-stream-analytics-pipeline)
*   [Analyse Stream data](#analyse-stream-data)
*   [Build Batch Workflow](#build-batch-workflow)
*   [Create Batch Analytics Pipeline](#create-batch-analytics-pipeline)
*   [Analyse Stream and Batch data](#analyse-stream-and-batch-data)
*   [Visualize using Quicksight](#visualize-using-quicksight)
*   [Perform Forecasting with Jupyter Notebooks](#perform-forecasting-with-jupyter-notebooks)
*   [Clean Up](#clean-up)
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
*   You do not have more than 3 VPCs already deployed in the active region

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
    *  **Data Topic:** smartbuilding/topic  
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
    * Select 'smart-home'
    * **Number of Devices:** 1 -> **Submit**
    
 We have now created a simulated smart home device which is collecting power usage data and publishing that data to AWS IoT Core on the 'smartbuilding/topic' topic.

#### Verify that the data is being published to AWS IoT

**Note**: *You will use the AWS console for the remainder of the workshop. Sign-in to the [AWS console](https://aws.amazon.com/console).*

We will verify that the smart home device is configured and publishing data to the correct topic.
 1. From the AWS console, choose the **IoT Core** service
 2. Navigate to **Test** (On the left pane) 
 3. Under **Subscription**, input **Subscription topic:** 'smartbuilding/topic' and click **Subscribe to topic**

After a few seconds, you should see your simulated devices's data that is published on the 'smartbuilding/topic' MQTT topic. 

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
4. Click on **Bucket Policy** and enter the following JSON policy:
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
    * **IoT Core topic filter:** 'smartbuilding/topic' 
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
     * **Attribute Name:** _id_ and click 'Next'
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
    * **Offset:** -5 Seconds
    * **Timestamp expression:** ``from_iso8601_timestamp(timestamp)``
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
    
6. Launch the CloudFormation stack:
    *   [Launch CloudFormation stack in us-east-1](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=IoTAnalyticsStack&templateURL=https://s3.amazonaws.com/iotareinvent18/iota-reinvent-cfn-east.json) (N. Virginia)
    *   [Launch CloudFormation stack in us-west-2](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/review?stackName=IoTAnalyticsStack&templateURL=https://s3.amazonaws.com/iotareinvent18/iota-reinvent-cfn-west.json) (Oregon)
    *   [Launch CloudFormation stack in us-east-2](https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/create/review?stackName=IoTAnalyticsStack&templateURL=https://s3.amazonaws.com/iotareinvent18/iota-reinvent-cfn-east-2.json) (Ohio)
    *   [Launch CloudFormation stack in eu-west-2](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create/review?stackName=IoTAnalyticsStack&templateURL=https://s3.amazonaws.com/iotareinvent18/iota-reinvent-cfn-west-2.json) (Ireland)

After you have been redirected to the AWS CloudFormation console, take the following steps to launch the CloudFormation stack:

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

Analyse Stream and Batch data 
----------------------------

### What you will learn: Step 2c.

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

    Create Data sets - 
    On the AWS IoT Analytics console home page, in the left navigation pane, choose Analyze -> Data sets :
        a. Click Create -> SQL Data sets -> Create SQL
        b. ID - batchdataset
        c. Data store source - iotastore , Click Next
        d. SQL Query - select * from iotastore limit 5000
        e. Keep rest of the options default and Create data set
    
    Now lets execute the dataset : 
        a. Datasets -> batchdataset (click on it) -> Actions -> Run now
        b. Wait for few mins for the results to appear in the Result preview section of the screen.
      

\[[Top](#Top)\]

Bonus section
=============

### If you have time , and want to go the extra mile, Please complete this section.

Visualize using Quicksight
--------------------------

### What you will learn: Step 3.

In this section we will visualize the time series data captured from your smart home.

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

    
    1. Go to the AWS Quicksight console in North Viriginia region - 
        a. Enroll for standard edition (if you have not used it before)
        b. Click on your login user (upper right) -> Manage Quicksight -> Account Settings -> Manage Quicksight Permissions -> Check IoT Analytics -> Apply
        c. Click on Quicksight logo (upper left) to navigate to home page 
        d. Change the region to your working region now
    2. Select New Analysis -> New data set -> Choose AWS IoT Analytics
    3. Enter Data source name -> smarthome-dashboard
    4. Select an AWS IoT Analytics dataset to import - batchdataset
    5. Create data source -> Visualize
    6. Determine the home energy consumption - 
        a. Choose the sub_metering_* readings for Value axis (Choose average from Value drop down) 
        b. Choose timestamp for X axis 
    
    The graphs will look similar to below. 
    
    Please feel free to play with different fields or visual types for visualizing other smart home related information. 
    
![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/quicksight.png "Quicksight")

\[[Top](#Top)\]

Perform Forecasting with Jupyter Notebooks
------------------------------------------

### What you will learn: Step 4.

In this section we will configure the sagemaker instance to forecast energy utilisation at home.

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

    
    1. Go to the AWS IoT Analytics console.
    2. Select Analyze -> Notebooks -> Create
    3. Choose Blank Notebook
    4. Name -> smarthome_notebook
    5. Select data source -> batchdataset
    6. Select notebook instance -> IoTAWorkshopSagemaker* (check the name from Cloudformation outputs tab )
    7. Create Notebook
    8. Select IoTAWorkshopSagemaker* -> IoTAnalytics -> smarthome_notebook.ipynb -> Open in Jupyter
    9. Continue without kernel. This will open the Jupyter notebook in sagemaker. 
    10.Download the Jupyter notebook from here - https://s3.amazonaws.com/iotareinvent18/SmartHomeNotebook.ipynb
    11.Click File -> Open -> Upload (right top) -> choose Jupyter notebook downloaded in step 10 -> upload
    12.Choose conda_mxnet_p36 for kernel -> Click set kernel
    13.Step through the steps now to forecast the energy utilisation of your smart home.
      a. Click Run For each step, if there is an asterisk, that means the step is still running.
      b. Please wait for the asterisk to go away , prioir to moving to next step.
    Please read the description prior to each step in the jupyter notebook to understand the ML process.

\[[Top](#Top)\]

Clean Up
--------

**Please cleanup the resources so that you dont incurr additional charges after workshop.**

    
    1. SSH to the Ec2 docker instance
        a. Execute clean-up.sh from /home/ec2-user/clean-up directory 
    
    2. Go to the AWS Cloudformation console
        a. Delete the IoT Device Simulator stack
        b. Delete the IoTAReinvent stack
    
    3. Go to the AWS Quicksight console
        a. Click on Manage Data (upper right)
        b. Select the dataset you created earlier , Click Delete data set 
    
    4. Go to the AWS ECS console
        a. Click on Repositories (left pane)
        b. Select the Repository you created earlier (container-app-ia), Click Delete 
    
    5. Go to the AWS Ec2 console
        a. Click on Key Pairs (left pane)
        b. Select the ssk key you created earlier, Click Delete 
    
    

\[[Top](#Top)\]

Troubleshooting
---------------

**Troubleshooting for IoT Core issues. Go to the AWS IoT Core console**

    1. Get started (only if no resources are provisioned)
    2. Click upgrade to JSON logging if prompted
    3. Click on Settings in left pane
    4. Logs (if DISABLED) -> Edit
    5. Change "Disable Logging" to "Debug (most verbose)"
    6. Set role -> Create New -> iotcoreloggingrole
    7. Update

The log files from AWS IoT are send to **Amazon CloudWatch**. The AWS console can be used to look at these logs.

For additional troubleshooting, refer to here [IoT Core Troubleshooting](https://docs.aws.amazon.com/iot/latest/developerguide/iot_troubleshooting.html)

\[[Top](#Top)\]

**Troubleshooting for IoT Analytics issues. Go to the AWS IoT Analytics console**

    1. Get started (only if no resources are provisioned)
    2. Click on Settings in left pane
    3. Logs (if DISABLED) -> Edit
    4. Change "Disable Logging" to "Error"
    5. Set role -> Create new -> iotanalyticsloggingrole -> Create Role
    6. Update 

The log files from AWS IoT Analytics will be send to **Amazon CloudWatch**. The AWS console can be used to look at these logs.

For additional troubleshooting, refer to here [IoT Analytics Troubleshooting](https://docs.aws.amazon.com/iotanalytics/latest/userguide/troubleshoot.html)

\[[Top](#Top)\]
