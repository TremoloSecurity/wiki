Drupal 7 Configuration and Integration
======================================

Drupal 7 is a powerful Content Management System (CMS) used extensively throughout the B2C, B2B and B2E markets. Drupal is more more then a website platform, its a web application development platform capable of hosting sophisticated applications. Just as with any application platform, Drupal has a myriad of ways to integrate user information and authenticate users, however the number of modules and tools can become difficult to manage. Unison can provide and end-to-end integration with Drupal including SSO, provisioning and directory services.

Externalizing identity services from Drupal offers several advantages:

1.  Simplify the your development - Identity management services such as directories, federation services and user provisioning are complex and can complicate your development model. Externalizing identity management allows you to develop against the base-line Drupal deployment your application needs while integrating an organization's complex identity management during a release cycle.
2.  Focus on content - If your customers want different authentication mechanisms for different parts of your application, externalizing Identity Management can extract that from Drupal. Focus on the content you are building, not identity management functionality.
3.  Integrate Drupal with other applications seamlessly - Deploying Drupal along side other applications? Externalizing SSO makes it much easier to create a single flow because the user doesn't need additional credentials.

There are several ways to integrate Unison and Drupal, however the easiest is to place Unison in front of Drupal as a reverse proxy with Unison connecting to Drupal's database to perform group based access decisions and to provision Drupal users and groups "just-in-time". The below diagram shows two use cases for Unison and Drupal. The first shows Unison and Drupal integrated in an environment with direct access to the enterprise directories. The second shows Unison and Drupal running in an external cloud, but still using identity management. In both instances Drupal is isolated from the identity management environment.

