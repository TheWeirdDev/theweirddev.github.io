---
layout: post
title:  "Alfresco: SSO with the external auth subsystem"
date:   2016-04-15 23:56:45
comments: true
toc: true
categories:
- blog
permalink: alfresco-external-sso
description: An example how to setup the external authentication subsystem with SSO.
---

This post will cover an example of how to setup SSO using the external authentication subsystem.
Before starting let's clarify a couple of points:

* In my case, Alfresco repository won't be accessed directly by users, instead they will use Share. 
* At the time a request reached Share, the user authentication has been already done by the SSO component which acts as a proxy. 
* Then, all we need is to configure Share to retrieve which user has been authenticated through SSO. 
* To do that, the authenticated user will be retrieved from a specific header.

> __Information:__
>
> I won't cover here the configuration of the SSO itself, but only the part that concerns Alfresco. 
>
> __Config used:__
>
> > * Shibboleth, SAML-based SSO
> > * Alfresco Community 5.0.c
> > * Tomcat 7

## 1. Alfresco global properties 

The following configuration goes into the `alfresco-global.properties` located in the __shared/classes__ folder on your tomcat installation. 

### a. authentication chain

To setup SSO, we need to configure the external authentication subsystem. The first step is to add it in the authentication chain.
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
external.authentication.defaultAdministratorUserNames=admin,samuelm
external.authentication.proxyHeader=CUSTOM_HEADER
external.authentication.proxyUserName=
{% endhighlight %}

> * __enabled__: enable the external subsystem (true or false).
> * __defaultAdministratorUserNames__: comma separated list of default admins (user name). I don't think it's possible to bypass SSO in order to log as admin. so it's better to select at least one value.  
> * __proxyHeader__: header name that contains the username of the authenticated user. A header is key/value, basically SSO will override each request adding the header, eg. CUSTOM_HEADER=samuelm.
> * __proxyUserName__: this property has to be set when using SSL, so we leave it empty. 
>
> Notice that this configuration can be moved in a separated file specific for this subsystem, which probably is a best practice. 

## 2. Share config

I used the following config file `tomcat/shared/classes/alfresco/web-extension/share-config-custom.xml`.

In the section Remote.
{% highlight xml %}
<config evaluator="string-compare" condition="Remote">
{% endhighlight %}

### a. Add connector

Create a new connector with the property __userHeader__ to define the name of the header. It should be the same name as in the external auth subsystem.
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

This end point will be used by Share to talk to the Alfresco repository. I had many troubles to configure it with ssl.
But since, I have Alfresco and Share on the same tomcat they don't need to talk over ssl. More details are in the next chapter.

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

> * __connector-id__: this end point will use the previous connector.
> * __endpoint-url__: the url to Alfresco repository, notice that "/wcs" is used instead of "/s".
> * __identity__: is set to user.
> * __external-auth__: true

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

In the `server.xml` of your tomcat installation.

### a. Open none-ssl connector

If it doesn't exist yet, add connector without SSL. This is used only for Share to talk with Alfresco using localhost. Another connector is used with SSL for clients to connect to Share (and Alfresco if needed).
A firewall has been setup to close the port 8080 from the outside.

{% highlight xml %}
<Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000"/>
{% endhighlight %}

### b. AJP connector

Disable the authentication for the AJP connector.

{% highlight xml %}
<Connector port="8009" protocol="AJP/1.3" redirectPort="8043" packetSize="65536" connectionTimeout="600000" tomcatAuthentication="false"/>
{% endhighlight %}



