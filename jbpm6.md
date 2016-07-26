Introduction and Overview
-------------------------

This page is a step-by-step guide on how you can deploy the JBoss jBPM system on a public cloud while continuing to provide security and authentication using your corporate standards.

### How Do You Run jBPM in the Cloud?

There are several aspects to this question, but the main one tackled in this white paper is from a security stand point. Deploying jBPM is pretty straight forward. Deploy JBoss, deploy a database, deploy jBPM. What about users?

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/jbpm_unknown.png)

Like most organizations yours has multiple Active Directory forests. Additionally, the person who owns the jBPM deployment from a business perspective doesn't own the Active Directory deployments so they're not able to simply agree to any infrastructure changes needed to make an integration work. Finally, you know there's going to be an audit to make sure the system is secure.

#### Local Users

The first option is to use JBoss' built in authentication system using properties files:

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/jbpm_localusers.png)

This solves the problem of having to connect to Active Directory, but wouldn't likely pass a security audit and is also painful to manage. This could get you up and running quickly but could quickly eliminate the benefits of running on a public cloud server with expensive user management costs.

#### Connect Directly to Active Directory

Another option is to connect directly to Active Directory from JBoss via a VPC:

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/jbpm_vpc_ldap.png)

A direct connection would help with user management costs, but introduces an inbound connection from an un-trusted cloud network to Active Directory. Since the owner of your jBPM deployment is unlikely to be the same person that owns Active Directory there may be an up-hill battle getting approval. There is also a potential integration issue as Active Directory doesn't perform exactly like other LDAP directories and has very specific rules. As an example, how will jBPM manage groups across forests? Finally, what will be the performance impact of jBPM communicating back into AD?

#### Use Unison

Tremolo Security's Unison helps answer many of the above issues while not requiring a direct connection into Active Directory:

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/jbpm_and_unison.png)

Unison sits between users and jBPM providing several services:

1.  Authentication via SAML2 Federation
2.  Just-In-Time Provisioning of user data
3.  Secure Integration into JBoss

This approach provides the security that the auditors require, does not require direct inbound connections to Active Directory and provides an abstraction layer for security to enable you to focus on the value jBPM brings rather then how to build a secure system.

### How the Pieces Fit

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/jbpm_unison_pieces.png)

The above diagram depicts the pieces to making this integration work. The below table gives an overview of these pieces:

| Component                 | Description                                                                                                                                |
|---------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| SAML2 Identity Provider   | Authenticates users, provides user and role information to the relying party                                                               |
| Unison                    | Accepts assertions from the identity provider, provisions users into the database and provides access to jBPM                              |
| Secure Last Mile          | Unison's proprietary method for providing identity information securely without "phoning home" to a policy decision point or policy server |
| Just-In-Time Provisioning | Updates a database of user roles and personalization information that can be used by jBPM tasks                                            |

The configuration for each component will be detailed below.

### How Does jBPM Do Identity?

jBPM is a J2EE application that relies on the container (generally JBoss) to perform authentication and provide role information. There are two components of jBPM that require identity information:

1.  jBPM Console - The console needs to know who you are and what is authorized based on your group/role memberships
2.  Human Tasks - A human task can be assigned to either a user or a group/role

Both the console and the human tasks rely on the standard JAAS interfaces for retrieving this information. There is no hook for user lookups inside of jBPM. So a human task that is assigned to the "HR" group will be available to anyone logging in with that role but the task doesn't know to send a notification out to all users in the group "HR".

To fill this gap, this tutorial includes the build out of a database of user profile information and group memberships. This database can be used like any other database from inside of tasks.

Before We Get Started
---------------------

### Assumptions

#### jBPM

This white paper assumes that we are working with the jBPM 6.0.1 demo installation which includes JBoss 7.1.1 and will be using a MySQL RDS server on Amazon Web Services. Its assumed that prior to stating this white paper, these components are running and jBPM and the dashboard are installed and running.

#### Unison

It is assumed that Unison has already been installed and is running. For a tutorial on how to get Unison running on AWS, click "HERE".

#### Identity Provider

Unison has been certified with several readily available identity providers, listed "HERE". For the sake of simplicity, this white paper will assume integration with the SAML2 Playground on tremolosecurity.com.

