<img src="https://github.com/robfatland/runawaytrain/blob/main/runawaytrain.png" width="500">


# runawaytrain


Purpose: Deploy some AWS machinery -- including a Lambda function -- to regularly check
AWS account integrity and respond to a compromised (bad guy intruder) state. Bad guys
have been known to spin up compute resources that burn thousands of dollars per hour, 
a deplorable turn of events that we refer to as a *runaway train*. 



There are two conditions that can trigger the shutdown: The number of running EC2 instances
in a Member account exceeds a given threshold; or recent spend in the Member Account exceeds 
a second threshold. Under either condition we proceed to stop all EC2 instances and discontinue 
User access via console, AWS Role and/or API.


The runaway train build applies to a single Payer (Management/Security) Account plus some
Sub-accounts aka Member Accounts. 


## Notes from 03-OCT-2023

- Terminology: The Payer Account is also referred to as the Management/Security (M/S) Account
- Terminology: The Sub-account is also referred to as a Member Account
- Terminology: CloudFormation stack is a text file that is 'run' to set up infrastructure in an account
    - We flag relevant infrastructure in these notes with a bolded **step**
    - These are things that need to happen around managing the account; code-able in a CF stack
    - *"Create a CloudFormation template to change, track, and redeploy without manual intervention."*
        - In addition to Billing Alerts the CF template can put HIPAA constraints in place
- Account Access
    - AWS IAM Identity Center has three options:
        - Identity Center Directory: Self-contained, lots of labor involved.
        - Active Directory: 'without using SAML federation'...not an option here
        - External Identity Provider: Federate with SAML or other external IDPs including Azure AD.
            - As UW uses Azure AD this is the way to go; passwords are per NetID.
            - New accounts **step**: Create IAM Roles for Users to assume on login
            - Requires coordination with UW IT (ticket submitted)
- Our AWS colleagues to report back on remarks from a Mgmt & Governance team specialist
    - Specifically: **One Lambda, many CloudTrail Billing Alerts** concept
- Error on creating trigger: Failed to fetch: Implies Lambda and Alert must both run in N.Va
    - Rebuild the Lambda in N.Va
