Introduction
------------

AWS lets you externalize your identities so that users don't need to be created inside of AWS. This offers several advantages over creating accounts inside of AWS:

1.  Apply your own password policies
2.  Easily control access to resources
3.  Immediately de-provision users when they leave your organization

Unison integrates with the console as a SAML2 identity provider:

![Architecture](https://www.tremolosecurity.com/docs/tremolosecurity-docs/images/awswiki/idaas_aws.png)

Unison provides a great platform for AWS console for a few reasons:

1.  Unison provides multiple types of authentication, including easy integration with several multi-factor options
2.  If you don't have an external multi-factor solution, Unison can provide either TOTP or One-Time-Password over SMS
3.  Unison + Scale provides a method for users to request access, approvers to approve access and reporting for access
4.  Unison provides a simple way to map from role names to the role names that AWS expects

Before beginning, stand up an IDaaS using Unison such as this : [Aws](Aws "wikilink")

Data Model
----------

AWS expect the assertion to have two attributes. The first, "<https://aws.amazon.com/SAML/Attributes/RoleSessionName>" is the user identifier. The second attribute, "<https://aws.amazon.com/SAML/Attributes/RoleSessionName>" lists out the possible roles a user has been assigned. This attribute must have a very specific format consisting of the account number, role name and identity provider name. Instead of having to create this role attribute manually, Unison provides an HttpFilter that will generate it for you based on your identity provider, account number and the role name.

The ability to easily define the roles attribute means that you can have a data model that looks like:

| IDaaS Group Name | AWS Role Name | SAML Role Attribute                                                                   |
|------------------|---------------|---------------------------------------------------------------------------------------|
| aws-SQSAdmins    | SQSAdmins     | arn:aws:iam::1234567890:role/SQSAdmins,arn:aws:iam::1234567890:saml-provider/AWS-Demo |

Note that the name of the group maps to the name of the role. This makes adding new roles and workflows very easy.

AWS
---

To setup in AWS:

1.  Setup an identity provider
2.  Create roles
3.  Download the AWS metadata

Here are the instructions from Amazon: <http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_saml.html>

Setting up Miltifactor Authentication
-------------------------------------

If you already have some method of multi-factor enabled, it can be integrated into an authentication chain. If you want to utilize Unison's TOTP implementation:

1.  Add a column to the users table called 'totp'
2.  Add the "Time Based One Time Password" mechanism in Authentication Mechanisms - <https://www.tremolosecurity.com/docs/tremolosecurity-docs/1.0.6/administration/webhelp/content/ch11s15s01.html>
3.  Add the TOTP mechanism to an authentication chain (or create a new one) - <https://www.tremolosecurity.com/docs/tremolosecurity-docs/1.0.6/administration/webhelp/content/ch11s15s02.html>

Once TOTP is configured, create a workflow that will provision the totp token:

1.  Add a provisioning target identical to the authorizations database target, but with only one attribute : totp
2.  Create a workflow with the "Create OTP Task"
3.  Expose the workflow to Scale so users can request it
4.  Deploy Scale TOTP - <https://www.tremolosecurity.com/docs/tremolosecurity-docs/1.0.6/scale/scale-manual-1.0.6.html#_scale_totp_token_retrieval>

At this point users can login, request a TOTP token and import it into their phone.

Setting up the Unison Identity Provider
---------------------------------------

Finally, Unison can be configured with an identity provider to SSO into the AWS Console:

1.  Run the Identity Provider wizard with the metadata downloaded from AWS
2.  If following the same group mappings as above, add the "Create attribute from group memberships" with a pattern of aws-(.\*) and group number 1 from the regular expression to the identity provider
3.  Add the "Create AWS Role Attribute" filter to the identity provider, using the attribute created by the "Create attribute from group memberships" filter
4.  Add a mapping from uid to <https://aws.amazon.com/SAML/Attributes/RoleSessionName>
5.  Finally, add the authentication chain and nameid mapping to the trust

AWS only supports IdP Initiated SSO. The easiest thing would be to create a link on the Scale portal to the identity provider

Tremolo Security
----------------
Useful links

* [Tremolo Security Home](https://www.tremolosecurity.com/)
* [Wiki Home](https://www.tremolosecurity.com/wiki/#!index.md)
* [Tremolo Security on Github](https://www.github.com/tremolosecurity/)
* Follow us on Twitter - [gimmick:twitterfollow](tremolosecurity) @tremolosecurity