### Parts List

Prior to integrating Unison and jBPM you'll need the following:

| Part                                                                | Description                                                                                             | Notes                                                        |
|---------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| MySQL JDBC Drivers                                                  | Needed to connect Unison to MySQL                                                                       | ![Picture](http://dev.mysql.com/downloads/connector/j/)                |
| MySQL Database for storing user data and administrative credentials | This database will be used to store data about users so jBPM tasks don't need to query Active Directory |                                                              |
| Host name and port of the jBPM console and dashboard                | Needed for creating a connection to jBPM                                                                |                                                              |
| Identity Provider metadata                                          | To create the connection with the identity provider                                                     | ![Picture](https://www.tremolosecurity.com/anon/metadata/saml2idp.xml) |

Integration Steps
-----------------

### The Identity Provider

The first step is to setup your identity provider, in this case the SAML2 Playground identity provider on tremolosecurity.com. To setup this provider, first follow the instructions "HERE".

The assertion generated by the identity provider will have 3 pieces of information:

1.  User Login Name - Subject NameID
2.  User's Display Name - displayName
3.  User's Email - mail
4.  User's Roles - roles

Once the identity provider is setup, download the metadata.

### Creating the User Database

Its common for a task to need to know some information about users. This may including needing to know who the members are of a group, a user's email address or display name. While a task can request this information from the user in a task form, it would be much easier if the information could be loaded from a database. While jBPM could reach back across the network to Active Directory to get this data that would defeat the purpose of separating the environments.

To that end Unison will update and maintain a user database. This database will be updated "Just-In-Time" as users login and user jBPM. Part of the parts list was a database that could be used for this purpose. To prep this database, run the following SQL:

sql
CREATE TABLE users (id int PRIMARY KEY AUTO_INCREMENT, login varchar(255), mail varchar(255),displayName varchar(255));
CREATE TABLE groups (id int PRIMARY KEY AUTO_INCREMENT, name varchar(255));
CREATE TABLE userGroups (int int PRIMARY_KEY AUTO_INCREMENT,userid int,groupid int);


The groups themselves don't need to be added to the groups table, as Unison will create them dynamically based on the data received in the assertions during authentication. This provides a tremendous amount of power and flexibility as now changes made in the central repositories in AD (such as new permissions) are immediately reflected in jBPM.

### Upload MySQL JDBC Drivers

Unison does not come with the MySQL JDBC drivers. They'll need to be uploaded using the following steps:

#### Go to Manage Proxy

From the main screen, click on the "Manage Proxy" link on the left hand side:

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/jdbc1.png)

#### Manage Proxy Libraries

At the bottom of the screen is a link to "Manage Proxy Libraries", click on that link

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/jdbc2.png)

#### Upload MySQL JDBC Driver

Click on "Browse" at the bottom of the screen, choose the MySQL JDBC driver (will be a jar file) and click "Save"

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/jdbc3.png)

#### Restart Unison

Login to Unison via SSH as tremoloadmin and restart:

```bash
$ /usr/local/tremolo/tremolo-service/bin/tremolo.sh restart
```

### Running the Application Integration Wizard

#### Login to the Unison Management Portal

Once Unison is restarted, login to the management portal

#### Start the Application Integration Wizard

Launch the application integration wizard by first logging into the Unison Management Portal and clicking on the "Application" button under "Setup Wizards"

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard1.png)

#### Screen 1 - Application Setup Wizard

This screen is a welcome screen and does not require any input, simply click "Next" in the lower right hand corner.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard2.png)

#### Screen 2 - Application Basic Information

On this screen specify how users will access jBPM and and where jBPM is located. The "Enterprise Facing" information is what users will type into their browsers where as "Application Facing" is where jBPM is currently hosted.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard3.png)

#### Screen 3 - Create User Directory

Many applications require an external directory to work properly. Since Unison will be keeping an up to date user database it could be useful to have that data mapped into Unison for directory operations. For instance if you wanted to limit who had access to the dashboard based on groups or wanted to add additional permissions based on a workflow. On this screen click "Create New Directory" so it is checked and click "Next"

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard4.png)

#### Screen 4 - Directory Information