- Important links to have on hand
    - [IAM Identity Center](https://aws.amazon.com/iam/identity-center/) for managing SSO
        - [UW ITConnect: Federated logon](https://itconnect.uw.edu/tools-services-support/access-authentication/sso/integrate-vendor-products/)
        - [UW ITConnect parent of the above](https://itconnect.uw.edu/tools-services-support/access-authentication/sso/)
        - [AWS view of SSO](https:/docs.aws.amazon.com/singlesignon)
    - boto3
	- [detaching a Role Policy](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/iam/client/detach_role_policy.html)
	- [deleting a Role Policy](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/iam/client/delete_role_policy.html)
	- [IAM top level page](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/iam.html)
    - AWS Organizations
        - [AWS Organizations cross-account Role documentation](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_access.html#orgs_manage_accounts_access-cross-account-role)
        - [AWS Organizations Account access management](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_access.html)

    - [HIPAA](https://aws.amazon.com/solutions/implementations/compliance-hipaa/)
        - [White paper](https://docs.aws.amazon.com/pdfs/whitepapers/latest/architecting-hipaa-security-and-compliance-on-aws/architecting-hipaa-security-and-compliance-on-aws.pdf)
        - [HIPAA-eligible services](https://aws.amazon.com/compliance/hipaa-eligible-services-reference/)
        - New accounts **step** when needed: Follow this blueprint
    - New accounts **step** create a billing alert
    - AWS Organizations
        - Important **step**: Create organizational units (OUs) in AWS Organizations
            - Used to group researcher’s Accounts and apply Policies by OU
    - Runaway Train feedback / discussion from the AWS SA
        - Icing Users, Instances, Access Keys and Roles/Policies will not 'lose work': Confirmed
            - Exception: Someone hard-codes an AK into a resource: Very poor practice anyway
        - Add to initial concept: Ice IAM Roles
            - When you federate you do not have individual IAM Users
            - Consequently the Lambda must delete Roles that federated users assume.
            - **Do not** wholesale delete all IAM Roles in the account as many are used by AWS services
            - Delete only Roles that Users are assuming
            - 'If any Federated User can assume a Role: You must delete that Role to stop the train'
            - [Boto3 reference on listing Roles / Tags](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/iam/client/list_role_tags.html)
        - A Lambda can assume a Role with permission to take action in sub-accounts
            - This is the **One Lambda** concept (in the Management/Security (M/S) account)
            - Provided the management account is secure:
                - An intruder in a member account has no way to hijack this M/S Lambda
        - Ensure member accounts have a Role available to the Lambda
            - **step** In Organizations: There is a default built-in Administrator Role for Member Accounts
            - Look at Control Tower: The Primary Payer Account has this Admin Role built in vis Member Accounts 
            - Create Member Accounts using Organizations
                - This automatically sets up the key *OrganizationAccountAccessRole* in the Member Account
        - Billing Alert: Generated under CloudWatch in the N.Virginia region
            - This runs over the consolidated bill in the Payer (M/S) Account
            - From the Payer account: It is assumed at this point there is no Member Account breakout or filter
                - This in turn means that Billing Alerts must be established on each Member Account (**step**)


> Key Concept: The Runaway Train solution consists of two entities: The Lambda Function on the M/S account and independently
> one Billing Alert in each Member Account. The Lambda can stop the train of its own volition based on an EC2 count
> exceeding a threshold. The Billing Alert can trigger the Lambda to stop the train by means of an SNS Topic.


- A single Lambda runs in the Payer account with Role-based access to sub-accounts
    - This assumes Roles in sequence to check on EC2 count
    - In contrast each sub-account has a CloudWatch spend alert installed
        - When this reads high spend it notifies an SNS topic
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


### A Python Lambda using the `boto3` AWS interface library

- This code imports the `boto3` library (SDK) to execute tasks on the AWS cloud
- One of the key lines of this code is `client = boto3.client('iam')`
    - This creates a `client` object used to deactivate Users and Access Keys
    - We could also write this code in terms of a `resource` rather than a `client`
        - [On the distinction (from StackOverflow)](https://stackoverflow.com/questions/42809096/difference-in-boto3-between-resource-client-and-session):

#### **Client** and **Resource** are distinct abstractions.

- They can both be used to make AWS service requests.
- **Client** is the original abstraction and it works as a low-level interface
    - Often in this case we work with Python dictionaries that include lists
- **Resource** is a newer and higher-level abstraction; so more `method()` focus


#### Examples of `.client` versus `.resource` code

```
# boto3 .client S3 listing (subject to limit 1000; cf paginator)
import boto3
client = boto3.client('s3')
response = client.list_objects_v2(Bucket='mybucket')
for content in response['Contents']:
    obj_dict = client.get_object(Bucket='mybucket', Key=content['Key'])
    print(content['Key'], obj_dict['LastModified'])
```

Example `.resource` code

```
# boto3 .resource S3 listing (no 1000-restriction; pagination automatic)
import boto3
s3 = boto3.resource('s3')
bucket = s3.Bucket('mybucket')
for obj in bucket.objects.all():
    print(obj.key, obj.last_modified)
```


Two reference repositories on GitHUb (Lambda code in practice):


- [costnotify repo](https://github.com/robfatland/costnotify): **Alarm (timer)** > **Lambda** > **SNS Email**
- [digital twin repo](https://github.com/robfatland/digitaltwin): Use of the `event` object passed to the event handler 



## Runaway train logic overview


- Payer account **P**
    - Associated sub-accounts **S1**, **S2**, ... , **SN**
    - Corresponding SNS Topics **SNS1**, ...
        - Corresponding subscribers; a 'Notification List' **L1**, ...
    - Threshold for EC2: **TE1**, ...
    - Threshold spend: **TS1**, ...
        - Examples: "EC2 instance count <= 3", "Spend per six hours < $800"
- On **P**: Every five minutes a CloudWatch alarm triggers Lambda
    - For all **S**: Count EC2 in all regions
        - If sum > **TE**: Halt the account and notify **L**
            - Halt: Stop all EC2 instances, disable User Roles, delete Access Keys
- On each **S**: A Billing Alert triggers an SNS Topic configured to trigger Lambda
    - Ideally this is the same Lambda but with specific event arguments
        - Response as above
         

### Two types of trigger rationale

Spend per unit time and total number of EC2 instances: Two independent ways to catch 
the problem and mitigate the damage.


### Resources


- Dashboards
    - [Cost Intelligence dashboard (intro)](https://aws.amazon.com/blogs/aws-cloud-financial-management/a-detailed-overview-of-the-cost-intelligence-dashboard/)
    - [Cloud Intelligence dashboard (lab)](https://wellarchitectedlabs.com/cost/200_labs/200_cloud_intelligence/)


- Billing alarms
    - [Key page](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html#turning_on_billing_metrics)
    - [Another key page](https://aws.amazon.com/blogs/mt/setting-up-an-amazon-cloudwatch-billing-alarm-to-proactively-monitor-estimated-charges)
- Lambda functions
    - [General: Lambda with API Gateway trigger](https://docs.aws.amazon.com/lambda/latest/dg/services-apigateway.html)
    - [Using a Lambda to stop EC2 instances](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2/client/stop_instances)



## Billing Alerts


### Establish Cost and Usage Report (CUR) logging into S3


This is on the spend side of the trigger concept. It is independent of the EC2 'how many?' process (see below).


* Set these up per the latest [AWS CUR bucket recipe page](https://docs.aws.amazon.com/cur/latest/userguide/cur-create.html)
* I named the S3 bucket for the data `organization-aws-cur`
    * This bucket is organized in a heirarchical folder structure; distal leaves are gzipped CSV CUR files
    * In this example we get to the file dated 06-SEP-2023, 03:27 Zulu
        * `organization-aws-cur/report/organization_aws_cur/20230901-20231001/20230906T032712Z/organization_aws_cur-00001.csv.gz


### Set up Billing Alerts

     
## EC2 instance profusion > threshold 


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
recoveryp       dsp#*%712POCxxs!)*632MKW
```

Lambda Python code (under development; do not use):

```
import json, boto3, os

snstopic      = os.environ['snstopic']
ec2limit      = int(os.environ['ec2limit'])
envaction     = os.environ['envaction']
accountnumber = os.environ['accountnumber']
snsregion     = os.environ['snsregion']
recoveryp     = os.environ['recoveryp']

def stop_running_instances(region_name):
    ec2 = boto3.resource('ec2', region_name=region_name)        # using .resource(): The collection of instances: this region
    instances = ec2.instances.filter(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    idlist = [instance.id for instance in instances]
    print('in region ' + region_name + ' we found ' + str(len(idlist)) + ' EC2 instances')
    for id in idlist:
        print('halting: ' + id)
        ec2.instances.filter(InstanceIds=[id]).stop()
    return len(idlist)

def IAMLockout():
    client = boto3.client('iam')            # Can print this; .client() is the original SDK interface; .resource is higher-level
    Ulist = client.list_users()['Users']
    nUsersBounced      = 0
    nAccessKeysDeleted = 0
    
    for user in Ulist:
        uname = user['UserName']
        if not uname == 'kjanet' and not uname == 'nykamp':          # for testing: Don't alter these two accounts

            akeys = client.list_access_keys(UserName=uname)          # str(len(akeys['AccessKeyMetadata']))
            for key in akeys['AccessKeyMetadata']:                   # key is a dictionary taken from a list of dictionaries
                key_uname = key['UserName']
                key_id    = key['AccessKeyId']
                client.delete_access_key(AccessKeyId=key_id, UserName=key_uname)
                print(key_id, key_uname)
                nAccessKeysDeleted += 1
                
            rp = recoveryp + uname
            # client.delete_login_profile(UserName=uname)         # this code is failing; check existence first
                                                                  # [ERROR] NoSuchEntityException
                                                                  # Note distinction .delete_login_profile() vs .delete_user() ...?...
            print(uname + ' login profile not deleted pending fix')
            
            # A note on restoring access: Do not do this. An inimical account would be restored as well; unless
            #   there is some sort of white list; but then that would be documented here so it would be vulnerable.
            #   client.create_login_profile(UserName=name, Password=rp, PasswordResetRequired=True)
            nUsersBounced += 1
    
    print('Bounced ' + str(nUsersBounced) + ' users and deleted ' + str(nAccessKeysDeleted) + ' access keys')
    return
    
def SendSNSMsg(msg):
    email_subject = 'Runaway Train notification email'
    email_body    = 'You subscribed.\n' + '\n' + msg
    sns           = boto3.client('sns')
    arnstring     = 'arn:aws:sns:' + snsregion + ':' + accountnumber + ':' + 'runawaytrain'
    response      = sns.publish(TopicArn=arnstring, Message=email_body, Subject=email_subject)
    return
    
def lambda_handler(event, conext):
    '''This is the function that runs when this Lambda is triggered'''
    
    # Two ways to get this Lambda to stop execution
    #   - pass an event argument 'action' with value 'pass'
    #   - set an environment variable 'envaction' with value 'pass'
    if 'action' in event.keys():
        print(event)
        if event['action'] == 'pass':
            print('ec2halt void because event action == pass')
            return { 'statusCode': 200, 'body': 'action == pass voids lambda ec2halt execution' }
        
    # Process a possible environment variable override: 'envaction' == 'pass'
    if envaction == 'pass':
        print('ec2halt void because envaction == pass')
        return { 'statusCode': 200, 'body': 'envaction == pass voids lambda ec2halt execution' }
    
    print("counting EC2 instances; must excede " + str(ec2limit) + " to trigger halt state")
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

    # should report on other actions as well:
    SendSNSMsg("ec2halt test: shut down " + str(sum(nEC2)) + " EC2 instances")
    
    IAMLockout()   # goes inside halt code
    
    return {'statusCode': 200, 'body': ackbody }
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
  

