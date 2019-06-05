AWS IoT Analytics Workshop
==========================

In this workshop, you will learn about the different components of AWS IoT Analytics. You will configure AWS IoT Core to ingest stream data from the AWS Device Simulator, process batch data using Amazon ECS, build an analytics pipeline using AWS IoT Analytics, visualize the data using Amazon QuickSight, and perform machine learning using Jupyter Notebooks. Join us, and build a solution that helps you perform analytics on appliance energy usage in a smart building and forecast energy utilization to optimize consumption.

Workshop Agenda
---------------

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

Prerequisites
-------------

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

Before you start with the workshop, please ensure that -
--------------------------------------------------------

 You are in us-east-1 (N Virgina), us-east-2 (Ohio) , us-west-2 (Oregon) or eu-west-1 (Ireland) region, and you do not have more than 3 VPCs already deployed in that region.


Let's get started
=================

## Solution Architecture Overview:

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")


## Build the Streaming workflow

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

Connect your Smart Home to AWS IoT Core
---------------------------------------

### What you will learn: Step 1a.

You will provision the smart home endpoint to publish telemetric data points to AWS IoT.

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

Please login to the IoT Device Simulator Management console (link copied from the earlier step) with the provided credentials.

Credentials for the Device Simulator will be mailed to the email address provided during CloudFormation stack creation.


### Create the Simulated Device

Navigate to **Modules** -> **Device Types** -> Click **Add Device Type**
    
 1. **Device Type Name:** smart-home  
 2. **Data Topic:** smartbuilding/topic  
 3. **Data Transmission Duration:** 7200000  
 4. **Data Transmission Interval:** 3000  
 5. **Message Payload:** Click Add Attribute and add the following attributes:  
        
|     Attribute Name    |            Data Type           | Float Precision | Integer Minimum Value | Integer Maximum Value |
|:---------------------:|:------------------------------:|:---------------:|:---------------------:|:---------------------:|
|     sub_metering_1    |              float             |               2 |                    10 |                   100 |
|     sub_metering_2    |              float             |               2 |                    10 |                   100 |
|     sub_metering_3    |              float             |               2 |                    10 |                    25 |
|  global_active_power  |              float             |               2 |                     1 |                     8 |
| global_reactive_power |              float             |               2 |                     5 |                    35 |
|        voltage        |              float             |               2 |                    10 |                   250 |
|       timestamp       | UTC Timestamp (Choose Default) |                 |                       |                       |

 6. Once the sample message payload shows all the attributes above, click **Save**
 7. Navigate to **Modules** -> **Widgets** -> **Add Widget** -> Select 'smart-home' -> Number of Devices: 1 -> **Submit**
    
 We have now created a simulated smart home device which is collecting power usage data and publishing that data to AWS IoT Core on the 'smartbuilding/topic' topic.

**Use the AWS console for the remainder of the Workshop**
---------------------------------------------------------

