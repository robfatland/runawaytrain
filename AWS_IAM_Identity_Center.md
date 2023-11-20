# AWS IAM Identity Center


## Links


- [UW IT Resource](https://wiki.cac.washington.edu/display/infra/Sign+in+to+the+AWS+Console+with+UW+NetID)
- [AWS Resource](https://us-east-1.console.aws.amazon.com/singlesignon/home?region=us-east-1#!/)
- SSO Userguide on...
    - [...connecting to Microsoft AD as IDP](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-ad.html)
    - [...IDP management in general](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-idp.html)
    - [...Organizations](https://docs.aws.amazon.com/singlesignon/latest/userguide/organization-instances-identity-center.html)
    - [...User Access](https://docs.aws.amazon.com/singlesignon/latest/userguide/useraccess.html)


## Basics

* Key questions
    * How do we enable AWS **IAM Identity Center**? Answer: It is already enabled in Region = us-west-2 (Oregon)
    * 'Can the UW IdP for NetID SSO to the AWS console be repurposed for AWS Organizations?'
    * 'What are the implications for using API-based access (`cli`, `boto3` etcetera)'
* Term: SSO = 'single sign on': I log on to an AWS Console Account using a NetID
* Term: SAML = Security Assertion Markup Language
* Term: IDP = Identity Provider, in this context UW IT using Azure AD
* In what follows: A User performs actions using a Client browser


## IAM Identity Center configuration



### Naming the IAM Identity Center instance


The instance name displays when application owners configure applications with IAM Identity Center, and to users in the AWS access portal. 
The name can be changed at a later time: `escience-aws`. The IIC has a name `ssoins-01...`. It has an organization ID `o-abcd...`. It has
an ARN `arn:aws:sso:::instance/ssoins-01...`. 


### Identity source

- Choose to make this an external provider
    - We manage users and groups in an external identity provider (IdP)
    - Users sign in to your IdP sign-in page, are redirected to the AWS access portal
    - After they sign in to the AWS access portal, they can access their assigned AWS accounts and cloud applications

### Authentication


### Management 


### Tags



## SSO transaction in modest detail


* A User connecting to the AWS console is directed to the IdP
* The IdP authenticates the User who receives a SAML assertion
    * Let's think of this as a badge held by the Client browser
    * It can carry a lot of information
    * It can stipulate a time frame / expiration for this authentication
* The Client browser is directed to the AWS Single Sign-on (SSO) endpoint
    * There it posts the SAML assertion
* The endpoint requests temporary security credentials on behalf of the User
* The endpoint creates a console sign-in URL that uses those credentials
* AWS sends the sign-in URL back to the client as a redirect
* The Client browser is redirected to the AWS Management Console
    * The SAML authentication response may include attributes that map to multiple IAM roles
        * In this case the User is prompted to select the Role for accessing the console
        * Note this allows for User overloading to multiple AWS identities
            * Contrast an AWS IAM root identity mapping 1:1 to AWS accounts
         

## AWS SSO Userguide notes


> We work from a 'self-managed' AD at UW; so as we do not need it: Disregard the **AWS Managed Microsoft AD**.


- AWS **IAM Identity Center**: Connect a self-managed directory in Active Directory (AD) using the **AWS Directory Service**
    - This AD directory defines the pool of identities administrators can pull from
        - ...use the **IAM Identity Center** console to assign single sign-on access
    - After connecting your corporate directory to **IAM Identity Center**
        - Grant AD users or groups access to AWS accounts and/or applications
- To use **AWS Directory Service** to connect AWS resources with the *existing* self-managed AD
    - In configuration: Set up a trust relationship
- **IAM Identity Center** uses the connection provided by **AWS Directory Service**
    - Performs pass-through authentication to the source AD instance
- Prerequisite: Ensure the AD Connector resides within your AWS Organizations management account


## UW Wiki: Configuration procedural 


Paste/edit from the wiki: This is a six-step procedure for a single AWS account 
presumed to be under the DLT Payer Account. It does not cover AWS Organizations. 


The AWS account owner configures federated sign-in:


- Send UW-IT the 12-digit-number AWS Account ID: To `iam-support@uw.edu`
    - UW-IT creates a UW group stem for you to manage AWS roles
        - `u_weblogin_aws_(accountid)`
- Add the UW IdP as an Identity Provider
    - AWS Console > IAM > Identity Providers > Create Provider
        - Type: `SAML`
        - Provider Name: `UW`
        - Metadata Document: Attach the IdP's metadata as a file
        - > `Create`
- Create an AWS Role for SAML login
    - > IAM > Roles > Create New Role
        - `SAML 2.0 federation`
        - SAML provider: Select from above
        - Attribute: `SAML:aud`: `https://signin.aws.amazon.com/saml`
    - `Next: Permissions`
        - Choose AWS Policies granted this Role
    - > `Review`
        - Role Name
            - lowercase letters, digits, dashes, dots only
            - must match the corresponding UW group ID (see below)
        - > Create Role
- Create a UW group to grant the above AWS role
    - UW groups service: `https://groups.uw.edu`
    - New group under the group stem created above
        - Last part of the Group ID (after final `_`) must match the AWS Role Name
            - `u_weblogin_aws_(accountid)_(rolename)`
    - Add UW NetIDs: Adds people who can sign in and assume the Role as members of this new group
        - Users must be added as members of the group (direct or effective)
        - Adding a user as `admin` (or another non-member role) will void SSO
- Sign in to the AWS Console
    - Amazon uses IdP-initiated SSO
        - This is also known as "unsolicited" SSO...
            - ...as it is not initiated by the service provider
    - A static link is used to initiate sign-in from the UW IdP
        - [Link: Sign in to the AWS Console](https://idp.u.washington.edu/idp/profile/SAML2/Unsolicited/SSO?providerId=urn:amazon:webservices)
        - Alternative: `https://idp.u.washington.edu/idp/profile/SAML2/Unsolicited/SSO?providerId=urn:amazon:webservices`
        - This initiates sign-in from the UW IdP to Amazon.
    - The maximum session duration defaults to one hour
        - Edit the AWS Role above to modify duration
        - After Session ends: Probably kicked off; do it again
- Note: 2FA is enabled for all users by default.
    - To learn more about 2FA options and eligibility, refer to two-factor authentication in IT Connect.


UW IT remarks: With AWS Accounts (not Organizations), we have used SAML with UW Groups enabling users to go to one URL 
and then be presented with whatever AWS account and roles that User has available. 



