<img src="https://github.com/robfatland/runawaytrain/blob/main/runawaytrain.png" width="500">


# runawaytrain


Purpose: Deploy AWS infrastructure -- including a Lambda function -- that regularly checks
AWS account integrity and responds to a compromised (bad guy) state. There are two trigger 
conditions involved: Counting the number of running EC2 instances across all regions (trigger:
if the count exceeds a threshold); and looking at recent spend on AWS over the past two hours
(trigger: spend exceeds a threshold). Once we hit a trigger: Stop all EC2 instances and 
locks out all IAM Users. Only the root user can come back in and start things back up again.


## Top view notes


- Once the machinery is in place as described below: Ideally we would like to deploy new copies
into sub-accounts in an automated 'infrastructure as code' manner.


- There are some details to be aware of
    - Notification of a trigger is sent via email using **SNS**
        - The SNS *topic* is defined and recipients *subscribe* to that topic
        - The triggered Lambda then *publishes* to that topic
    - **Policies** are permission slips on AWS: They must be assigned (for example to the Lambda function) in order to take certain actions
    - **Environment variables** can be used to store sensitive information such as an AWS account ID
    - A Lambda function is passed a JSON object called **`event`** that can control the Lambda behavior
    - Debugging
        - print statements work fine
        - Helpful: Error messages when the Lambda fails
        - There are log files that contain additional diagnostic narrative
    - Regions
        - Be aware of regional locations when building infrastructure
            - For example see the `snsregion` environment variable described below 