Next specify the type of directory to configure. Unison has an integrated virtual directory which allows for nearly any data source to be used for directory data. In this instance, a MySQL database will be used. The first step is to choose BasicDB from the "Source" box shown below:

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard5.png)

Once the database configuration is loaded, configure using the below information:

| Option                            |  Value                                   |
|-----------------------------|-------------------------------------|
| Name in Tremolo             | jBPMUsers                           |
| Driver                      | com.mysql.jdbc.Driver               |
| URL                         | jdbc:mysql://host:port/databasename |
| User                        | *Account Username*                  |
| Password                    |                                     |
| Validate                    |                                     |
| Maximum Connections         | 10                                  |
| Maximum Idle Time           | 30000                               |
| Users Table Name            | users                               |
| Users Table Primary Key     | id                                  |
| Connection Validation Query | SELECT 1                            |
| Use Groups?                 | *checked*                           |
| Group Table Name            | groups                              |
| Group Table Primary Key     | id                                  |
| Link Table Name             | userGroups                          |
| Link Table User Column      | userId                              |
| Link Table Group Column     | groupId                             |

The JDBC connection information was included in the "Before You Begin" section of this document. Before clicking "Next", add user mapping information at the bottom of the screen by clicking on the "Add User Mapping" and "Add Group Mapping" links:

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard6.png)

##### User Mapping

| LDAP Name  | DB Column  |
|-------------|-------------|
| uid         | login       |
| mail        | mail        |
| displayName | displayName |

##### Group Mapping

| LDAP Name   | DB Column |
|--------------|------------|
| cn           | name       |
| uniqueMember | login      |

#### Screen 5 - Validating the Database Configuration

Once the configuration is complete Unison will attempt to connect to the directory. If there are any issues they'll be reported on this screen. Once the configuration is working, click on "Next".

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard7.png)

#### Screen 6 - Authentication Configuration

This is when we tell Unison how to authenticate users. Since this system is running in a cloud we will use SAML2. First choose "New Chain" as the Authentication Type:

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard8.png)

Then choose "SAML2" as the Authentication Mechanism:

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard9.png)

Once the SAML2 configuration screen has loaded, click on "Using Meta Data" so we can quickly configure the system:

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard10.png)

Now copy and paste the meta data from the Tremolo Security SAML2 Playground configuraiton and fill in the rest of the configuration with:

| Option                            | Value                                                  |
|------------------------------------|---------------------------------------------------------|
| Required Authentication Type       | Password over SSL                                       |
| Other Authentication Type          |                                                         |
| NameID Format                      | ![Picture](urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified) |
| Force response to SSL              | *Unchecked*                                             |
| Require Signed MetaData            | *Unchecked*                                             |
| Require Encrypted Assertion        | *Unchecked*                                             |
| Assertion Decryption Key           |                                                         |
| Sign Authentication Requests       | *Unchecked*                                             |
| Authentication Request Signing Key |                                                         |
| Optional Jump Page URI             |                                                         |
| LDAP Name Attribute                | uid                                                     |
| DN Org Unit Name                   | saml2                                                   |
| Default Object Class               | inetOrgPerson                                           |

Finally, click "Next" in the lower right hand corner.

#### Screen 7 - Enable Just-In-Time Provisioning

Just-In-Time Provisioning will be used to update the user database as users login to jBPM. This happens by executing a workflow when the user logs in, taking their attributes from the SAML2 assertion and writing them to the database, including the user's role information. Check the "Use Just-In-Time Provisioning?" checkbox and click "Next"

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard11.png)

#### Screen 8 - Configure Provisioning Target

Prior to defining the mapping from the assertion to the database a database provisioning target must be configured. This will use much of the same information as from the directory configuration. Even though in this instance the directory and the provisioning database are working off the same data it is not always the case. For instance it wouldn't be unusual for the directory to have centralized permissions but an application to have its own datastore.

The first step is to create a new store by first selecting "New Provisioning Target" next to "Existing Provisioning Target"

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard12.png)

Once the target screen refreshes, select the "Relational Database" option from "New Target Type"

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard13.png)

After the screen refreshes showing the configuration options, configure using the below table:

