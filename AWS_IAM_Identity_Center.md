# AWS IAM Identity Center


## Links


- [UW IT Resource](https://wiki.cac.washington.edu/display/infra/Sign+in+to+the+AWS+Console+with+UW+NetID)
- [AWS Resource](https://us-east-1.console.aws.amazon.com/singlesignon/home?region=us-east-1#!/)
- SSO Userguide on...
    - [...connecting to Microsoft AD as IDP](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-ad.html)
    - [...IDP management in general](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-idp.html)
    - [...Organizations](https://docs.aws.amazon.com/singlesignon/latest/userguide/organization-instances-identity-center.html)
    - [...User Access](https://docs.aws.amazon.com/singlesignon/latest/userguide/useraccess.html)


## Narrative: Condensed version

* Can the UW IdP for NetID SSO to the AWS console be repurposed for AWS Organizations?
* What are the implications for using API-based access (`cli`, `boto3` etcetera) 
* SSO is 'single sign on': I log on to an AWS Console Account using a NetID
* SAML is Security Assertion Markup Language
* IdP is Identity Provider, in this context UW IT
* A User performs actions using a Client browser


## User Experience: Longer narrative, more detail


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

- 


## UW Wiki: Configuration procedural 


Pasting and editing from the wiki: The information is for a single account, not an 
AWS Organization. 


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



