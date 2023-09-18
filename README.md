<img src="https://github.com/robfatland/runawaytrain/blob/main/runawaytrain.png" width="500">


# runawaytrain
Purpose: Stop EC2 instances under trigger circumstances.
Resource: The [costnotify repo](https://github.com/robfatland/costnotify) illustrating **Alarm** > **Lambda** > **SNS Email** (runs daily).

## Premise

- Payer account **P**
    - Has associated sub-accounts **S1**, **S2**, ... , **SN**
    - Has corresponding Notify lists
    - Has corresponding Fail criteria
        - Example "Number of EC2 instances exceeds 1"
- Every five minutes a script runs to determine Success / Fail
    - Ideal circumstances: Nothing has happened in the past five minutes
    - Bad: Some series of events results in evaluation Fail
        - For Fail enact response
            - Halt all EC2
            - Disable all IAM Users: Kicked off
            - Disable all Roles
            - Notify persons on the Notify list
         

## Notes

Be aware of...
* [...the Cost Intelligence dashboard (intro)](https://aws.amazon.com/blogs/aws-cloud-financial-management/a-detailed-overview-of-the-cost-intelligence-dashboard/)
* [...the Cloud Intelligence dashboard (lab)](https://wellarchitectedlabs.com/cost/200_labs/200_cloud_intelligence/)


## Procedure


### Establish Cost and Usage Report (CUR) logging into S3

* Set these up per the latest [AWS CUR bucket recipe page](https://docs.aws.amazon.com/cur/latest/userguide/cur-create.html)
* I named the S3 bucket for the data `organization-aws-cur`
    * This bucket is organized in a heirarchical folder structure; distal leaves are gzipped CSV CUR files
    * In this example we get to the file dated 06-SEP-2023, 03:27 Zulu
        * `organization-aws-cur/report/organization_aws_cur/20230901-20231001/20230906T032712Z/organization_aws_cur-00001.csv.gz
     

### Start some EC2 instances

* Launch instances
    * qty=3 x t2.micro, Amazon Linux, Region = Oregon, generated `train.pem`, `chmod 400 train.pem`, allow ssh traffic from 0.0.0.0\0
    * Select the `default` Security Group to avoid SG clutter 

### Set up billing alarms

* [First key page](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html#turning_on_billing_metrics)
* [Second key page](https://aws.amazon.com/blogs/mt/setting-up-an-amazon-cloudwatch-billing-alarm-to-proactively-monitor-estimated-charges)

  
## Creating the Lambda

* General information: [Lambda with an API Gateway trigger](https://docs.aws.amazon.com/lambda/latest/dg/services-apigateway.html)
* To the purpose: [Using a Lambda to stop EC2 instances](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2/client/stop_instances)

- Oregon, create function `ec2halt` from scratch in language `Python 3.11`
- This Lambda will crash and burn without the necessary Role/Policy needed to work with EC2 instances
    - Dashboard > Configuration > Permissions > `ec2halt-role-w3p78jgn` > hyperlink to IAM page
        - Add Permissions > Attach Policies
        - Select AmazonEC2FullAccess and add it... Will this work? Yes, it worked.

- Modify the code in `lambda_function.py` to something like this:


```
import json
import boto3

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
    nEC2 = []       # list of how many EC2 stopped, by region, as follows:
    regions = ['us-east-1', 'us-east-2', 'us-west-1', 'us-west-2',                                                     \
               'ap-south-1', 'ap-northeast-2', 'ap-northeast-3', 'ap-southeast-1', 'ap-southeast-2', 'ap-northeast-1', \
               'ca-central-1',                                                                                         \
               'eu-central-1', 'eu-west-1', 'eu-west-2', 'eu-west-3', 'eu-north-1',                                    \
               'sa-east-1']
    for region in regions: nEC2.append(stop_running_instances(region))
    print("Stopped!")
    ackbody = 'ec2halt ack: ' + str(nEC2)
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

  

