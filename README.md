<img src="https://github.com/robfatland/runawaytrain/blob/main/runawaytrain.png" width="500">


# runawaytrain
Purpose: Stop EC2 instances under trigger circumstances. Bonus: Lock out all IAM Users and prevent any new instances from spawning.
As a resource I refer to the [costnotify repo](https://github.com/robfatland/costnotify). It illustrates 
**Alarm** > **Lambda** > **SNS Email** (and it runs daily on a cron timer).

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

There are two aspects to the trigger concept: Spend per unit time and bulk number of EC2 instances.
One could argue that one or the other is sufficient; but my premise is that having multiple checks
on account usage is a good idea.


To be aware of...
* [...the Cost Intelligence dashboard (intro)](https://aws.amazon.com/blogs/aws-cloud-financial-management/a-detailed-overview-of-the-cost-intelligence-dashboard/)
* [...the Cloud Intelligence dashboard (lab)](https://wellarchitectedlabs.com/cost/200_labs/200_cloud_intelligence/)


## Procedure: Billing alarms

### Establish Cost and Usage Report (CUR) logging into S3

This is on the spend side of the trigger concept. It is independent of the EC2 'how many?' process (see below).

* Set these up per the latest [AWS CUR bucket recipe page](https://docs.aws.amazon.com/cur/latest/userguide/cur-create.html)
* I named the S3 bucket for the data `organization-aws-cur`
    * This bucket is organized in a heirarchical folder structure; distal leaves are gzipped CSV CUR files
    * In this example we get to the file dated 06-SEP-2023, 03:27 Zulu
        * `organization-aws-cur/report/organization_aws_cur/20230901-20231001/20230906T032712Z/organization_aws_cur-00001.csv.gz


### Set up billing alarms

* [First key page](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html#turning_on_billing_metrics)
* [Second key page](https://aws.amazon.com/blogs/mt/setting-up-an-amazon-cloudwatch-billing-alarm-to-proactively-monitor-estimated-charges)
     
## Procedure: EC2 instance profusion 

### Start some EC2 instances

* Launch instances
    * qty=10 x t2.micro, Amazon Linux, Region = Oregon, generated `train.pem`, allow ssh traffic from 0.0.0.0\0
    * Select the `default` Security Group to avoid SG clutter 


  
## Creating the Lambda to test halting EC2 instances


To start out I will just use the Lambda <Test> button.


* General information: [Lambda with an API Gateway trigger](https://docs.aws.amazon.com/lambda/latest/dg/services-apigateway.html)
* To the purpose: [Using a Lambda to stop EC2 instances](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2/client/stop_instances)


- Oregon, create function `ec2halt` from scratch in language `Python 3.11`
    - Notice that a unique Role is created and assigned to this Lambda function: **`ec2halt-role-w3p78jgn`**
- This Lambda will crash and burn without an attached Policy that permits EC2 manipulation
    - Dashboard > Configuration > Permissions > `ec2halt-role-w3p78jgn` > hyperlink to IAM page
        - Add Permissions > Attach Policies
        - Select AmazonEC2FullAccess and add it... Will this work? Yes, it worked.
- Configuration > Environment Variables > Edit
    - Add keypairs: `snstopic runawaytrain` and `ec2limit 4`
        - The snstopic will be used to send an email to the notification list for this account
        - The ec2limit is the maximum number of instances: Higher than this triggers the halt
- Modify the code in `lambda_function.py` to something like this:


```
import json, boto3, os

snstopic = os.environ['snstopic']
ec2limit = int(os.environ['ec2limit'])

def stop_running_instances(region_name):
    ec2 = boto3.resource('ec2', region_name=region_name)        # a collection of instances in this region
    instances = ec2.instances.filter(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    idlist = [instance.id for instance in instances]
    print('in region ' + region_name + ' we found ' + str(len(idlist)) + ' EC2 instances')
    for id in idlist:
        print('halting: ' + id)
        ec2.instances.filter(InstanceIds=[id]).stop()
    return len(idlist)
    
def lambda_handler(event, conext):
    '''This function runs on the Lambda trigger'''
    print("counting EC2 instances; must excede " + str(ec2limit) + " to trigger halt"
    nEC2 = []       # list of how many EC2 stopped, by region, as follows:
    # WARNING This region list is specific to an AWS account. Caveat emptor: Modify as appropriate.
    regions = ['us-east-1', 'us-east-2', 'us-west-1', 'us-west-2',                                                     \
               'ap-south-1', 'ap-northeast-2', 'ap-northeast-3', 'ap-southeast-1', 'ap-southeast-2', 'ap-northeast-1', \
               'ca-central-1',                                                                                         \
               'eu-central-1', 'eu-west-1', 'eu-west-2', 'eu-west-3', 'eu-north-1',                                    \
               'sa-east-1']
    ec2counter = 0

    # count up the instances found in all possible regions
    for region in regions:
        ec2 = boto3.resource('ec2', region_name=region)     # a collection
        instances = ec2.instances.filter(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
        ec2counter += len([instance for instance in instances])
        if ec2counter > ec2limit: break         # once the ec2limit is exceded: Stop counting

    # if halt condition is True: stop all instances in all regions
    if ec2counter > ec2limit:
        for region in regions: nEC2.append(stop_running_instances(region))
        ackbody = 'ec2halt ack: ' + str(sum(nEC2))
    else:
        ackbody = "ec2halt instance count is <= limit"

    return { 'statusCode': 200, 'body': ackbody }
```

> NOTE: The objects returned by filter() are of type `boto3.resources.collection.ec2.instancesCollection` (with no len() available).
The instance id is the useful information; so the code creates a list of ids from the filtered ec2 collection.


- Click the <Deploy> button to register the new code with the Lambda function
- Configure a Test run (use defaults and give it a simple name)
- Modify the timeout interval
    - Configuration > General Configuration > Edit > Set to 5 min 0 sec
    - My early tests suggest 1 second is needed to stop each instance
         - regions x (instances / region) gives roughly the time needed to complete (seconds)
         - reconsider this 5 minute timeout: That is only enough to stop 300 instances
- Click the <Test> button and verify that the EC2 instances created above are stopped


### Testing across regions

The test account has access to 17 regions in the US, Canada, South America, Europe and the Asian Pacific. As a low bar I
created 10 instances in the US, 5 in Europe, 4 in Asia Pacific, and 2 in South America.
This Lambda function stopped all 21 of them in 31 seconds.
  