Resources:
- [costnotify repo](https://github.com/robfatland/costnotify): **Alarm (timer)** > **Lambda** > **SNS Email**
- [digital twin repo](https://github.com/robfatland/digitaltwin): Use of the `event` object passed to the event handler 



## Premise


- Payer account **P**
    - Has associated sub-accounts **S1**, **S2**, ... , **SN**
    - Has corresponding Notify lists (people who should know something is up) **L1** etc
    - Has corresponding Fail criteria (maximum number of VMs anticipated, etcetera)
        - Example: "Number of EC2 instances will never exceed 3"
- For **P** and sub-accounts **S#**:
    - Every (say) five minutes a script runs to determine the system state: **Ok / Fail**
        - Most of the time: Nothing has happened in the past five minutes, state = **Ok**
        - Perhaps: Events result in state = **Fail**
            - Enact response
                - Halt all EC2 instances across all available regions (now working; see below)
                - Disable all IAM User access to this account
                - Disable all IAM Roles (?)
                - Send a notification email to the Notify list
         

## Notes

There are two aspects to the trigger concept: Spend per unit time and a simple total number of EC2 instances.
One could argue that one or the other is sufficient; but my premise is that having multiple checks
on account use is a good idea.


### Resources


Dashboards


* [...the Cost Intelligence dashboard (intro)](https://aws.amazon.com/blogs/aws-cloud-financial-management/a-detailed-overview-of-the-cost-intelligence-dashboard/)
* [...the Cloud Intelligence dashboard (lab)](https://wellarchitectedlabs.com/cost/200_labs/200_cloud_intelligence/)


Billing alarms


* [First key page](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html#turning_on_billing_metrics)
* [Second key page](https://aws.amazon.com/blogs/mt/setting-up-an-amazon-cloudwatch-billing-alarm-to-proactively-monitor-estimated-charges)


Lambda functions

* General information: [Lambda with an API Gateway trigger](https://docs.aws.amazon.com/lambda/latest/dg/services-apigateway.html)
* To the purpose: [Using a Lambda to stop EC2 instances](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2/client/stop_instances)


## Procedure: Billing alarms

### Establish Cost and Usage Report (CUR) logging into S3


This is on the spend side of the trigger concept. It is independent of the EC2 'how many?' process (see below).


* Set these up per the latest [AWS CUR bucket recipe page](https://docs.aws.amazon.com/cur/latest/userguide/cur-create.html)
* I named the S3 bucket for the data `organization-aws-cur`
    * This bucket is organized in a heirarchical folder structure; distal leaves are gzipped CSV CUR files
    * In this example we get to the file dated 06-SEP-2023, 03:27 Zulu
        * `organization-aws-cur/report/organization_aws_cur/20230901-20231001/20230906T032712Z/organization_aws_cur-00001.csv.gz


### Set up billing alarms

     
## Procedure: EC2 instance profusion 


### Start some EC2 instances


For testing purposes a keypair `.pem` file is not necessary. 


* Launch instances in at least 3 regions
    * qty=3 (per region) x t2.micro, Amazon Linux, no keypair, use `default` Security Group


The threshold (max number of EC2 instances) will be 4. In this way no single region triggers the halt.


## Procedure: Creating an SNS email mechanism

- Find the SNS service and create a new topic 'runawaytrain'
- Type = Standard, Display Name = Runaway Train; and Save it
- Go to Subscriptions > Create Subscription
    - topic ARN: Use drop-down to select the runawaytrain ARN
    - Protocol: Email
    - Recipient: `my_email@uw.edu`
    - Create Subscription
    - Now in my email Inbox: Confirm the subscription
    - Repeat for all desired emails from the notification list for this account


To wire the Lambda function up to the SNS subscription: See the `costnotify` code.
Requires introducing another environment variable, the 12-digit account number.


## Procedure: Creating the Lambda to test halting EC2 instances


The Lambda function counts EC2 instances across all possible regions. 
If this count exceeds the environment variable `ec2limit` the Lambda stops all
instances. Since Lambda is intended to run on a timer, its action can be overridden 
in two ways: Using the `event` JSON object that is passed to the Lambda event handler
when it runs; and by means of an environment variable associated with the Lambda
function.


- `event` JSON object: key `action`, value `pass` (halt) or `proceed`
- environment variable `envaction`, value `pass` (halt) or `proceed`


- Oregon, create function `ec2halt` from scratch in language `Python 3.11`
    - Notice that a unique Role is created and assigned to this Lambda function: **`ec2halt-role-w3p78jgn`**
- This Lambda will crash and burn without an attached Policy that permits EC2 manipulation
    - Dashboard > Configuration > Permissions > `ec2halt-role-w3p78jgn` > hyperlink to IAM page
        - Add Permissions > Attach Policies
        - Select AmazonEC2FullAccess and add it
        - Select AmazonSNSFullAccess and add it
    - For the Lambda role there should be 3 attached policies: Lambda basic execution, EC2 full access and SNS full access
- Configuration > Environment Variables > Edit
    - Add keypairs: See table below for keys and values
        - The snstopic will be used to send an email to the notification list for this account
        - The ec2limit is the maximum number of instances: Higher than this triggers the halt
        - `envaction` is `proceed` or `pass`, the latter preventing Lambda from running
        - `accountnumber` is the 12-digit account identifier, used for the SNS topic ARN
- Modify the code in `lambda_function.py` to something like this:


Here are the environment variables used by the Lambda function:


```
snstopic        runawaytrain
ec2limit        4
envaction       proceed
accountnumber   123456789012
snsregion       us-west-2
```

Here is the Lambda Python code:

```
import json, boto3, os

snstopic      = os.environ['snstopic']
ec2limit      = int(os.environ['ec2limit'])
envaction     = os.environ['envaction']
accountnumber = os.environ['accountnumber']
snsregion     = os.environ['snsregion']

def stop_running_instances(region_name):
    ec2 = boto3.resource('ec2', region_name=region_name)        # a collection of instances in this region
    instances = ec2.instances.filter(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    idlist = [instance.id for instance in instances]
    print('in region ' + region_name + ' we found ' + str(len(idlist)) + ' EC2 instances')
    for id in idlist:
        print('halting: ' + id)
        ec2.instances.filter(InstanceIds=[id]).stop()
    return len(idlist)

def FreezeIAM():
    client = boto3.client('iam')
    iam = boto3.resource('iam')
    access_key = iam.AccessKey('user_name','id')
    print("Client:")
    print(client)
    print("IAM:")
    print(iam)
    print("Access Key:")
    print(access_key)
    return
    
def SendSNSMsg(msg):
    email_subject = 'Runaway Train notification email'
    email_body    = 'You subscribed.\n' + msg
    sns           = boto3.client('sns')
    arnstring     = 'arn:aws:sns:' + snsregion + ':' + accountnumber + ':' + 'runawaytrain'
    response      = sns.publish(TopicArn=arnstring, Message=email_body, Subject=email_subject)
    return
    
def lambda_handler(event, conext):
    '''This function runs on the Lambda trigger'''
    
    # Process a possible JSON event override: 'action' == 'pass'
    if 'action' in event.keys():
        print(event)
        if event['action'] == 'pass':
            print('ec2halt never minded by action == pass')
            return { 'statusCode': 200, 'body': 'action == pass voids lambda ec2halt execution' }
        elif event['action'] == 'proceed': print("event action signals proceed normally")
        else: print("unrecognized event action value")
        
    # Process a possible environment variable override: 'envaction' == 'pass'
    if envaction == 'pass':
        print('ec2halt never minded by envaction == pass override')
        return { 'statusCode': 200, 'body': 'envaction == pass voids lambda ec2halt execution' }
        
    # proceed normally    
    
    print("counting EC2 instances; must excede " + str(ec2limit) + " to trigger halt")
    nEC2 = []       # a list of how many EC2 instances were stopped, by region
    # WARNING This region list is specific to an AWS account. Caveat emptor: Modify as appropriate.
    regions = ['us-east-1', 'us-east-2', 'us-west-1', 'us-west-2',                                                     \
               'ap-south-1', 'ap-northeast-2', 'ap-northeast-3', 'ap-southeast-1', 'ap-southeast-2', 'ap-northeast-1', \
               'ca-central-1',                                                                                         \
               'eu-central-1', 'eu-west-1', 'eu-west-2', 'eu-west-3', 'eu-north-1',                                    \
               'sa-east-1']
    ec2counter = 0

    # count up instances across all regions
    for region in regions:
        ec2 = boto3.resource('ec2', region_name=region)     # a collection
        instances = ec2.instances.filter(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
        ec2counter += len([instance for instance in instances])
        if ec2counter > ec2limit:
            print("cumulative sum " + str(ec2counter) + " (limit " + str(ec2limit) + ") triggers halt")
            break         # once the ec2limit is exceded: Stop counting

    # if halt condition is True: stop all instances in all regions
    if ec2counter > ec2limit:
        for region in regions: nEC2.append(stop_running_instances(region))
        ackbody = 'ec2halt ack: ' + str(sum(nEC2))

    else: ackbody = "ec2halt found instance count <= limit"

    FreezeIAM()
    SendSNSMsg("Here is the test message")
            
    return { 'statusCode': 200, 'body': ackbody }
```


> NOTE: Objects returned by filter() are type `boto3.resources.collection.ec2.instancesCollection`.
> Conversion to list form makes these easy to count via `len()`. `.id` is the instance identifier,
> a unique string.


- Click the <Deploy> button to register the new code with the Lambda function
- Configure a Test run (use defaults and give it a simple name)
    - The test button uses a JSON structure passed to the Lambda as variable 'event' 
        - This will be 'proceed' by default; but 'pass' will cause the Lambda to not run    
        - Provide (via Edit) a keypair "action": "pass" (the do-not-run state)
        - Modify "pass" to "proceed" to run normally
        - This information is processed at the top of the event handler

```
{
  "action": "pass",
  "key2": "value2",
  "key3": "value3"
}
```
- Modify the timeout interval
    - Configuration > General Configuration > Edit > Set to 5 min 0 sec
    - My early tests suggest 1 second is needed to stop each instance
         - regions x (instances / region) gives roughly the time needed to complete (seconds)
         - reconsider this 5 minute timeout: That is only enough to stop 300 instances
- Click the <Test> button and verify that the EC2 instances created above are stopped


### Testing across regions and debugging


The Lambda Python code generates red ink when something is amiss. Upon fixing it one must 
remember to click the **Deploy** button to actually refresh the code. 


The test account has access to 17 regions in the US, Canada, South America, Europe and the Asian Pacific. 
I created two or more EC2 instances in eight of these regions to test the Lambda halt function.
I verified that both the `envaction` and `action` paths disable the Lambda when given value `pass`.


Stopping N instances takes on the order of N seconds. 


> Diagnostics can be found in the Monitor tab, Logs sub-tab. 
  

