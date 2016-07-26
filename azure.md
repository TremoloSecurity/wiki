Introduction
-------------------
With the agreement between Red Hat and Microsoft for Red Hat Enterprise Linux (RHEL) to be a supported platform on Azure, it opens a new cloud and opportunities for your enterprise.  Once you start deploying RHEL to Azure, how will you manage access?  This white paper provides the technical details around how we setup the below demo video that shows how you can combine OpenUnison and Red Hat Identity Management (aka FreeIPA) to manage access to RHEL servers as well as some common system administration applications that you might run on your Azure cloud to more easily manage and secure it.  Here's the demo video:  

<iframe src="https://player.vimeo.com/video/160002916" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
<p><a href="https://vimeo.com/160002916">Manage Linux Identities on Azure with FreeIPA and OpenUnison</a> from <a href="https://vimeo.com/tremolo">Tremolo Security Inc.</a> on <a href="https://vimeo.com">Vimeo</a>.</p>

## Pre-Requisites

Before deploying anything, choose a domain.  We chose a domain (azure.cloud) that wouldn't conflict with our internal domain (ent2k12.domain.com).  Once we decided on a domain, we built out our RHIdM server on RHEL 7 following [the instructions](https://access.redhat.com/products/identity-management#getstarted) on Red Hat's web site.  Once RHIdM was setup, the next step was to deploy RHEL servers for OpenUnison, GrayLog and our test server for logins, with all servers being members of the domain azure.cloud.  On the test server, we also installed cockpit.  We connected GrayLog to FreeIPA over LDAP, with all groups for GrayLog starting with "graylog-".

**Side Note: ** The plan was to use Docker but we couldn't get S4U2Self tickets working inside of a docker image.

Finally, once the OpenUnison server was setup we downloaded Tomcat 8 from http://tomcat.apache.org and installed:

1. Git
2. OpenJDK 8 __<-- MUST BE 8__
3. Maven

We also setup port forwarding from 80->8080 and 443->8443:
```
$ firewall-cmd --zone=public --add-port=80/tcp --permanent
$ firewall-cmd --zone=public --add-port=443/tcp --permanent
$ firewall-cmd --zone=public --add-port=8080/tcp --permanent
$ firewall-cmd --zone=public --add-port=8443/tcp --permanent
$ firewall-cmd --zone=public --add-masquerade --permanent
$ firewall-cmd --zone=public --add-forward-port=port=443:proto=tcp:toport=8443 --permanent
$ firewall-cmd --zone=public --add-forward-port=port=80:proto=tcp:toport=8080 --permanent
$ systemctl restart firewalld
$ systemctl enable firewalld
```

## Setting up OpenUnison