| Option                           | Value                                |
|-----------------------------------|---------------------------------------|
| Name                              | jBPM-Database                         |
| Driver                            | com.mysql.jdbc.Driver                 |
| URL                               | *jdbc:mysql://host:port/databasename* |
| Begin Escape Character (Optional) | \                                    |
| End Escape Character (Optional)   | \                                    |
| User Name                         | *User name for JDBC Access*           |
| Password                          | *User Password*                       |
| Validate                          |                                       |
| Maximum Connections               | 10                                    |
| Maximum Idle Connections          | 10                                    |
| User Table                        | users                                 |
| User SQL                          |                                       |
| User Table Primary Key Field      | id                                    |
| User Table Login Name Field       | login                                 |
| Group Management Mode             | ManyToMany                            |
| Group Table Name                  | groups                                |
| Group SQL                         |                                       |
| Group Table Primary Key           | id                                    |
| Group Table Name Field            | name                                  |
| Group Link Table Name             | userGroups                            |
| Group Link User Field             | userId                                |
| Group Link Group Field            | groupId                               |
| Custom Provider                   |                                       |

Finally click "Next" in the lower right hand corner.

#### Screen 9 - Test Target Configuration

After configuring the target Unison will attempt to verify the connection. Once verification is complete click "Next".

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard14.png)

#### Screen 10 - Map Attributes

The final step in the provisioning screens is to map the attributes from the assertion into the database.

##### Attribute Mappings

To map attributes from the assertion to the users table in the database click on "Add Attribute". The "Provisioned To" field is the name of the column in the users table, "From Authentication" is the source data. Depending on the source type this could be a static value, an attribute from the assertion or a composite of multiple attributes. Configure the attributes according to the below table:

| Provisioned To | Source Type | From Authentication |
|-----------------|--------------|----------------------|
| login           | user         | uid                  |
| mail            | user         | mail                 |
| displayName     | user         | displayName          |

##### Group Mappings

Unison will update the groups table based on information from the assertion. There are two options for this:

1.  Manually specify an attribute name and value to map to a group in the database
2.  Simply set the groups based on the value of a single role

The first option is useful when you have to create groups based on different attributes or have to map from one set of groups to another. The second option is better when you can control the assertion and specify all groups in a single role.

This example will use the second model. Click on the "Map all values of one attribute to groups" checkbox under Group Mappings and specify "roles" for " Attribute source from authentication for all group names : "

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard15.png)

Finally click "Next"

#### Screen 11 - Last Mile

The final configuration screen is to secure the connection between Unison and JBoss. This can be done in several ways, but Unison's preferred method is to use Tremolo Security's Last Mile Security system. This system uses an encrypted header to send identity information securely to an application without having to "phone home". In this case we'll be telling JBoss who the user is and what roles they have. This will also make sure that if someone tries to simply go straight to JBoss, they can't use jBPM.

The default selection for "Method" is "Secure Last Mile", leave that option. Check "Set User Groups to Role Attribute?:" and specify "roles" for "Role Attribute Name:"

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/wizard16.png)

Finally click "Finish".

#### Screen 12 - Summary Page

This final page provides some additional information for next steps. Clock "Close Wizard" to return to the Management Portal home page.

### Testing Federation

Once the wizard has completed, we could start the integration steps with jBPM. Before we do that, its recommended to first test federation and jit provisioning. This way if there are issues with the integration with jBPM you won't be wasting time trying to figure out if the issue is in the authentication layer, the provisioning layer or the Last Mile layer.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/unison_layers.png)

Setting up the system to test federation is very easy:

1.  Add the Login Test filter to the jBPM application
2.  Add the Stop Processing filter to the jBPM application
3.  Reload the proxy

The Login Test filter will generate a simple HTML table showing all of the user's attributes, headers and other request and session information. This will tell you that the user is properly authenticated and working with the relational database. The Stop Processing filter will tell Unison not to forward this request to the reverse proxy, stopping the request. Finally, we'll reload the proxy configuration and begin testing.

#### Add Login Test Filter

First login to the management portal and click on "Applications" on the left hand side.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/testfed1.png)

Next click "Edit" next to the application you created when running the application integration wizard.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/testfed2.png)

At the bottom of the screen, under URLs find the URI "/" and click "Edit"

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/testfed3.png)