Sign-in to the [AWS console](https://aws.amazon.com/console).

Next, we will verify that the smart home device is configured and publishing data to the correct topic.
 1. From the AWS console, choose the **IoT Core** service
 2. Navigate to **Test** (On the left pane) 
 3. Under **Subscription**, input **Subscription topic:** 'smartbuilding/topic' and click **Subscribe to topic**

After a few seconds, you should see your simulated devices's data that is published on the 'smartbuilding/topic' MQTT topic. 

\[[Top](#Top)\]

Create Stream Analytics Pipeline
--------------------------------

### What you will learn: Step 1b.

In this section we will create the IoT Analytics components, analyze data and define different pipeline activities.

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

**Go to the AWS IoT Analytics console**

    Create Channel - 
    
    On the AWS IoT Analytics console home page, in the left navigation pane, choose Prepare -> Channels :
        a. Click Create
        b. Channel ID - streamchannel (Keep all other options deafault , click Next)
        c. IoT Core topic filter - smartbuilding/topic
        d. IAM role  name - Create new -> Enter role name smarthome-role
        e. Create Channel
    
    Create Data store - 
    On the AWS IoT Analytics console home page, in the left navigation pane, choose Prepare -> Data stores :
        a. Click Create
        b. ID - iotastore (Keep all other options deafault)
        c. Click Create data store
    
    Create Pipeline - 
    On the AWS IoT Analytics console home page, in the left navigation pane, choose Prepare -> Pipeline :
        a. Click Create
        b. Pipeline ID - streampipeline
        c. pipeline source - streamchannel (Click Next)
        d. Set attributes of messages - You should see incoming messages here -> Click Next
            i.  Do not select any attributes, by default , all will be stored.
            ii. If you dont see incoming messages , check with support.  
        e. Add activity - Calculate a message attribute
            i. Attribute Name : cost
            ii.Formula : (sub_metering_1 + sub_metering_2 + sub_metering_3) * 1.5
        f. Click update preview -> Cost field should appear in outgoing message
        g. Add activity - Remove attributes from the message
            i. select Attribute name : _id_
            ii.Click Next
        h. Click update preview -> _id_ field should disappear from outgoing message 
        i. Pipeline output -  Select iotastore
        j. Click Create pipeline
    

Now we have created the IoT Analytics Pipeline lets analyze the data.

\[[Top](#Top)\]

Analyse Stream data
-------------------

### What you will learn: Step 1c.

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

    Create Data sets - 
    On the AWS IoT Analytics console home page, in the left navigation pane, choose Analyze -> Data sets :
        a. Click Create -> SQL Data sets -> Create SQL
        b. ID - streamdataset
        c. Data store source - iotastore , Click Next
        d. SQL Query - Keep the default
        e. Data Selection Window - Choose Delta Time
            i. Offset ->  -5 seconds (please ensure its negative 5)
            ii.Timestamp -> from_iso8601_timestamp(timestamp)
        d. Keep rest of the options default and Create data set
    
    Now lets execute the dataset : 
        a. Datasets ->  streamdataset (click on it) -> Actions -> Run now
        b. Wait for few mins for the results to appear in the Result preview section of the screen.
        c. If there are no results in Result preview pane , re-run the dataset again.
    

\[[Top](#Top)\]

Build Batch Workflow
------------------------

In this section we will load the public dataset in batch from S3 to IoT Analytics data store using containers.

### What you will learn: Step 2a.

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

### Launch Docker Instance with CloudFormation

By choosing one of the links below you will be automatically redirected to the CloudFormation section of the AWS Console where your stack will be launched.

#### Prior to launch the CloudFormation stack you need to have:

*   An ssh key pair to log into the EC2 instance. If you don't have an ssh key pair you can create one in the EC2 console -> Key pairs

* * *

*   [Launch CloudFormation stack in us-east-1](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=IoTAnalyticsStack&templateURL=https://s3.amazonaws.com/iotareinvent18/iota-reinvent-cfn-east.json) (N. Virginia)
*   [Launch CloudFormation stack in us-west-2](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/review?stackName=IoTAnalyticsStack&templateURL=https://s3.amazonaws.com/iotareinvent18/iota-reinvent-cfn-west.json) (Oregon)
*   [Launch CloudFormation stack in us-east-2](https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/create/review?stackName=IoTAnalyticsStack&templateURL=https://s3.amazonaws.com/iotareinvent18/iota-reinvent-cfn-east-2.json) (Ohio)
*   [Launch CloudFormation stack in eu-west-2](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create/review?stackName=IoTAnalyticsStack&templateURL=https://s3.amazonaws.com/iotareinvent18/iota-reinvent-cfn-west-2.json) (Ireland)

After you have been redirected to the AWS CloudFormation console take the following steps to launch you stack:

    1. Parameters
    2. Select SSHKeyName
    3. Capabilities -> check "I acknowledge that AWS CloudFormation might create IAM resources." at the bottom of the page
    4. Create
    5. Wait until the complete stack is created 

The Cloudformation will take 5-7 mins to complete. In the **Outputs** section of your CloudFormation stack you will find the **ssh login string** for your EC2 instance.

#### Create the docker image -

    1. ssh to docker instance : 
        a. cd /home/ec2-user/docker-setup
    2. Build the docker image : (Don't miss the dot at the end)
        a. docker build -t container-app-ia .
    3. You should see a new image in your Docker repo. Verify it by running:
        a. docker image ls | grep container-app-ia
    4. Create a new repository in ECR:
        a.aws ecr create-repository --repository-name container-app-ia
        b.Copy the repositoryUri in text editor for use in step 6 & 7 
    5. Login to your Docker environment: (Dont miss the Tilda)
        a. `aws ecr get-login --no-include-email`
    6. Tag the image you created with the ECR Repository Tag:
        a. docker tag container-app-ia:latest paste-repositoryUri-copied-in-step4:latest
    7. Push the image to ECR
        a. docker push "paste-repositoryUri-copied-in-step4"
    

\[[Top](#Top)\]

Create Batch Analytics Pipeline
-------------------------------

### What you will learn: Step 2b.

![alt text](https://github.com/aws-samples/aws-iot-analytics-workshop/blob/master/images/arch.png "Architecture")

In this section we will create the IoT Analytics components, analyze data and define different pipeline activities.

**Go to the AWS IoT Analytics console**

    Create Channel - 
      
      On the AWS IoT Analytics console home page, in the left navigation pane, choose Prepare -> Channels :
          a. Click Create
          b. Channel ID - batchchannel (Keep all other options deafault , click Next)
          c. IoT Core topic filter - keep it blank
          d. Create Channel
      
      Create Pipeline - 
      On the AWS IoT Analytics console home page, in the left navigation pane, choose Prepare -> Pipeline :
          a. Click Create
          b. Pipeline ID - batchpipeline
          c. pipeline source - batchchannel, Click Next
          d. Set attributes of messages - You WON'T See any incoming messsages here -> Click Next
          e. Add activity - Calculate a message attribute 
              i. Attribute Name : cost
              ii.Formula : (sub_metering_1 + sub_metering_2 + sub_metering_3) * 1.5
              Click Next
          f. Pipeline output -  Select iotastore
          g. Click Create pipeline
      

Now we have created the IoT Analytics Pipeline lets load the batch data.

#### Create Container Data Set

A container data set allows you to automatically run your analysis tools and generate results. It brings together a SQL data set as input, a Docker container with your analysis tools and needed library files, input and output variables, and an optional schedule trigger. The input and output variables tell the executable image where to get the data and store the results.

Navigate to AWS IoT Analytics console, in the left navigation pane, choose Analyze.

     
    1. Click on Analyze -> Data Sets → Create
    2. Choose Container Data Sets → Create Container 
    3. Choose a unique ID for the Container Data Set → container_dataset, click Next
    4. Choose the option - >  Create an analysis without a dataset -> Create
    5. Frequency → Keep default (Not scheduled) -> Click Next
    6. Select from your  ECR Repository → Choose the repository container-app-ia
    7. Select your image → Choose the image with *latest* tag
    8. Configure the input variables (as below) -> Click Next
    
         | Name                  | Type   | Value          |
         |-----------------------|--------|----------------|
         | inputDataS3BucketName | String | iotareinvent18 |
         | inputDataS3Key        | String | inputdata.csv  |
         | iotchannel            | String | batchchannel   |
           
    9. Select a Role → Choose the IAM Role → search & select iotAContainerRole
    10. Configure the capacity for container :
        Compute Resource : 4 vCPUs and 16 GiB Memory
        Volume size (GB) : 2
    11. Configure the retention of your results → Keep its default and Click on Create Data set
    
    Navigate to the IoT Analytics console home page,  in the left navigation pane, choose Analyze.
    
    12. Click on Data Set → container_dataset 
    13. On the data set page, in the upper-right corner, choose Actions, and then choose Run now
    
    It can take 5-7 minutes for the data set to complete. If no errors, it will show SUCCEEDED under the name of the data set in the upper left-hand corner. 
    If it fails , check the Troubleshooting section or call Support Staff.
    

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
