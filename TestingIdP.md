Introduction
------------

The testing identity provider is a service provided by Tremolo Security to help test Unison, or really any other SAML2 Service Provider or Relying Party. The testing IdP provides a mechanism to generate an assertion without having to stand up your own identity provider. This makes testing much easier as you don't need to worry about integrating with directories, generating certificates or configuring another system.

![](https://www.tremolosecurity.com/anon/wiki/images/testidp/testidp-intro.png)

The testing IdP supports IdP initiated SSO with the following features:

-   Sign Responses
-   Sign Assertions
-   Set the RelayState
-   Set the NameID Format
-   Set the NameID
-   Set the AuthnContextClassRef
-   Add attributes

This tutorial will take you through the process of integrating the testing IdP with Unison. Though the testing IdP should work with other products, its not officially supported.

First Steps
-----------

Prior to starting this tutorial:

1.  Register for an account on TremoloSecurity.com
2.  Download or deploy Unison
3.  Follow the instructions in Chapter 2 "Where Do I Start?" of the Unison Administration Manual

Creating The Identity Provider
------------------------------

The first step is to login to ![](https://www.tremolosecurity.com/support/). On the left hand side there will be an option called "Test SAML2 SP/RP", click this option.

![](https://www.tremolosecurity.com/anon/wiki/images/testidp/idp1.png)

Next right-click on the link for "Identity Provider Metadata" and select "Save As...". This will save the metadata from import into Unison.

![](https://www.tremolosecurity.com/anon/wiki/images/testidp/idp2.png)

At this point all certificates have been generated and the metadata is now available for import into Unison. The next step is to create a SAML2 authentication chain in your Unison server.

Create the SAML2 Authentication Chain
-------------------------------------

Login to the Unison Management Portal (on port 9090) and click on "Authentication Chains"

![](https://www.tremolosecurity.com/anon/wiki/images/testidp/idp3.png)

Next click on "Add Authentication Chain" and fill in the information for the new chain

![](https://www.tremolosecurity.com/anon/wiki/images/testidp/idp4.png)

After clicking "Submit" a new link at the bottom of the page will appear called "Add Authentication Mechanism", click on that link. When the new form comes up, choose "SAML2" from the list box next to "Name" and click "Submit"

![](https://www.tremolosecurity.com/anon/wiki/images/testidp/idp5.png)

Once the SAML2 configuration is loaded, click on "Using Meta Data" under "Configure Using", open the idp metadata from the previous section in a text editor and copy and paste the content into the large text box that appears.

![](https://www.tremolosecurity.com/anon/wiki/images/testidp/idp6.png)

Under "Authentication Information" for "Required Authentication Type" choose "Password over SSL". Under "Directory Mapping Information specify the below information:

| Option                     |     |  Value             |
|----------------------|-----|---------------|
| LDAP Name Attribute  | =   | uid           |
| DN Org Unit Name     | =   | SAML2         |
| Default Object Class | =   | inetOrgPerson |

Click "Save". Once the form is reloaded, under "From MetaData" there is a check box called "Require Signed Response", click that box

![](https://www.tremolosecurity.com/anon/wiki/images/testidp/idp7.png)

Click "Save" again. Finally, at the bottom of the screen where it says "Generate Service Provider MetaData" specify the host name unison is running on and click "Generate"

![](https://www.tremolosecurity.com/anon/wiki/images/testidp/idp8.png)

The new window that pops up will be used to add Unison to the testing IdP, save it locally.

Enable the SAML2 Authentication Chain
-------------------------------------

The last step in Unison is to enable the SAML2 chain we just created. Go to the TestLogin application created in Chapter 2 of the Unison Administration Manual. Next click "Edit" next to the "/login" URL. Change the "Authentication Chain" setting from "Default Form" to "TestIdP" and click "Submit". Finally, reload the proxy configuration.

Add Relying Party
-----------------

Log back into the Unison support site and go back to the testing IdP. At the bottom of the screen where it says "Add New Relying Party" copy and paste the contents of the metadata generated from Unison in the previous step and click "Create Relying Party".

![](https://www.tremolosecurity.com/anon/wiki/images/testidp/idp9.png)

Test Unison
-----------

On the testing IdP page, click "Launch" next to the relying party that was just created.

![](https://www.tremolosecurity.com/anon/wiki/images/testidp/idp10.png)

The launch screen is where you can start the SSO process. The minimum information you need to specify is the "Relay State" and the "Name ID". The Relay State tells Unison what your final destination will be while the Name ID will tell Unison what user you are. For Relay State, specify the full URL for the TestLogin application that was just updated. For instance, if the host for Unison is ec-df4tg.cloud.com then the value should be "![](https://ec-df4tg.cloud.com/login)". Specify "testuser" (without quotes) as the Name ID. This will link the user with the test user created in the testing application.

In addition to the username, additional information can be added to the assertion. For instance personalization information or entitlements. For this test, add an attribute called "MyAttribute" with the value "MyValue" and click "Add Attribute". The attribute will now be listed.

![](https://www.tremolosecurity.com/anon/wiki/images/testidp/idp11.png)

Finally, at the top of the screen click "Launch!" The final screen should be a new window that shows:

![](https://www.tremolosecurity.com/anon/wiki/images/testidp/idp12.png)

Note that the username is "testuser" and the attribute created in the previous step is present.

What's Next?
------------

You can experiment with this tool to use different users and add additional attributes. When integrating with an application, you can use this tool to simulate how your identity provider might work without having to set one up on your own!

Tremolo Security
----------------
Useful links

* [Tremolo Security Home](https://www.tremolosecurity.com/)
* [Wiki Home](https://www.tremolosecurity.com/wiki/#!index.md)
* [Tremolo Security on Github](https://www.github.com/tremolosecurity/)
* Follow us on Twitter - [gimmick:twitterfollow](tremolosecurity) @tremolosecurity
