# runawaytrain
...stopping EC2 instances under trigger circumstances...


This repo builds on the [costnotify repo](https://github.com/robfatland/costnotify) which illustrates
important elements of **Alarm** to **Lambda** to **Notify via SNS**.

## Premise

- Payer account **P**
    - Has associated sub-accounts **S1**, **S2**, ... , **SN**
    - Has corresponding Notify lists
    - Has corresponding Fail criteria
        - Example "Number of EC2 instances exceeds 1"
- Every five minutes a script runs to determine Success / Fail
    - For Fail enact response
        - Halt all EC2
        - Disable all IAM Users: Kicked off
        - Disable all Roles
        - Notify persons on the Notify list