Under Filters click "Add Filter"

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/testfed4.png)

For the "Class Name" choose "Login Test" and click "Submit"

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/testfed5.png)

When the screen refreshes specify a "Logout URI" of "/logout" and click "Submit". Finally, click on "Return to URL Configuration" to return to the overall URL configuration.

#### Add Stop Processing Filter

Click on "Add Filter" under the "Filters" section and select "Stop Processing" from the drop down and click "Submit"

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/testfed6.png)

#### Reload the Proxy

Once everything is configured, the configuration needs to be loaded into Unison. The first step is to click on the "Manage Proxy" link on the left hand menu.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/testfed7.png)

Once the configuration screen has load, click on "Reload Proxy Configuration" near the bottom of the screen.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/testfed8.png)

#### Testing

While you can use any identity provider that supports SAML2, for the sake of simplicity this tutorial doesn't provide instructions for setting up the identity provider. The Tremolo Security support site at ![Picture](https://www.tremolosecurity.com/support/) provides a testing identity provider that can easily be used to simulate an enterprise login. The wiki page TestIdP has instructions on how to use it.

### Integrating JBPM

Once federation and just-in-time provisioning has been validated the next step is to integrate the jBPM application into Unison. From a high level the steps will be:

1.  Remote the testing filters
2.  Make sure we have connectivity between Unison and jBPM
3.  Update the Last Mile configuration to generate JBoss configuration
4.  Deploy the Unison login module for JBoss 7.x
5.  Update the jBPM console and dashboard war files
6.  Deploy war files & restart JBoss

The details for each step is in this section

#### Remove Login Test Filters

Prior to connecting Unison and jBPM we should remove the filters that we added to test federation and provisioning. First, login to the management portal and click on the "Applications" link on the left hand side.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/testfed1.png)

Next click on the jBPM application.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/testfed2.png)

Then click on the "/" URL.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/testfed3.png)

Under "Filters" next to "com.tremolosecurity.prelude.filters.LoginTest" click "Delete"

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/integrate1.png)

Click "Yes" to confirm the deletion of this filter.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/integrate2.png)

Repeat this same procedure with the "com.tremolosecurity.prelude.filters.StopProcessing" filter and reload the proxy.

#### Test Accessing jBPM

Once the proxy is reloaded, you should make sure that Unison is able to communicate with jBPM. Given that this white paper assumes that you are using the Tremolo Security SAML2 Playground as an identity provider, trigger an IdP initiated login the same way you did to test federation. You should see the jBPM console login screen. If you get an error, check Unison's logs and make sure:

1.  There are no firewalls closing access between Unison and jBPM
2.  JBoss is listening on all ports, not just 127.0.0.1 (which it will do be default)
3.  JBoss is running

#### Update Last Mile Configuration

Once connectivity is confirmed, its time to start telling jBPM who the logged in user is. With JBoss 7.x this is done using a JBoss login module and a tomcat valve. The first step is to collect the information needed to create the trust between Unison and JBoss.

First, login to the management portal and click on the "Applications" link on the left hand side.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/testfed1.png)

Next click on the jBPM application.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/testfed2.png)

Then click on the "/" URL.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/testfed3.png)

Under "Filters" next to " com.tremolosecurity.proxy.filters.LastMile" click "Edit"

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/integrate3.png)

The application integration wizard pre-configured almost everything on this filter. It created a key, created attribute mappings and set the various other settings. The one component it doesn't set is the "Filter Type". This option tells Unison what system you are integrating with and will generate the appropriate configuration. In this instance choose "JBoss 7.1/JBoss EAP 6.2" and click "Submit" (near the top of the page).

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/integrate4.png)

When the page refreshes, under "Servlet Configuration" there will now be a jboss-web configuration. Copy and paste this XML to a safe place. Next right click on "Download Keystore for this URL" and save the keystore locally.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/integrate5.png)

Finally, reload the proxy configuration. From this point forward, all steps will be performed on JBoss.

#### Deploy Unison JBoss Module

From this point forward, all work will be performed on the JBoss server. There are two steps to enabling Last Mile integration with JBoss:

1.  Deploy the Unison login module
2.  Configure the applications to use Last Mile for logins

