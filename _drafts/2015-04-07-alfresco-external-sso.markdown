---
layout: post
title:  "Alfresco, setup SSO using external subsystem"
date:   2016-04-10 23:56:45
comments: true
toc: true
categories:
- blog
permalink: alfresco-external-sso
description: An example how setup the external SSO mechanism.
---

We will see, in this post, an example of how to setup Alfresco with the external SSO subsystem.
Let's list important points here in order to clarify the task:

* In my case, Alfresco repository won't be accessed directly by users, instead they will use Share. 
* At the time a request reached Share, the user authentication has been already done by the external SSO component which acts as proxy. 
* Then, all we need is to configure Share to retrieve which user has been authenticated through SSO. 
* In my case the authenticated user will be taken from a specific header.

> __Information:__
>
> I won't cover here the configuration of the SSO itself, but only the part that concerns Alfresco. 
>
> __Config used:__
>
> > * Shibboleth, OpenSAML
> > * Alfresco Community 5.0.d
> > * Tomcat 7

## 1. Alfresco global properties 

The following configuration goes into the `alfresco-global.properties` located in the shared/classes folder on your tomcat installation.

### a. authentication chain

To set up our SSO we will need the external authentication subsystem. 
{% highlight properties %}
#
# The authentication chain
# -------------
authentication.chain=external1:external,ldap1:ldap-ad,alfrescoNtlm1:alfrescoNtlm
{% endhighlight %}

The order in the chain is important since it's a prioritized list. I put the external system as the first item in the list `external1:external`.
I left ldap and the default Alfresco authentication in the chain. If SSO fails the next authentication system will take over.

> If you have never touched the authentication chain by default you should have this one:
`authentication.chain=alfrescoNtlm1:alfrescoNtlm`. NTLM is the default Alfresco authentication system, it uses the users stored in Alfresco DB (username and password based). 
NTLM support SSO if you synchronise the users in Alfresco with your Windows domain. This is not compatible with my case. 

### b. ldap

I want to use ldap only to synchronise the authorities and not for authentication:
{% highlight properties %}
#
# Ldap settings
# -------------
ldap.authentication.active=false
ldap.synchronization.active=true
{% endhighlight %}

### c. external (SSO) configuration
{% highlight properties %}
#
# External authentication (SSO) settings
# -------------
external.authentication.enabled=true
external.authentication.defaultAdministratorUserNames=customadmin
external.authentication.proxyHeader=CUSTOM_HEADER
external.authentication.proxyUserName=
{% endhighlight %}

> * __enabled__: enable the external subsystem (true or false).
> * __defaultAdministratorUserNames__: The comma separated list of default admins (user name). I don't think it's possible to bypass SSO in order to log as admin. so it's better the select at least a default admin.
> * __proxyHeader__: the header that contains the user name of the authenticated user.
> * __proxyUserName__: this property has to be set when using SSL, so we leave it empty. 

## 2. Share config

I used the following config file `tomcat/shared/classes/alfresco/web-extension/share-config-custom.xml`.

In the section Remote (make sure you have only one section)
{% highlight xml %}
<config evaluator="string-compare" condition="Remote">
{% endhighlight %}

### a. Add connector

Create a new connector to specify the name of the header. It should be the same name as the one defined previously.
{% highlight xml %}
<connector>
    <id>alfrescoHeader</id>
    <name>Alfresco Connector</name>
    <description>Connects to an Alfresco instance using header-based authentication</description>
    <class>org.alfresco.web.site.servlet.SlingshotAlfrescoConnector</class>
    <userHeader>CUSTOM_HEADER</userHeader>
</connector>
{% endhighlight %}

### b. Add/change end-point

{% highlight xml %}
<endpoint>
    <id>alfresco</id>
    <name>Alfresco - user access</name>
    <description>Access to Alfresco Repository WebScripts that require user authentication</description>
    <connector-id>alfrescoHeader</connector-id>
    <endpoint-url>http://localhost:8080/alfresco/wcs</endpoint-url>
    <identity>user</identity>
    <external-auth>true</external-auth>
</endpoint>
{% endhighlight %}

* This end point will use the previous connector.
* identity is set to user
* external-auth

> This end point will be used by Share to talk to the Alfresco repository. I had many troubles to configure it with ssl.
But since, I have Alfresco and Share on the same tomcat they don't need to talk over ssl. More details are in the next chapter.

### c. Full config
{% highlight xml %}
   <config evaluator="string-compare" condition="Remote">
      <remote>
         <endpoint>
            <id>alfresco-noauth</id>
            <name>Alfresco - unauthenticated access</name>
            <description>Access to Alfresco Repository WebScripts that do not require authentication</description>
            <connector-id>alfresco</connector-id>
            <endpoint-url>http://localhost:8080/alfresco/s</endpoint-url>
            <identity>none</identity>
         </endpoint>
        
        <!-- not used now, just keep it in case -->
         <connector>
            <id>alfrescoCookie</id>
            <name>Alfresco Connector</name>
            <description>Connects to an Alfresco instance using cookie-based authentication</description>
            <class>org.alfresco.web.site.servlet.SlingshotAlfrescoConnector</class>
         </connector>

         <connector>
            <id>alfrescoHeader</id>
            <name>Alfresco Connector</name>
            <description>Connects to an Alfresco instance using header and cookie-based authentication</description>
            <class>org.alfresco.web.site.servlet.SlingshotAlfrescoConnector</class>
            <userHeader>CUSTOM_HEADER</userHeader>
         </connector>

         <endpoint>
            <id>alfresco</id>
            <name>Alfresco - user access</name>
            <description>Access to Alfresco Repository WebScripts that require user authentication</description>
            <connector-id>alfrescoHeader</connector-id>
             <endpoint-url>http://localhost:8080/alfresco/wcs</endpoint-url>
            <identity>user</identity>
            <external-auth>true</external-auth>
        </endpoint>

         <endpoint>
            <id>alfresco-feed</id>
            <name>Alfresco Feed</name>
            <description>Alfresco Feed - supports basic HTTP authentication via the EndPointProxyServlet</description>
            <connector-id>http</connector-id>
            <endpoint-url>http://localhost:8080/alfresco/s</endpoint-url>
            <basic-auth>true</basic-auth>
            <identity>user</identity>
         </endpoint>
      </remote>
   </config>
{% endhighlight %}

## 3. Tomcat

In the server.xml

### a. Open none-ssl connector

If not already exist, add connector without SSL. This is used only for Share to talk with Alfresco using localhost. Another connector is used with SSL for clients to connect to Share (and Alfresco if needed).
A firewall has been setup to close the port 8080 for the ouside.

{% highlight xml %}
<Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000"/>
{% endhighlight %}

### b. AJP connector

Disable the authentication for the AJP connector.

{% highlight xml %}
<Connector port="8009" protocol="AJP/1.3" redirectPort="8043" packetSize="65536" connectionTimeout="600000" tomcatAuthentication="false"/>
{% endhighlight %}



