



<br><br>
<img src="[https://github.com/robfatland/runawaytrain/blob/main/runawaytrain.png]" width="100">
<br><br>



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
    * No attention paid to Security Group so presumably this used a default
*  

### Set up billing alarms

* [First key page](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html#turning_on_billing_metrics)
* [Second key page](https://aws.amazon.com/blogs/mt/setting-up-an-amazon-cloudwatch-billing-alarm-to-proactively-monitor-estimated-charges)

  
## Creating the Lambda

* General information: [Lambda with an API Gateway trigger](https://docs.aws.amazon.com/lambda/latest/dg/services-apigateway.html)
* [Using a Lambda to stop EC2 instances](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2/client/stop_instances)

- Name `ec2halt` language `Python 11`

```
import json
import boto3

ec2 = boto3.resource('ec2', region_name='us-west-2')

def stop_running_instances(ec2):
    instances = ec2.instances.filter(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    for instance in instances:
        id=instance.id
        ec2.instances.filter(InstanceIds=[id]).stop()
print("All Instances are now stopped.")
    
if __name__ == "__main__":
    stop_running_instances(ec2)
```