Before setting up OpenUnison, we had to "download and install" some client libraries.  The code for FreeIPA/RHIdM integration, ScaleJS and Kerberos LastMile are not a part of the main OpenUnison source tree as of 1.0.6 (they'll be merged in for 1.0.7).  So to get everything working the easiest thing is to build from source:

```
$ git clone https://github.com/TremoloSecurity/Unison-FreeIPA.git
$ cd Unison-FreeIPA/unison-services-freeipa
$ mvn install
$ cd ../..
$ git clone https://github.com/TremoloSecurity/Unison-LastMile-Kerberos.git
$ cd Unison-LastMile-Kerberos/unison-lastmile-kerberos
$ mvn install
$ cd ../..
$ git clone https://github.com/TremoloSecurity/ScaleJS.git
$ cd ScaleJS
$ mvn install
```

Once all the libraries are built and installed into the local maven repository we can start setting up OpenUnison.  First we created a project directory and added the following pom.xml file built from [OpenUnison Documentation](https://www.tremolosecurity.com/docs/tremolosecurity-docs/1.0.6/openunison/openunison.html):

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.mycompany.openunison</groupId>
  <artifactId>openunison</artifactId>
  <packaging>war</packaging>
  <version>1.0-SNAPSHOT</version>
  <name>openunison Maven Webapp</name>
  <url>http://maven.apache.org</url>
  <repositories>
    <repository>
      <id>Tremolo Security</id>
      <url>https://www.tremolosecurity.com/nexus/content/repositories/releases/</url>
    </repository>
  </repositories>
  <dependencies>
    <dependency>
      <groupId>com.tremolosecurity.unison</groupId>
      <artifactId>open-unison-webapp</artifactId>
      <version>1.0.6</version>
      <type>war</type>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>com.tremolosecurity.unison</groupId>
      <artifactId>open-unison-webapp</artifactId>
      <version>1.0.6</version>
      <type>pom</type>
    </dependency>
    <dependency>
      <groupId>com.tremolosecurity.unison</groupId>
      <artifactId>unison-services-freeipa</artifactId>
      <version>1.0.7</version>
      <type>jar</type>
    </dependency>
    <dependency>
      <groupId>com.tremolosecurity.unison</groupId>
      <artifactId>unison-lastmile-kerberos</artifactId>
      <version>1.0.7</version>
      <type>jar</type>
    </dependency>
    <dependency>
      <groupId>com.tremolosecurity.unison</groupId>
      <artifactId>unison-services-freeipa</artifactId>
      <version>1.0.7</version>
      <type>jar</type>
    </dependency>
    <dependency>
      <groupId>com.tremolosecurity.unison</groupId>
      <artifactId>unison-scalejs-main</artifactId>
      <version>1.0.6</version>
      <type>jar</type>
    </dependency>
    <dependency>
      <groupId>com.tremolosecurity.unison</groupId>
      <artifactId>unison-scalejs-token</artifactId>
      <version>1.0.6</version>
      <type>jar</type>
    </dependency>
    <dependency>
      <groupId>org.apache.qpid</groupId>
      <artifactId>qpid-jms-client</artifactId>
      <version>0.8.0</version>
    </dependency>
    <dependency>
      <groupId>org.graylog2</groupId>
      <artifactId>gelfj</artifactId>
      <version>1.1.14</version>
      <scope>compile</scope>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.1</version>
        <configuration>
          <source>1.7</source>
          <target>1.7</target>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-war-plugin</artifactId>
        <version>2.6</version>
        <configuration>
          <overlays>
            <overlay>
              <groupId>com.tremolosecurity.unison</groupId>
              <artifactId>open-unison-webapp</artifactId>
            </overlay>
          </overlays>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

Then created the webapp directory:
```
$ cd openunison
$ mkdir -p src/main/webapp/META-INF
$ mkdir -p src/main/webapp/WEB-INF/lib
```

Based on the documentation for OpenUnison, created META-INF/context.xml with the following content:
```
<Context>
  <Environment name="unisonConfigPath" value="/etc/openunison/unison.xml" type="java.lang.String"/>
  <Environment name="unisonLog4jPath" value="/etc/openunison/log4j.xml" type="java.lang.String"/>
  <Environment name="unisonServiceConfigPath" value="/etc/openunison/unisonService.props" type="java.lang.String"/>
</Context>
```

Since Microsoft doesn't have its own Maven repository, you'll need to download the [SQL Server JDBC drivers from Microsoft](https://msdn.microsoft.com/en-us/library/mt484311.aspx) and place them in src/main/webapp/WEB-INF/lib.

Next, create `/etc/openunison` and make sure it has read/write rights for the user that will run Tomcat.  Finally, create `/etc/openunison/log4j.xml` with the following content:

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE log4j:configuration SYSTEM "log4j.dtd">
<log4j:configuration xmlns:log4j="http://jakarta.apache.org/log4j/">
  <appender name="console" class="org.apache.log4j.ConsoleAppender">
    <param name="Target" value="System.out"/>
    <layout class="org.apache.log4j.PatternLayout">
      <param name="ConversionPattern" value="%-5p %c{1} - %m%n"/>
    </layout>
  </appender>

  <root>
    <priority value ="info" />
    <appender-ref ref="console" />
  </root>

</log4j:configuration>
```

Finally, for initial setup create a file called `/etc/openunison/unionService.props` with the following content:

```
com.tremolosecurity.openunison.forceToSSL=true
com.tremolosecurity.openunison.openPort=80
com.tremolosecurity.openunison.securePort=443
com.tremolosecurity.openunison.externalOpenPort=8080
com.tremolosecurity.openunison.externalSecurePort=8443
#Uncomment and set for production deployments
#com.tremolosecurity.openunison.activemqdir=/var/lib/unison-activemq
```

## Create the KeyStore

In order for OpenUnison to work, we need to create a keystore in `/etc/openunison/unisonKeyStore.jks`:

```
$ keytool -genseckey -alias session-unison -keyalg AES -keysize 256 -storetype JCEKS -keystore /etc/openunison/unisonKeyStore.jks
Enter keystore password:
Re-enter new password:
Enter key password for <session-unison>
 (RETURN if same as keystore password):

$ keytool -genkeypair -storetype JCEKS -alias unison-tls -keyalg RSA -keysize 2048 -sigalg SHA256withRSA -keystore /etc/openunison/unisonKeyStore.jks
Enter keystore password:
What is your first and last name?
.
.
.

$ keytool -genseckey -alias session-queues  -keyalg AES -keysize 256 -storetype JCEKS -keystore ./unisonKeyStore.jks
.
.
.

$ keytool -genseckey -alias session-workflows  -keyalg AES -keysize 256 -storetype JCEKS -keystore ./unisonKeyStore.jks
$ keytool -genseckey -alias session-randompwd -keyalg AES -keysize 256 -storetype JCEKS -keystore /etc/openunison/unisonKeyStore.jks
```

Also, since RHIdM's ipaweb application runs on TLS we should trust the certificate by first getting the root CA cert out of RHIdM then trusting it:

```
keytool -import -keystore /etc/openunison/unisonKeyStore.jks -storeType JCEKS -alias ipaclient -rfc -file /etc/openunison/ipaclient.pem -trustcacerts
```

## Configuring the Virtual Directory

OpenUnison's internal virtual directory, built on [MyVirtualDirectory](https://myvd.sourceforge.net/), needs to be told to load identity data from the 389 server running on RHIdM.  Since OpenUnison assumes an inetOrgPerson and groupOfUniqueNames objectClass we needed to tweak the standard configuration to allow for mapping of posix groups to groupOfUniqueNames.  Create a file called `/etc/openunison/myvd.conf` with the following content:

```
#Global AuthMechConfig
server.globalChain=

server.nameSpaces=rootdse,myvdroot,freeipa
server.rootdse.chain=dse
server.rootdse.nameSpace=
server.rootdse.weight=0
server.rootdse.dse.className=net.sourceforge.myvd.inserts.RootDSE
server.rootdse.dse.config.namingContexts=o=Tremolo
server.myvdroot.chain=root
server.myvdroot.nameSpace=o=Tremolo
server.myvdroot.weight=0
server.myvdroot.root.className=net.sourceforge.myvd.inserts.RootObject

server.freeipa.chain=dnmapper,groupmapper,membermapper,LDAP
server.freeipa.nameSpace=ou=azure.cloud,o=Tremolo
server.freeipa.weight=100
server.freeipa.dnmapper.className=net.sourceforge.myvd.inserts.mapping.DNAttributeMapper
server.freeipa.dnmapper.config.dnAttribs=manager,uniqueMember,distinguishedName,memberOf
server.freeipa.dnmapper.config.urlAttribs=
server.freeipa.dnmapper.config.remoteBase=cn=accounts,dc=azure,dc=cloud
server.freeipa.dnmapper.config.localBase=ou=azure.cloud,o=Tremolo
server.freeipa.groupmapper.className=net.sourceforge.myvd.inserts.mapping.AttributeValueMapper
server.freeipa.groupmapper.config.mapping=objectClass.groupofnames=groupOfUniqueNames
server.freeipa.membermapper.className=net.sourceforge.myvd.inserts.mapping.AttributeMapper
server.freeipa.membermapper.config.mapping=uniqueMember=member
server.freeipa.LDAP.className=net.sourceforge.myvd.inserts.ldap.LDAPInterceptor
server.freeipa.LDAP.config.host=ipa.azure.cloud
server.freeipa.LDAP.config.port=389
server.freeipa.LDAP.config.proxyDN=cn=Directory Manager
server.freeipa.LDAP.config.proxyPass=SomePassword
server.freeipa.LDAP.config.remoteBase=cn=accounts,dc=azure,dc=cloud
server.freeipa.LDAP.config.type=ldap
```
Make sure to update __server.freeipa.LDAP.config.proxyPass__ with the correct password (obviously you wouldn't use cn=Directory Manager in real life).

## Creating a Kerberos Keytab

S4U2Self (and S4U2Proxy) are extensions to Kerberos introduced by Microsoft to make it possible to generate a Kerberos ticket on behalf of a user without knowing their password.  This is important in this situation because we don't know the user's password and ipaweb doesn't support any other form of SSO other then Kerberos.  Since we're going to be using S4U2Self to SSO into ipaweb we'll need a keytab from RHIdM and copy it to `/etc/openunison/unison-s4u.keytab` using the [instructions from the GitHub repository](https://github.com/TremoloSecurity/Unison-FreeIPA/blob/master/README.md).   

## Configure the SQL Server Database

Since we're running on Azure we wanted to use their SQL Server as a Service for our audit database.  The easiest way to do this is to run the [SQL file for SQL Server](https://raw.githubusercontent.com/TremoloSecurity/UnisonAdditionalSources/master/sql/unison-sqlserver-auditdb.sql) against our newly created database through Visual Studio.

## Configure Azure Service Bus

OpenUnison needs a JMS server of some kind for reliability and scaleability.  Every task that executes in a workflow is translated into an encrypted package that is placed on the queue.  If the server dies, or a directory won't respond, or you are clustering OpenUnison for high availability the messages on the queue are preserved.  Any JMS 1.1 compliant queue will do, but why set something up when Azure provides one with no server footprint?

Create an Azure Service Bus in Azure using the classic console (as of now Microsoft hasn't integrated it into the new console).  Once you'd picked a name, make sure to get the SharedAccessKey.  Create two queues:

1. tremolounisonsmtpqueue
2. tremolounisontaskqueue

Hold onto the connection host name and the SharedAccessKey for later.

## Configure OpenUnison

The main configuration file for OpenUnison is `/etc/openunison/unison.xml`.  Create this file using the following template:
```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<tremoloConfig xmlns="http://www.tremolosecurity.com/tremoloConfig">
  <applications openSessionCookieName="openSession" openSessionTimeout="9000">
    <application name="LinuxServices" azTimeoutMillis="30000">
      <urls>
        <url regex="false" authChain="adfs" overrideHost="true" overrideReferer="true">
          <host>freeipa.azure.cloud</host>
          <filterChain>
            <filter class="com.tremolosecurity.proxy.filters.HideCookie" />
            <filter class="com.tremolosecurity.unison.proxy.lastmile.kerberos.KerberosLastMile">
              <param name="uidAttributeName" value="uid"/>
              <param name="targetServicePrincipal" value="HTTP/ipa.azure.cloud@AZURE.CLOUD"/>
              <param name="keytabPath" value="/etc/openunison/unison-s4u.keytab"/>
              <param name="keytabPrincipal" value="HTTP/openunison.azure.cloud@AZURE.CLOUD"/>
            </filter>
          </filterChain>
          <uri>/</uri>
          <proxyTo>https://ipa.azure.cloud${fullURI}</proxyTo>
          <results>
            <auFail>Invalid Login</auFail>
            <azSuccess>
            </azSuccess>
            <azFail>Invalid Login</azFail>
          </results>
          <azRules>
            <rule scope="group" constraint="cn=admins,cn=groups,ou=azure.cloud,o=Tremolo"/>
          </azRules>
        </url>


      </urls>
      <cookieConfig>
        <sessionCookieName>tremolosession</sessionCookieName>
        <domain>azure.cloud</domain>
        <scope>-1</scope>
        <logoutURI>/logout</logoutURI>
        <keyAlias>session-unison</keyAlias>
        <secure>false</secure>
        <timeout>900</timeout>
      </cookieConfig>
    </application>
    <application name="Scale" azTimeoutMillis="30000">
      <urls>
        <url regex="false" authChain="adfs" overrideHost="true" overrideReferer="true">
          <host>openunison.azure.cloud</host>
          <filterChain>
          </filterChain>
          <uri>/scale</uri>
          <results>
            <auFail>Invalid Login</auFail>
            <azSuccess>
            </azSuccess>
            <azFail>Invalid Login</azFail>
          </results>
          <azRules>
            <rule scope="dn" constraint="o=Tremolo"/>
          </azRules>
        </url>
        <url regex="false" authChain="adfs" overrideHost="true" overrideReferer="true">
          <host>openunison.azure.cloud</host>
          <filterChain>
            <filter class="com.tremolosecurity.prelude.filters.LoginTest">
              <param name="logoutURI" value="/logout"/>
            </filter>
          </filterChain>
          <uri>/testlogin</uri>
          <proxyTo>http://dnm${fullURI}</proxyTo>
          <results>
            <auFail>Invalid Login</auFail>
            <azSuccess>
            </azSuccess>
            <azFail>Invalid Login</azFail>
          </results>
          <azRules>
            <rule scope="dn" constraint="o=Tremolo"/>
          </azRules>
        </url>
        <url regex="false" authChain="adfs" overrideHost="true" overrideReferer="true">
          <host>openunison.azure.cloud</host>
          <filterChain>
            <filter class="com.tremolosecurity.prelude.filters.StopProcessing"/>
          </filterChain>
          <uri>/</uri>

          <results>
            <auFail>Invalid Login</auFail>
            <azSuccess>ScaleRedirect</azSuccess>
            <azFail>Invalid Login</azFail>
          </results>
          <azRules>
            <rule scope="dn" constraint="o=Tremolo"/>
          </azRules>
        </url>

        <url regex="false" authChain="adfs" overrideHost="false" overrideReferer="false">
          <host>openunison.azure.cloud</host>
          <filterChain>
            <filter class="com.tremolosecurity.scalejs.ws.ScaleMain">
              <param name="displayNameAttribute" value="displayName"/>
              <param name="frontPage.title" value="Azure Cloud Linux Management"/>
              <param name="frontPage.text" value="Use this portal as the gateway for accessing your linux servers and requesting access to systems."/>
              <param name="canEditUser" value="true"/>
              <param name="workflowName" value="ipa-update-sshkey"/>
              <param name="attributeNames" value="uid"/>
              <param name="uid.displayName" value="Login ID"/>
              <param name="uid.readOnly" value="true"/>
              <param name="attributeNames" value="sn"/>
              <param name="sn.displayName" value="Last Name"/>
              <param name="sn.readOnly" value="true"/>
              <param name="attributeNames" value="givenName"/>
              <param name="givenName.displayName" value="First Name"/>
              <param name="givenName.readOnly" value="true"/>
              <param name="attributeNames" value="displayName"/>
              <param name="displayName.displayName" value="Display Name"/>
              <param name="displayName.readOnly" value="true"/>
              <param name="attributeNames" value="ipaSshPubKey"/>
              <param name="ipaSshPubKey.displayName" value="SSH Public Key"/>
              <param name="ipaSshPubKey.readOnly" value="false"/>
              <param name="ipaSshPubKey.required" value="true"/>
              <param name="attributeNames" value="loginShell"/>
              <param name="loginShell.displayName" value="Login Shell"/>
              <param name="loginShell.readOnly" value="false"/>
              <param name="loginShell.required" value="true"/>
              <param name="uidAttributeName" value="uid"/>
              <param name="roleAttribute" value=""/>
              <param name="approvalAttributeNames" value="uid"/>
              <param name="approvalAttributeNames" value="givenName"/>
              <param name="approvalAttributeNames" value="sn"/>
              <param name="approvalAttributeNames" value="mail"/>
              <param name="approvalAttributeNames" value="displayName"/>
              <param name="approvals.uid" value="Login ID"/>
              <param name="approvals.givenName" value="First Name"/>
              <param name="approvals.sn" value="Last Name"/>
              <param name="approvals.mail" value="Email Address"/>
              <param name="approvals.displayName" value="Display Name"/>
              <param name="showPortalOrgs" value="false"/>
              <param name="logoutURL" value="/logout"/>
            </filter>
          </filterChain>
          <uri>/scale/main</uri>
          <results>
            <auSuccess>
            </auSuccess>
            <auFail>
            </auFail>
            <azSuccess>
            </azSuccess>
            <azFail>
            </azFail>
          </results>
          <azRules>
            <rule scope="dn" constraint="o=Tremolo"/>
          </azRules>
        </url>

        <url regex="false" authChain="adfs" overrideHost="false" overrideReferer="false">
          <host>openunison.azure.cloud</host>
          <filterChain>
            <filter class="com.tremolosecurity.scalejs.token.ws.ScaleToken">
              <param name="displayNameAttribute" value="displayName"/>
              <param name="frontPage.title" value="Azure Cloud Linux Temporary Password"/>
              <param name="frontPage.text" value="Use this password for applications that do not support SSO"/>
              <param name="logoutURL" value="/logout"/>
              <param name="homeURL" value="/scale/index.html"/>
              <param name="attributeName" value="carLicense" />
              <param name="encryptionKey" value="session-randompwd" />
              <param name="tokenClassName" value="com.tremolosecurity.scalejs.token.password.LoadToken" />
            </filter>
          </filterChain>
          <uri>/scale-token/token</uri>
          <results>
            <auSuccess>
            </auSuccess>
            <auFail>
            </auFail>
            <azSuccess>
            </azSuccess>
            <azFail>
            </azFail>
          </results>
          <azRules>
            <rule scope="dn" constraint="o=Tremolo"/>
          </azRules>
        </url>

        <url regex="false" authChain="adfs" overrideHost="true" overrideReferer="true">
          <host>openunison.azure.cloud</host>
          <filterChain>
            <filter class="com.tremolosecurity.prelude.filters.StopProcessing"/>
          </filterChain>
          <uri>/logout</uri>
          <proxyTo>http://dnm${fullURI}</proxyTo>
          <results>
            <azSuccess>Logout</azSuccess>
          </results>
          <azRules>
            <rule scope="dn" constraint="o=Tremolo"/>
          </azRules>
        </url>
      </urls>
      <cookieConfig>
        <sessionCookieName>tremolosession</sessionCookieName>
        <domain>azure.cloud</domain>
        <scope>-1</scope>
        <logoutURI>/logout</logoutURI>
        <keyAlias>session-unison</keyAlias>
        <secure>false</secure>
        <timeout>900</timeout>
      </cookieConfig>
    </application>
  </applications>
  <myvdConfig>/etc/openunison/myvd.conf</myvdConfig>
  <authMechs>
    <mechanism name="loginForm">
      <uri>/auth/formLogin</uri>
      <className>com.tremolosecurity.proxy.auth.FormLoginAuthMech</className>
      <init/>
      <params>
        <param>FORMLOGIN_JSP</param>
      </params>
    </mechanism>
    <mechanism name="anonymous">
      <uri>/auth/anon</uri>
      <className>com.tremolosecurity.proxy.auth.AnonAuth</className>
      <init>
        <param name="userName" value="uid=Anonymous"/>
        <param name="role" value="Users"/>
      </init>
      <params/>
    </mechanism>
    <mechanism name="certAuth">
      <uri>/auth/ssl</uri>
      <className>com.tremolosecurity.proxy.auth.CertAuth</className>
      <init>
        <param name="crl.names" value=""/>
      </init>
      <params/>
    </mechanism>
    <mechanism name="SAML2">
      <uri>/auth/SAML2Auth</uri>
      <className>com.tremolosecurity.proxy.auth.SAML2Auth</className>
      <init/>
      <params/>
    </mechanism>
    <mechanism name="jit">
      <uri>/auth/oath2</uri>
      <className>com.tremolosecurity.provisioning.auth.JITAuthMech</className>
      <init>
      </init>
      <params>
      </params>
    </mechanism>
    <mechanism name="resetpw">
      <uri>/auth/clearpwd</uri>
      <className>com.tremolosecurity.scalejs.token.password.RegisterPasswordResetAuth</className>
      <init>
        <param name="workflowName" value="reset-password-on-logout" />
        <param name="uidAttributeName" value="uid" />
      </init>
      <params>
      </params>
    </mechanism>
  </authMechs>
  <authChains>
    <chain name="anon" level="0">
      <authMech>
        <name>anonymous</name>
        <required>required</required>
        <params/>
      </authMech>
    </chain>
    <chain name="formloginFilter" level="20">
      <authMech>
        <name>loginForm</name>
        <required>required</required>
        <params>
          <param name="FORMLOGIN_JSP" value="/auth/forms/defaultForm.jsp"/>
          <param name="uidAttr" value="uid"/>
          <param name="uidIsFilter" value="false"/>
        </params>
      </authMech>
    </chain>
    <chain name="adfs" level="20">
      <authMech>
        <name>SAML2</name>
        <required>required</required>
        <params>
          <param name="entityID" value="https://idp.ent2k12.domain.com/adfs/services/trust"/>
          <param name="idpRedirURL" value="https://idp.ent2k12.domain.com/adfs/ls/"/>
          <param name="idpURL" value="https://idp.ent2k12.domain.com/adfs/ls/"/>
          <param name="idpRedirLogoutURL" value="https://idp.ent2k12.domain.com/adfs/ls/"/>
          <param name="idpSigKeyName" value="verify-https://idp.ent2k12.domain.com/adfs/services/trust-idp-sig"/>
          <param value="uid" name="ldapAttribute" />
          <param value="ADFS" name="dnOU" />
          <param value="inetOrgPerson" name="defaultOC" />
          <param name="authCtxRef" value="urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport"/>
          <param name="spSigKey" value="unison-tls" />
          <param name="logoutURL" value="https://openunison.azure.cloud/auth/forms/logout.jsp" />
          <param name="sigAlg" value="RSA-SHA256" />
          <param name="signAuthnReq" value="true"/>
          <param name="assertionsSigned" value="true" />
          <param name="responsesSigned" value="false" />
          <param name="assertionEncrypted" value="false" />
          <param name="spEncKey" value="" />
          <param name="forceToSSL" value="false" />
          <param name="jumpPage" value="" />
          <param name="dontLinkToLDAP" value="true" />
        </params>
      </authMech>
      <authMech>
        <name>jit</name>
        <required>required</required>
        <params>
          <!-- The name of the attribute that will be used as the user identifier-->
          <param name="nameAttr" value="uid" />
          <!-- The name of the workflow -->
          <param name="workflowName" value="ipa-jit" />
        </params>
      </authMech>
      <authMech>
        <name>resetpw</name>
        <required>required</required>
        <params />
      </authMech>
    </chain>
  </authChains>
  <resultGroups>
    <resultGroup name="ScaleRedirect">
      <result>
        <type>redirect</type>
        <source>static</source>
        <value>/scale/index.html</value>
      </result>
    </resultGroup>
    <resultGroup name="Logout">
      <result>
        <type>redirect</type>
        <source>static</source>
        <value>/auth/forms/logout.jsp</value>
      </result>
    </resultGroup>
    <resultGroup name="Invalid Login">
      <result>
        <type>redirect</type>
        <source>static</source>
        <value>/auth/forms/defaultFailedLogin.jsp</value>
      </result>
    </resultGroup>
  </resultGroups>
  <keyStorePath>/etc/openunison/unisonKeyStore.jks</keyStorePath>
  <keyStorePassword>XXXXXX</keyStorePassword>
  <provisioning>
    <approvalDB>
      <!-- The JDBC Driver, must be part of the classpath -->
      <driver>com.microsoft.sqlserver.jdbc.SQLServerDriver</driver>
      <!-- JDBC URL for the database -->
      <url>jdbc:sqlserver://XXXXXXX.database.windows.net:1433;database=openunison;encrypt=true;trustServerCertificate=false;hostNameInCertificate=*.database.windows.net;loginTimeout=30;</url>
      <!-- User for accessing the database -->
      <user>XXXX@XXXX</user>
      <!-- Password for the database -->
      <password>XXXX</password>
      <!-- Maximum number of connections to the database -->
      <maxConns>10</maxConns>
      <!-- Maxixum number of idle connections -->
      <maxIdleConns>10</maxIdleConns>
      <!-- The name of the attribute on the user that is a unique identifier -->
      <userIdAttribute>uid</userIdAttribute>
      <!-- Query used to verify that the connection is alive -->
      <validationQuery>SELECT 1</validationQuery>
      <!-- The list of attributes to record in the approvers table.  NOTE: each value tag MUST line up with a column created on the approvers table -->
      <approverAttributes>
        <value>uid</value>
        <value>givenName</value>
        <value>sn</value>
        <value>mail</value>
      </approverAttributes>
      <!-- The list of attributes to record in the users table.  NOTE: each value tag MUST line up with a column created on the users table -->
      <userAttributes>
        <value>uid</value>
        <value>givenName</value>
        <value>sn</value>
        <value>mail</value>
      </userAttributes>

      <maskAttribute>carLicense</maskAttribute>
      <maskAttribute>carlicense</maskAttribute>

      <!-- determines if the audit database should be enabled, not required for using Just-In-Time provisioning -->
      <enabled>true</enabled>
      <!-- The alias of the key used to encrypt all workflows stored in the database -->
      <encryptionKey>session-workflows</encryptionKey>
      <!-- SMTP Server host name -->
      <smtpHost>smtp.gmail.com</smtpHost>
      <!-- SMTP Server port -->
      <smtpPort>587</smtpPort>
      <!-- User for accessing the SMTP server -->
      <smtpUser>donotreply@test.com</smtpUser>
      <!-- Password for the smtp server -->
      <smtpPassword>sdfgsdfgsdfgdsfgsdfgdsfgdsfg</smtpPassword>
      <!-- Subject of emails sent to approvers -->
      <smtpSubject>Awaiting Approvals</smtpSubject>
      <!-- Email address for the "From" field of all emails -->
      <smtpFrom>donotreply@tedst.com</smtpFrom>
      <!-- Determines if TLS should be used with the SMTP server -->
      <smtpTLS>true</smtpTLS>
      <!-- Set to true if an outbound SOCKS proxy is required for access to the SMTP server -->
      <smtpUseSOCKSProxy>false</smtpUseSOCKSProxy>
      <!-- Host of the SOCKS proxy -->
      <smtpSOCKSProxyHost>
      </smtpSOCKSProxyHost>
      <!-- Port of the SOCKS proxy -->
      <smtpSOCKSProxyPort>0</smtpSOCKSProxyPort>
      <!-- How Unison should identify its self to the SMTP server -->
      <smtpLocalhost>
      </smtpLocalhost>
    </approvalDB>
    <queueConfig isUseInternalQueue="false" encryptionKeyName="session-queues"  taskQueueName="TremoloUnisonTaskQueue" smtpQueueName="TremoloUnisonSMTPQueue" connectionFactory="org.apache.qpid.jms.JmsConnectionFactory" maxProducers="1" maxConsumers="1">
      <param name="remoteURI" value="amqps://myqueues.servicebus.windows.net?jms.ConnectTimeout=60000&amp;amqp.idleTimeout=180000" />
      <param name="username" value="RootManageSharedAccessKey" />
      <param name="password" value="XXXXXXXXXXXXX" />

    </queueConfig>
    <listeners />
    <scheduler useDB="false" threadCount="3" instanceLabel="OpenUnisonFreeIPA" instanceIPMask="127" />
    <targets >
      <target name="freeipa" className="com.tremolosecurity.unison.freeipa.FreeIPATarget">
        <params>
          <param name="url" value="https://ipa.azure.cloud" />
          <param name="userName" value="admin" />
          <param name="password" value="XXXXXXXXXXX" />
          <param name="createShadowAccounts" value="false" />
        </params>
        <targetAttribute name="sn" sourceType="user" source="sn" />
        <targetAttribute name="givenname" sourceType="user" source="givenName" />
        <targetAttribute name="mail" sourceType="user" source="mail" />
        <targetAttribute name="uid" sourceType="user" source="uid" />
        <targetAttribute name="cn" sourceType="user" source="uid" />
        <targetAttribute name="displayname" sourceType="user" source="displayName" />
        <targetAttribute name="gecos" sourceType="user" source="displayName" />
        <targetAttribute name="carlicense" sourceType="user" source="carlicense" />
      </target>
      <target name="freeipa-sshkey" className="com.tremolosecurity.unison.freeipa.FreeIPATarget">
        <params>
          <param name="url" value="https://ipa.azure.cloud" />
          <param name="userName" value="admin" />
          <param name="password" value="XXXXXXXXXXX" />
          <param name="createShadowAccounts" value="false" />
        </params>
        <targetAttribute name="uid" sourceType="user" source="uid" />
        <targetAttribute name="ipasshpubkey" sourceType="user" source="ipasshpubkey" />
        <targetAttribute name="loginshell" sourceType="user" source="loginshell" />
      </target>
      <target name="freeipa-groups" className="com.tremolosecurity.unison.freeipa.FreeIPATarget">
        <params>
          <param name="url" value="https://ipa.azure.cloud" />
          <param name="userName" value="admin" />
          <param name="password" value="XXXXXXXXXXX" />
          <param name="createShadowAccounts" value="false" />
        </params>
        <targetAttribute name="uid" sourceType="user" source="uid" />
      </target>
    </targets>
    <org name="MyOrg" description="MyOrg Azure Cloud" uuid="687da09f-8ec1-48ac-b035-f2f182b9bd1e" >
      <azRules/>
      <orgs name="Administrator Access" description="Various administrative roles" uuid="e6b5b365-9dc4-47f4-803f-a1255cf31b57">
        <azRules />
      </orgs>
      <orgs name="Log Management" description="Log management roles" uuid="5d6bab30-9dd9-4cfd-8aff-a395fed377dc">
        <azRules />
      </orgs>
      <orgs name="Identity Management Reports" description="Reports Available to Identity Management Admins" uuid="1c2cad5b-f62c-491c-84f3-068f6231f053">
        <azRules>
          <rule scope="group" constraint="cn=admins,cn=groups,ou=azure.cloud,o=Tremolo"/>
        </azRules>
      </orgs>

    </org>
    <reports>
      <report orgID="687da09f-8ec1-48ac-b035-f2f182b9bd1e" name="My Open Requests" description="List of your currently open requests and the approvers responsible for acting on them" groupBy="id" groupings="true">
        <paramater>currentUser</paramater>
        <sql>select  approvals.id,approvals.label AS Approval ,&#xD;
        approvals.createTS AS 'Approval Opened',&#xD;
        workflows.name AS 'Workflow Name',&#xD;
        workflows.requestReason AS 'Request Reason', &#xD;
        ([users].givenName + ' ' + users.sn) as 'Subject Name', &#xD;
        users.mail as 'Subject Email', &#xD;
        approvers.givenName as 'First Name',&#xD;
        approvers.sn as 'Last Name',&#xD;
        approvers.mail as 'Email'  &#xD;
from approvals inner join workflows on approvals.workflow=workflows.id inner join users on workflows.userid=users.id inner join allowedApprovers on approvals.id=allowedApprovers.approval inner join approvers on approvers.id=allowedApprovers.approver where users.userKey=? AND approvedTS is null order by approvals.createTS ASC, approvals.id ASC</sql>
        <headerFields>Approval</headerFields>
        <headerFields>Subject Name</headerFields>
        <headerFields>Subject Email</headerFields>
        <headerFields>Workflow Name</headerFields>
        <headerFields>Request Reason</headerFields>
        <dataFields>First Name</dataFields>
        <dataFields>Last Name</dataFields>
        <dataFields>Email</dataFields>
    </report>
    <report orgID="687da09f-8ec1-48ac-b035-f2f182b9bd1e" name="Approvals Completed by Me" description="All approvals you approved or denied" groupBy="wid" groupings="false">
        <paramater>currentUser</paramater>
        <sql>select  workflows.id AS wid, approvals.id AS aid,approvals.label AS Approval ,approvals.createTS AS 'Approval Opened',workflows.name AS 'Workflow Name',workflows.requestReason AS 'Request Reason', (users.givenName + ' ' + users.sn) as 'Subject Name', users.mail as 'Subject Email', approvers.givenName as 'First Name',approvers.sn as 'Last Name',approvers.mail as 'Email',&#xD;
&#xD;
&#xD;
(CASE WHEN approvals.approved = 1&#xD;
  THEN 'Approved'&#xD;
  ELSE 'Rejected'&#xD;
  END) AS 'Approval Result'&#xD;
&#xD;
&#xD;
&#xD;
,approvals.approvedTS AS 'Approved Date',approvals.reason AS Reason from approvals inner join approvers on approvals.approver=approvers.id inner join workflows on workflows.id=approvals.workflow inner join users on users.id=workflows.userid WHERE approvers.userKey=? order by approvals.approvedTS DESC; </sql>
        <dataFields>Workflow Name</dataFields>
        <dataFields>Subject Name</dataFields>
        <dataFields>Subject Email</dataFields>
        <dataFields>Request Reason</dataFields>
        <dataFields>Approval</dataFields>
        <dataFields>Approval Result</dataFields>
        <dataFields>Approved Date</dataFields>
    </report>











    <report orgID="1c2cad5b-f62c-491c-84f3-068f6231f053" name="Open Approvals" description="Lists all of the approvals that are currently waiting action" groupBy="id" groupings="true">
        <sql>select
          approvals.id,approvals.label AS Approval ,
          approvals.createTS AS 'Approval Opened',
          workflows.name AS 'Workflow Name',
          workflows.requestReason AS 'Request Reason',
          (users.givenName + ' ' + users.sn) as 'Subject Name',
          users.mail as 'Subject Email',
          approvers.givenName as 'First Name',
          approvers.sn as 'Last Name',approvers.mail as 'Email'
from approvals inner join workflows on approvals.workflow=workflows.id inner join users on workflows.userid=users.id inner join allowedApprovers on approvals.id=allowedApprovers.approval inner join approvers on approvers.id=allowedApprovers.approver
where approvedTS is null
order by approvals.createTS ASC, approvals.id ASC</sql>
        <headerFields>Approval</headerFields>
        <headerFields>Subject Name</headerFields>
        <headerFields>Subject Email</headerFields>
        <headerFields>Workflow Name</headerFields>
        <headerFields>Request Reason</headerFields>
        <dataFields>First Name</dataFields>
        <dataFields>Last Name</dataFields>
        <dataFields>Email</dataFields>
    </report>
    <report orgID="1c2cad5b-f62c-491c-84f3-068f6231f053" name="Completed Approvals" description="All approvals completed in a given set of dates" groupBy="wid" groupings="true">
        <paramater>beginDate</paramater>
        <paramater>endDate</paramater>
        <sql>select
	workflows.id AS wid,
	approvals.id AS aid,
	approvals.label AS Approval ,
	approvals.createTS AS 'Approval Opened',
	workflows.name AS 'Workflow Name',
	workflows.requestReason AS 'Request Reason',
	(users.givenName + ' ' + users.sn) as 'Subject Name',
	users.mail as 'Subject Email',
	approvers.givenName as 'First Name',
	approvers.sn as 'Last Name',
	approvers.mail as 'Email',
	CASE approvals.approved
		WHEN 1 THEN 'Approved'
		ELSE 'Rejected'
	END AS 'Approval Result',
	approvals.approvedTS AS 'Approved Date',
	approvals.reason AS Reason
from approvals inner join approvers on approvals.approver=approvers.id inner join workflows on workflows.id=approvals.workflow inner join users on users.id=workflows.userid
WHERE approvals.approvedTS &gt; ? AND approvals.approvedTS &lt; ? order by approvals.id ASC,workflows.id ASC
</sql>
        <headerFields>Workflow Name</headerFields>
        <headerFields>Subject Name</headerFields>
        <headerFields>Subject Email</headerFields>
        <headerFields>Request Reason</headerFields>
        <dataFields>Approval</dataFields>
        <dataFields>First Name</dataFields>
        <dataFields>Last Name</dataFields>
        <dataFields>Email</dataFields>
        <dataFields>Approval Result</dataFields>
    </report>
    <report orgID="1c2cad5b-f62c-491c-84f3-068f6231f053" name="Single User Change Log" description="All changes to the chosen user" groupBy="id" groupings="true">
        <paramater>userKey</paramater>
        <sql>select
users.givenName AS 'First Name', users.sn AS 'Last Name', users.mail AS 'Email Address' ,workflows.id,
workflows.name as 'Workflow Name',workflows.startTS AS 'Workflow Started',workflows.completeTS AS 'Workflow Completed',workflows.requestReason AS 'Request Reason',
auditLogType.name  AS 'Action',CASE WHEN isEntry = 1 THEN 'Object' ELSE 'Attribute' END AS 'Target Type',auditLogs.attribute AS 'Name',auditLogs.val AS 'Value'


 from users inner join auditLogs on users.id=auditLogs.userid inner join auditLogType on auditLogType.id=auditLogs.actionType inner join workflows on workflows.id=auditLogs.workflow where users.userKey=?
 order by workflows.completeTS ASC ,workflows.id ASC , auditLogs.isEntry DESC</sql>
        <headerFields>Workflow Name</headerFields>
        <headerFields>Request Reason</headerFields>
        <headerFields>Workflow Started</headerFields>
        <headerFields>Workflow Completed</headerFields>
        <headerFields>First Name</headerFields>
        <headerFields>Last Name</headerFields>
        <headerFields>Email Address</headerFields>
        <dataFields>Action</dataFields>
        <dataFields>Target Type</dataFields>
        <dataFields>Name</dataFields>
        <dataFields>Value</dataFields>
    </report>
    <report orgID="1c2cad5b-f62c-491c-84f3-068f6231f053" name="Change Log for Period" description="Changes to all users between the two selected dates" groupBy="id" groupings="true">
        <paramater>beginDate</paramater>
        <paramater>endDate</paramater>
        <sql>select
users.givenName AS 'First Name', users.sn AS 'Last Name', users.mail AS 'Email Address' ,workflows.id,
workflows.name as 'Workflow Name',workflows.startTS AS 'Workflow Started',workflows.completeTS AS 'Workflow Completed',workflows.requestReason AS 'Request Reason',
auditLogType.name  AS 'Action',CASE WHEN isEntry = 1 THEN 'Object' ELSE 'Attribute' END AS 'Target Type',auditLogs.attribute AS 'Name',auditLogs.val AS 'Value'


 from users inner join auditLogs on users.id=auditLogs.userid inner join auditLogType on auditLogType.id=auditLogs.actionType inner join workflows on workflows.id=auditLogs.workflow where workflows.completeTS &gt;= ? and workflows.completeTS &lt;= ?
 order by workflows.completeTS ASC ,workflows.id ASC , auditLogs.isEntry DESC</sql>
        <headerFields>Workflow Name</headerFields>
        <headerFields>Request Reason</headerFields>
        <headerFields>Workflow Started</headerFields>
        <headerFields>Workflow Completed</headerFields>
        <headerFields>First Name</headerFields>
        <headerFields>Last Name</headerFields>
        <headerFields>Email Address</headerFields>
        <dataFields>Action</dataFields>
        <dataFields>Target Type</dataFields>
        <dataFields>Name</dataFields>
        <dataFields>Value</dataFields>
    </report>
  </reports>
    <portal>
      <urls label="Red Hat IDM" url="https://freeipa.azure.cloud/" name="RedHatIdM" org="687da09f-8ec1-48ac-b035-f2f182b9bd1e" icon="">
        <!--
                                  The azRules will tell Unison if the link should be displayed to the user
                                -->
        <azRules>
          <rule scope="group" constraint="cn=admins,cn=groups,ou=azure.cloud,o=Tremolo"/>
        </azRules>
      </urls>
      <urls label="Cloud Password" url="/scale-token/index.html" name="ScaleTokenPassword" org="687da09f-8ec1-48ac-b035-f2f182b9bd1e" icon="">
        <azRules />
      </urls>
      <urls label="Cockpit" url="https://ou-ipaclient-rhel72.azure.cloud:9090/" name="Cockpit"  org="687da09f-8ec1-48ac-b035-f2f182b9bd1e" icon="">
        <azRules />
      </urls>
      <urls label="Graylog" url="http://logserver.azure.cloud:9000/" name="Graylog" org="687da09f-8ec1-48ac-b035-f2f182b9bd1e" icon="" >
        <azRules>
          <rule scope="group" constraint="cn=graylog-administrator,cn=groups,ou=azure.cloud,o=Tremolo"/>
          <rule scope="group" constraint="cn=graylog-reader,cn=groups,ou=azure.cloud,o=Tremolo"/>
        </azRules>
      </urls>
    </portal>
    <workflows>
      <workflow name="ipa-jit" label="" description="" inList="false" orgid="687da09f-8ec1-48ac-b035-f2f182b9bd1e">
        <mapping strict="true">
          <map>
            <mapping targetAttributeName="mail" targetAttributeSource="mail" sourceType="user"/>
            <mapping targetAttributeName="givenName" targetAttributeSource="givenName" sourceType="user"/>
            <mapping targetAttributeName="sn" targetAttributeSource="sn" sourceType="user"/>
            <mapping targetAttributeName="displayName" targetAttributeSource="displayName" sourceType="user"/>
            <mapping targetAttributeName="uid" targetAttributeSource="uid" sourceType="user"/>
          </map>
          <customTask className="com.tremolosecurity.scalejs.token.password.SetRandomPassword">
            <param name="keyName" value="session-randompwd" />
            <param name="attributeName" value="carlicense" />
          </customTask>
          <provision sync="false" target="freeipa" setPassword="true"/>
          <resync keepExternalAttrs="false" />
        </mapping>
      </workflow>
      <workflow name="ipa-update-sshkey" label="" description="" inList="false" orgid="687da09f-8ec1-48ac-b035-f2f182b9bd1e">
        <mapping strict="true">
          <map>
            <mapping targetAttributeName="uid" targetAttributeSource="uid" sourceType="user"/>
            <mapping targetAttributeName="ipasshpubkey" targetAttributeSource="ipaSshPubKey" sourceType="user"/>
            <mapping targetAttributeName="loginshell" targetAttributeSource="loginShell" sourceType="user"/>
          </map>
          <customTask className="com.tremolosecurity.provisioning.customTasks.LoadGroups">
            <!-- The attribute name to search for on the user's account   -->
            <param name="nameAttr" value="uid"/>
            <!-- If set to true, only loads the groups from the virtual directory that the user's object is NOT already a member of               -->
            <param name="inverse" value="false"/>
          </customTask>
          <provision sync="true" target="freeipa-sshkey" setPassword="false"/>
        </mapping>
      </workflow>
      <workflow name="request-admin" label="Cloud Administrator" description="Administrator access to cloud resources" inList="true" orgid="e6b5b365-9dc4-47f4-803f-a1255cf31b57">
        <approval label="Approve Cloud Administrator">
          <emailTemplate>
                                          ${givenName},
                                          You have open approvals
                                        </emailTemplate>
            <approvers>
              <rule scope="group" constraint="cn=admin-approvers,cn=groups,ou=azure.cloud,o=Tremolo" />
            </approvers>
            <mapping strict="true">
              <map>
                <mapping targetAttributeName="uid" targetAttributeSource="uid" sourceType="user"/>
              </map>
              <addGroup name="admins" />
              <provision sync="false" target="freeipa-groups" setPassword="false"/>
            </mapping>
          </approval>
        </workflow>
        <workflow name="request-sudoall" label="Sudo All" description="Sudo All access on all servers" inList="true" orgid="e6b5b365-9dc4-47f4-803f-a1255cf31b57">
          <approval label="Approve Sudo All">
            <emailTemplate>
                                              ${givenName},
                                              You have open approvals
                                            </emailTemplate>
              <approvers>
                <rule scope="group" constraint="cn=admin-approvers,cn=groups,ou=azure.cloud,o=Tremolo" />
              </approvers>
              <mapping strict="true">
                <map>
                  <mapping targetAttributeName="uid" targetAttributeSource="uid" sourceType="user"/>
                </map>
                <addGroup name="sudoalls" />
                <provision sync="false" target="freeipa-groups" setPassword="false"/>
              </mapping>
            </approval>
          </workflow>
          <workflow name="request-graylog-reader" label="Graylog Read Only" description="View log data" inList="true" orgid="5d6bab30-9dd9-4cfd-8aff-a395fed377dc">
            <approval label="Approve Sudo All">
              <emailTemplate>
                                                ${givenName},
                                                You have open approvals
                                              </emailTemplate>
                <approvers>
                  <rule scope="group" constraint="cn=admin-approvers,cn=groups,ou=azure.cloud,o=Tremolo" />
                </approvers>
                <mapping strict="true">
                  <map>
                    <mapping targetAttributeName="uid" targetAttributeSource="uid" sourceType="user"/>
                  </map>
                  <addGroup name="graylog-reader" />
                  <provision sync="false" target="freeipa-groups" setPassword="false"/>
                </mapping>
              </approval>
            </workflow>
            <workflow name="request-graylog-admin" label="Graylog Administrator" description="Manage &amp; Configure Graylog" inList="true" orgid="5d6bab30-9dd9-4cfd-8aff-a395fed377dc">
              <approval label="Approve Graylog Administrator">
                <emailTemplate>
                                                  ${givenName},
                                                  You have open approvals
                                                </emailTemplate>
                  <approvers>
                    <rule scope="group" constraint="cn=admin-approvers,cn=groups,ou=azure.cloud,o=Tremolo" />
                  </approvers>
                  <mapping strict="true">
                    <map>
                      <mapping targetAttributeName="uid" targetAttributeSource="uid" sourceType="user"/>
                    </map>
                    <addGroup name="graylog-administrator" />
                    <provision sync="false" target="freeipa-groups" setPassword="false"/>
                  </mapping>
                </approval>
              </workflow>
              <workflow name="request-demogroup" label="Demo Group" description="Demo Things" inList="true" orgid="5d6bab30-9dd9-4cfd-8aff-a395fed377dc">
                <approval label="Approve Graylog Administrator">
                  <emailTemplate>
                                                    ${givenName},
                                                    You have open approvals
                                                  </emailTemplate>
                    <approvers>
                      <rule scope="group" constraint="cn=admin-approvers,cn=groups,ou=azure.cloud,o=Tremolo" />
                    </approvers>
                    <mapping strict="true">
                      <map>
                        <mapping targetAttributeName="uid" targetAttributeSource="uid" sourceType="user"/>
                      </map>
                      <addGroup name="demogroup" />
                      <provision sync="false" target="freeipa-groups" setPassword="false"/>
                    </mapping>
                  </approval>
                </workflow>
          <workflow name="reset-password-on-logout" label="Reset password on logout" description="Resets the user's password when the user is logged out" inList="false" orgid="687da09f-8ec1-48ac-b035-f2f182b9bd1e">
            <mapping strict="true">
              <map>
                <mapping targetAttributeName="uid" targetAttributeSource="uid" sourceType="user"/>
              </map>
              <customTask className="com.tremolosecurity.provisioning.customTasks.LoadGroups">
                <!-- The attribute name to search for on the user's account   -->
                <param name="nameAttr" value="uid"/>
                <!-- If set to true, only loads the groups from the virtual directory that the user's object is NOT already a member of               -->
                <param name="inverse" value="false"/>
              </customTask>
              <customTask className="com.tremolosecurity.provisioning.customTasks.LoadAttributes">
                <!-- An attribute name to load, case sensitive and can be listed multiple times       -->
                <param name="name" value="givenname"/>
                <param name="name" value="sn"/>
                <param name="name" value="mail"/>
                <param name="name" value="displayname"/>
                <param name="name" value="gecos"/>

                <param name="nameAttr" value="uid"/>
              </customTask>
              <customTask className="com.tremolosecurity.scalejs.token.password.SetRandomPassword">
                <param name="keyName" value="session-randompwd" />
                <param name="attributeName" value="carlicense" />
              </customTask>
              <provision sync="false" target="freeipa" setPassword="true"/>
            </mapping>

          </workflow>
        </workflows>
      </provisioning>
      </tremoloConfig>

```

You'll need to make AT LEAST the following updates:

1. Add your SQL Server information to the auditDB section
2. Add your queue information to the queue section for Azure Service Bus
3. Add credentials to the FreeIPA targets
4. Update host names

## Configure ADFS Integration

The final step was to configure OpenUnison and ADFS to talk.  We configured them using the command line tools described in the OpenUnison documentation.  

## Deployment

Finally, we're ready to build and deploy.  Go into the project folder and:

```
$ mvn clean
$ mvn package
$ rm -rf /path/to/tomcat/webapps/*
$ cp target/openunison-1.0-SNAPSHOT.war /path/to/tomcat/webapps/ROOT.war
$ /path/to/tomcat/bin/startup.sh
```

If everything goes according to plan, when you go to your root directory on openunison (ie https://openunison.azure.cloud/) you'll be redirected to ADFS to authenticate and then to ScaleJS just like in the demo video!

Tremolo Security
----------------
Useful links

* [Tremolo Security Home](https://www.tremolosecurity.com/)
* [Wiki Home](https://www.tremolosecurity.com/wiki/#!index.md)
* [Tremolo Security on Github](https://www.github.com/tremolosecurity/)
* Follow us on Twitter - [gimmick:twitterfollow](tremolosecurity) @tremolosecurity