**NOTE:** Before performing any configuration steps, make sure your JVM is configured to use unlimited encryption (if you are using OpenJDK this is done automatically).

The first step is to upload the JBoss 7.x Last Mile system to your jBPM server. Once unzipped, copy the content of the "modules" directory to JBoss. On the demo thats */path/to/jbpm-installer/jboss-as-7.1.1.Final/modules/*. (Once copied, there should **NOT** be a "modules" directory inside of the JBoss modules directory).

Next, edit the JBoss configuration file */path/to/jbpm-installer/jboss-as-7.1.1.Final/standalone/configuration/standalone-full.xml* and find the `<security-domains>` section. Inside of the `<security-domains>` section add the following XML:

```
<security-domain name="unisonsecuritydomain" cache-type="default">
    <authentication>
        <login-module  code="com.tremolosecurity.lastmile.jboss71.loginModule.UnisonLoginModule" flag="required"/>
    </authentication>
</security-domain>
```

The next step is to configure the two war files.

#### Configure jBPM and Dashboards

JBoss web applications' security can't be configured without unpacking the jars. With the demo running, move the */path/to/jbpm-installer/jboss-as-7.1.1.Final/standalone/deployments/jbpm-console.war* and */path/to/jbpm-installer/jboss-as-7.1.1.Final/standalone/deployments/dashboard-builder.war* to a temporary directory. This will undeploy the war files. At this point shut down the jBPM demo.

First explode the jbpm-console.war file:

$ unzip jbpm-console.war

Next, delete the original war file (make sure to keep a backup somewhere safe)

$ rm jbpm-console.war

Edit the *WEB-INF/jboss-web.xml* file. Remove `<security-domain>other</security-domain>` and add the `<security-domain>` and `<valve>` section for the Last Mile security configuration. Before saving the configuration, make sure to specify the *pathToKeyStore* parameter in the file to point to the lastmile.jks file uploaded earlier on.

Next edit the *WEB-INF/jboss-deployment-structure.xml* file, add

```
<module name="com.tremolosecurity.lastmile.jboss71" />
```
in the <dependencies> section.

Finally, repackage the war

```
$ zip -r jbpm-console.war
```

and copy it to */path/to/jbpm-installer/jboss-as-7.1.1.Final/standalone/deployments/jbpm-console.war*.

Repeat these same steps with dashboard-builder.war and start the demo.

### Testing jBPM

#### Checking Last Mile is Running

Once JBoss is restarted (if you see errors such as "Address Already in Use" restart the demo to clear them) first test to make sure that the Unison Last Mile is enabled. The easiest way to do this is to try going directly to JBoss, bypassing Unison. If your server is running on jbpm.mycompany.com open a browser and navigate to *![Picture](http://jbpm.mycompany.com:8080/jbpm-console/)*. You will see the below error message saying that access is denied.

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/integration6.png)

This is important as it shows that a user can't come sideways and jBPM and try to spoof access. All access must come through Unison.

#### Logging into jBPM

The final step is to try logging into jBPM. Login using the Tremolo Security SAML2 Playground and you'll be welcomed with the main page. You'll know you are logged in because the user's login name will be in the upper right hand corner of the screen. Clicking on your username will show all the roles you are logged in as. Finally, try logging out by selecting "Log Out".

![Picture](https://www.tremolosecurity.com/anon/wiki/images/jbpm/integrate7.png)

If you adjust the roles for a given user, you'll see they change after logging out and logging back in!

Next Steps
----------

Now that jBPM and Unison are integrated, whats next? The possibilities are nearly endless. From a deployment perspective, you'll likely want to integrate jBPM with a real identity provider. Unison supports most popular SAML2 identity providers (ie ADFS, Shiboleth, etc). In addition Unison can also be an identity provider.

If you plan on integrating jBPM with applications via the web services APIs, Unison supports OAuth2 bearer tokens. See the reference guide for how to configure them.

Tremolo Security
----------------
Useful links

* [Tremolo Security Home](https://www.tremolosecurity.com/)
* [Wiki Home](https://www.tremolosecurity.com/wiki/#!index.md)
* [Tremolo Security on Github](https://www.github.com/tremolosecurity/)
* Follow us on Twitter - [gimmick:twitterfollow](tremolosecurity) @tremolosecurity
