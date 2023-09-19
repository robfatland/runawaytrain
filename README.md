<img src="https://github.com/robfatland/runawaytrain/blob/main/runawaytrain.png" width="500">


# runawaytrain


Purpose: An AWS Lambda that runs regularly, counts the number of EC2 instances across all regions,
and if this total exceeds a threshold: Stops all EC2 instances and locks all IAM Users out of the
account.


Once the machinery is in place as described below: Ideally we would like to deploy new copies
into sub-accounts in an automated 'infrastructure as code' manner.


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


  
## Creating the Lambda to test halting EC2 instances


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
        - Select AmazonEC2FullAccess and add it... Will this work? Yes, it worked.
- Configuration > Environment Variables > Edit
    - Add keypairs: `snstopic runawaytrain`, `ec2limit 4`, and `envaction proceed`
        - The snstopic will be used to send an email to the notification list for this account
        - The ec2limit is the maximum number of instances: Higher than this triggers the halt
        - The envaction is either `proceed` or `pass`, where the latter prevents the Lambda from running
- Modify the code in `lambda_function.py` to something like this:


```
import json, boto3, os

snstopic = os.environ['snstopic']
ec2limit = int(os.environ['ec2limit'])
envaction = os.environ['envaction']

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


### Testing across regions

The test account has access to 17 regions in the US, Canada, South America, Europe and the Asian Pacific. As a low bar I
created 10 instances in the US, 5 in Europe, 4 in Asia Pacific, and 2 in South America.
This Lambda function stopped all 21 of them in 31 seconds.
  