![Unison and Drupal with Directory Connections](https://www.tremolosecurity.com/anon/wiki/images/drupal/drupal_direct.png)
**Unison and Drupal with Directory Connections**

![Unison and Drupal in an External Cloud](https://www.tremolosecurity.com/anon/wiki/images/drupal/drupal_cloud.png)
**Unison and Drupal in an External Cloud**

In both instances, the integration process is very similar, with the only differences being how the user authenticates and where group information comes from. When running with directories, Unison can authenticate directly and extract the user's group information. When running in the cloud, Unison would rely on SAML2 to authenticate to a corporate identity provider and to determine a user's groups. Once a user is authenticated, Unison will execute a workflow to update the users table and users\_roles table When Unison and Drupal are on the same network, the above steps will generally take a negligible amount of time and will go unnoticed by the user.

The following sections detail the configuration of Unison for just-in-time provisioning into Drupal as well as the application configuration on a reverse proxy.

Drupal Configuration and Integration
--------------

This guide covers how to configure Unison to work with Drupal assuming that you have already configured directories and authentication.

Before You Begin
----------------

Before beginning this integration make sure to have:

1.  Obtained the JDBC driver and connection information for Drupal's database
2.  Created mappings of user attributes and group memberships
3.  Download the Apache LastMile module for your web server

Directory Integration
---------------------

You can directly integrate the Drupal database as a directory for Unison. This step is optional, but can provide additional options for securing Drupal using Unison. When dealing with web application authorizations there are generally 2 layers:

1.  Coarse Grained Authorization - Do you have access to this URL?
2.  Entitlements - Are you authorized to perform this action or see this link?

While Drupal does the ability to limit who has access to pages based on a role, adding an additional layer in-front of Drupal could shield it from some known vulnerabilities that other systems wouldn't be vulnerable to. Integrating Drupal's database as a directory into Unison can also make it available to other applications running in your cloud that can only talk LDAP.

### Before Configuration

#### Steps in Unison

1.  Login to Unison's management system (on port 9090 by default)
2.  Click "Manage Proxy" on the left hand side
3.  Scroll to the bottom of the page and click "Manage Proxy Libraries"
4.  Next to "Library" click the browser or choose button depending on your browser and upload the JDBC drivers for your database
5.  Click "Save"
6.  Restart Unison from the command line

#### Steps in the Drupal Database

In order to make the Drupal database easier to work with, execute the following steps. These steps have been written for MySQL but should work for any SQL database:

1.  Set the anonymous user (id of 0) name - **update users set name='Anonymous' where uid=0;**
2.  Create the unisonroles view - **create view unisonroles AS SELECT rid AS roleid, name AS rolename FROM role;**

### Creating the Directory

1.  Login to the Unison Management Portal (on port 9090 by default)
2.  Click on User Directories
3.  Under "Create Directory" choose "BasicDB"
4.  In the "Driver" drop down you should see the name of the driver for the JDBC library you uploaded. If it isn't there, make sure you have the right libraries and they've been uploaded to /usr/local/tremolo/tremolo-service/ext-lib.
5.  Use the below tables to configure the directory

#### Directory Configuration

| Option                      | Value                                                |
|-----------------------------|------------------------------------------------------|
| Name                        | Drupal                                               |
| User Directory              | Checked                                              |
| Enabled                     | Checked                                              |
| Driver                      | *Chose your driver*                                  |
| User                        | *Specify the user used to access the database*       |
| Password                    | *Specify the user's password*                        |
| Validate                    | *Re-enter the password*                              |
| Maximum Connections         | 10                                                   |
| Maximum Idle Time           | 30000                                                |
| Users Table Name            | users                                                |
| Users Table Primary Key     | uid                                                  |
| Connection Validation Query | SELECT 1 *This may change depending on the database* |
| Use Groups?                 | Checked                                              |
| Group Table Name            | unisonroles                                          |
| Group Table Primary Key     | roleid                                               |
| Link Table Name             | users\_roles                                         |
| Link Table User User Column | uid                                                  |
| Link Table Group Column     | rid                                                  |

#### User Mappings

| LDAP Name | DB Column |
|-----------|-----------|
| uid       | name      |
| mail      | mail      |

#### Group Mappings

| LDAP Name    | DB Column |
|--------------|-----------|
| cn           | rolename  |
| uniqueMember | name      |

### Post Configuration And Validation

After saving the configuration, reload the proxy by going to "Manage Proxy" and clicking "Reload Proxy Configuration" at the bottom. Now click on "Find Users" and type "uid" into the "Attribute" field under "Simple Lookup" and "admin" in the "Value" field and click "Search". The user should show under the search results. Clicking on the will bring up a screen that should their login as "uid", email address a "mail" and list all memberships in the Drupal database.

Just-In-Time Provisioning
-------------------------

The easiest way to provide identity information to Drupal is by using just-in-time provisioning. Just-in-time means Unison will update Drupal's database when the user authenticates based on some authoritative source (i.e. a set of roles or groups in a directory or attributes in a SAML assertion). When a user logs in, Drupal is automatically updated with the user's most up-to-date attributes and permissions. If a user changes roles or updates their contact information, this data is updated in the Drupal database as soon as the user logs in.

### Provisioning Targets

The first step when configuring just-in-time provisioning with Drupal is to create a provisioning target for the Drupal database. By default Drupal stores only the user's login name, email and display name.

#### Before Configuration

If you haven't already setup the JDBC drivers in Unison, follow these instructions:

1.  Login to Unison's management system (on port 9090 by default)
2.  Click "Manage Proxy" on the left hand side
3.  Scroll to the bottom of the page and click "Manage Proxy Libraries"
4.  Next to "Library" click the browser or choose button depending on your browser and upload the JDBC drivers for your database
5.  Click "Save"
6.  Restart Unison from the command line

#### Configuration Steps

1.  Login to the Unison management system (on port 9090 by default)
2.  Click on "Provisioning Targets"
3.  Click on "Add Provisioning Target"
4.  For the name specify Drupal, for the Class Name choose "Relational Database"
5.  Click "Submit"
6.  Fill in the configuration with the below data

##### Target Configuration
| Option                            | Value                                    |   
|-----------------------------------|------------------------------------------|
| Name                              | Drupal                                   |
| Class Name                        | Relational Database                      |
| Driver                            | *Choose the driver for your database*'   |
| URL                               | *Connection url for your database*       |
| Begin Escape Character (Optional} | *Blank*                                  |
| End Escape Character (Optional)   | *Blank*                                  |
| User Name                         | *User with write access to the database* |
| Password                          | *Account password*                       |
| Verify                            | *Verify the password*                    |
| Maximum Connections               | 10                                       |
| Maximum Idle Connections          | 10                                       |
| User Table                        | users                                    |
| User SQL                          |                                          |
| User Table Primary Key Field      | uid                                      |
| User Table Login Name Field       | users.name                               |
| Group Management Mode             | ManyToMany                               |
| Group Table Name                  | role                                     |
| Group SQL                         |                                          |
| Group Table Primary Key           | rid                                      |
| Group Table Name Field            | role.name                                |
| Group Link Table Name             | users\_roles                             |
| Group Link User Field             | uid                                      |
| Group Link Group Field            | rid                                      |
| Custom Provider                   |                                          |

##### Target Attribute Map

| Name     | Type   | Value             |
|----------|--------|-------------------|
| name     | user   | uid               |
| mail     | user   | mail              |
| uid      | user   | drupalid          |
| status   | static | 1                 |
| timeZone | static | America/New\_York |


### Workflows

Workflows will depend on your business needs. Assuming you are using Just-In-Time provisioning from a SAML2 assertion, use the below steps to create a basic workflow:

1.  Login to the Unison management portal (port 9090 by default)
2.  Click on "Workflows"
3.  Click "Add Workflow"
4.  Use:
    -   Name - Drupal7JIT
    -   Label - *Leave blank*
    -   Description - *Leave blank*
    -   Organization - Root

5.  Click "Submit"
6.  Click on "Edit Raw XML"
7.  Make sure the **workflow** tag has a beginning and end tag
8.  Paste the below text between the begin and end tags of the workflow

```
   <customTask className="com.tremolosecurity.provisioning.customTasks.Attribute2Groups">
       <param name="attributeName" value="roles"/>
   </customTask>
   <customTask className="com.tremolosecurity.unison.drupal.drupal7.provisioning.Drupal7GetSequence">
       <param name="targetName" value="Drupal"/>
   </customTask>
   <mapping strict="true">
       <provision sync="true" target="Drupal" setPassword="false"/>
       <resync keepExternalAttrs="true"/>
       <map>
           <mapping targetAttributeName="drupalid" targetAttributeSource="drupalid" sourceType="user"/>
           <mapping targetAttributeName="mail" targetAttributeSource="mail" sourceType="user"/>
           <mapping targetAttributeName="uid" targetAttributeSource="uid" sourceType="user"/>
       </map>
   </mapping>
```

Finally, click "Save" then "Close"

Proxy Integration
-----------------

Once the directory and provisioning steps are completed Drupal can be integrated into Unison.

### Before Configuration

#### In Drupal

1.  Download and install the correct mod\_auth\_tremolo module for your webserver
2.  Download and enable the Webserver authentication module
3.  Go to the Drupal database and create a mapping for the admin user
    -   insert into authmap (uid,authname,module) values (1,'admin','webserver\_auth');

4.  Configure Drupal for use with a reverse proxy
    -   Option 1 - Set the base\_url option in sites/default/settings.php to the url of Unison
    -   Option 2 - Enable proxy settings in Drupal (*NOTE: Unison by default uses https, if Drupal is not running https patch Drupal to support the X-Forwarded-Proto header*)

#### In Unison

Before integrating Drupal, create an authentication chain. If Drupal is running in the cloud its recommended to create a SAML2 chain. See the [TestingIdP](#!TestingIdP.md) page for how to create a connection to the TremoloSecurity.com testing identity provider.

### Configuring Unison

1.  Login to the Unison Management Portal (port 9090 by default)
2.  Click on "Applications"
3.  Click on "Add Application"
4.  Use the below data

| Option                                | Value                      |
|---------------------------------------|----------------------------|
| Name                                  | Drupal                     |
| Type                                  | User Application           |
| Session Cookie                        | tremolosession             |
| Session Cookie Secure                 | Checked                    |
| Session Inactivity Timeout (Seconds)  | 900                        |
| Session Cache Timeout in Milliseconds | 3000                       |
| Cookie Domain                         | *The host name for Unison* |
| Logout URI                            | /user/logout               |
| Session Key Alias                     | tremolosession             |

Finally, click "Submit". When the page reloads there will be heading called "URLs". Under this heading click "Add URL" and use the below table to fill in the information:

| Option                        | Value                                                                                            |
|-------------------------------|--------------------------------------------------------------------------------------------------|
| URI                           | /                                                                                                |
| Regular Expression            | Unchecked                                                                                        |
| Proxy To Application          | Checked                                                                                          |
| Proxy To                      | http://host.for.drupal.server:port${fullURI}                                                     |
| Authentication Chain          | Choose the chain created in "Before Configuration"                                             |
| Authentication Success Result |                                                                                                  |
| Authentication Failure Result | Default Login Failure                                                                            |
| Authorization Success Result  |                                                                                                  |
| Authorization Failure Result  | Default Login Failure                                                                            |

**Note:** If manually settings the base\_url in Drupal, make sure that "Override URL Host" is checked. If using Drupal proxy configuration, make sure it is NOT checked.

Click "Submit". After the page reloads there will be three new subheadings on the page. For "Hosts", add a new host that matches the host for Unison. This should lineup to the Cookie Domain in the application configuration. For "Rules", create a single rule with an LDAP Scope of "dn" and a Constraint of "o=Tremolo". For Filters, add:

1.  Decode Form Parameter Name
2.  Hide Cookies from Client
3.  Create XForward Headers *Leave the "Create Standard Headers" option unchecked*
4.  Last Mile Security *Use the below options*
    -   Encryption Key - Blank
    -   Specify New Encryption Key - drupal
    -   Time Skew - 10000
    -   Header Name - tremoloHeader
    -   Post Validation Class Name - *Blank*
    -   Verify Only - Unchecked
    -   Attribute Mapping
        -   uid --&gt; uid *Check User Identifier*
        -   X-Forwarded-For --&gt; X-Forwarded-For
        -   X-Forwarded-Host --&gt; X-Forwarded-Host
        -   X-Forwarded-Proto --&gt; X-Forwarded-Proto
    -   Create Headers - Checked
    -   Keystore Path - *Blank*
    -   Ignore URI - *Blank*
    -   Filter Type - Apache

### Post Configuration

#### In Unison

1.  Make a copy of the text in the **Location** tag from the "Servlet Configuration" of the Last Mile Security filter
2.  Reload the proxy configuration

#### In Drupal

1.  Add Last Mile Security configuration to the .htaccess file for Drupal
2.  Restart the Drupal Web Server
3.  Login to Drupal **through Unison** as admin
4.  Go the module configuration for the Webserver authentication module, under "Advanced Settings" check "Skip authorisation table check" and save

Common Issues
---------

### Style sheets don't appear


This generally occurs because base\_url is not set or the proxy is not configured properly.

### Some content is missing


If a page loads, but some of the content is missing it may be because many Drupal modules do not work well with https and assume an http connection. The browser will show a warning that it won't load unsecure content.

Tremolo Security
----------------
Useful links

* [Tremolo Security Home](https://www.tremolosecurity.com/)
* [Wiki Home](https://www.tremolosecurity.com/wiki/#!index.md)
* [Tremolo Security on Github](https://www.github.com/tremolosecurity/)
* Follow us on Twitter - [gimmick:twitterfollow](tremolosecurity) @tremolosecurity
