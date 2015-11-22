---
layout: post
title:  "Java Http client and Json"
date:   2015-09-16 23:56:45
comments: true
categories:
- blog
permalink: http-client
description: Simple example of a Post request using http client from the Apache HTTP Components 4.5 (Java 7).
---

>Config:
>
> > * Java 7
> > * Maven 3.2.3
> > * HTTP Components 4.5

We will see how to use the http client from the Apache HTTP Components version 4.5 which is currently the last
version. In particular, we will build an HTTP post request to send a json object. Then, we will see a
way to read the response to get a POJO.

To illustrate this short article, let's imagine we have a web service which computes some access rights. The service
accept
post requests with Json content and return a json response. From a Java client we want to call
this web service, the idea is to have method which takes a POJO AccessRequest in parameter and then returns a POJO
AccessResponse.

>> AccessResponse getAccess(AccessRequest);

## Maven pom

{% highlight xml%}
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.myco</groupId>
  <artifactId>access-rights-client</artifactId>
  <packaging>jar</packaging>
  <version>1.0.0-SNAPSHOT</version>
  <name>access-rights-client</name>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <source>1.7</source>
          <target>1.7</target>
        </configuration>
      </plugin>
    </plugins>
  </build>

  <dependencies>
    <dependency>
      <groupId>com.google.code.gson</groupId>
      <artifactId>gson</artifactId>
      <version>2.3.1</version>
    </dependency>
    <dependency>
      <groupId>org.apache.httpcomponents</groupId>
      <artifactId>httpclient</artifactId>
      <version>4.5</version>
    </dependency>

    <!-- Test dependencies -->
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.12</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

</project>
{% endhighlight %}

## POJO

These classes will be jonsonify and used in the http request and response.

{% highlight java%}
public class AccessRequest implements Serializable
{
  private Long userId;
  private Long resourceId;

  public void setUserId(Long userId)
  {
    this.userId = userId;
  }

  public Long getUserId()
  {
    return userId;
  }

  public void setResourceId(Long resourceId)
  {
    this.resourceId = resourceId;
  }

  public Long getResourceId()
  {
    return resourceId;
  }

  @Override
  public String toString()
  {
    return "AccessRequest{" + userId + ", " + resourceId + '}';
  }
}
{% endhighlight %}

{% highlight java%}
public class AccessResponse implements Serializable
{
  private boolean access;
  private String reason;

  public boolean hasAccess()
  {
    return access;
  }

  public void setAccess(boolean access)
  {
    this.access = access;
  }

  public String getReason()
  {
    return reason;
  }

  public void setReason(String reason)
  {
    this.reason = reason;
  }

  @Override
  public String toString()
  {
    return "AccessResponse{" + access + ", " + reason + '}';
  }
}
{% endhighlight %}

## Http Client

{% highlight java%}
  // It's much easier to use here a ResponseHandler because it closes all the streams
  ResponseHandler<List<ReadAccess>> rHandlerAccessAll = new ResponseHandler<List<ReadAccess>>()
  {
    @Override
    public List<ReadAccess> handleResponse(final HttpResponse response) throws IOException
    {
      StatusLine statusLine = response.getStatusLine();
      HttpEntity entity = response.getEntity();

      if (statusLine.getStatusCode() >= 300)
        throw new HttpResponseException(statusLine.getStatusCode(), statusLine.getReasonPhrase());

      if (entity == null)
        throw new ClientProtocolException("Response contains no content");

      Type listOfReadAccesses = new TypeToken<List<ReadAccess>>()
      {
      }.getType();
      Gson gson = new GsonBuilder().create();
      return gson.fromJson(EntityUtils.toString(entity), listOfReadAccesses);
    }
  };
{% endhighlight %}

{% highlight java%}
  try(CloseableHttpClient httpClient = HttpClients.createDefault())
     {
       URI uri = new URIBuilder().setPath(accessRulesEndPoint + "/all").setParameter("username", userName).build();
       HttpGet get = new HttpGet(uri);
       return httpClient.execute(get, rHandlerAccessAll);
     }
{% endhighlight %}


