---
layout: post
title:  "Alfresco create patch"
date:   2016-03-14 02:00:45
comments: true
toc: false
categories:
- blog
permalink: alf-simple-patch
description: How to create a simple Alfresco patch.
---

When extending Alfresco having to write a patch is rather a common use case. Either you've changed your content model so you have to update a
certain volume of documents. Or you want to apply new permissions, create new authorities, and so on. There might be a multitude of reasons for creating
a patch. But in fact, what exactly is a patch?

> Well, it's pretty simple. It's a piece of work you want to run __once__. Usually after a deploy (version/fix). The idea is to update  __the existing persistent items__ (Database, Solr, files system).

And really, that's all. It can be done by a simple web script, however, web
scripts can be run multiple times, and you still have to run it manually. In fact, Alfresco provides a component for that specific case.
 And it does more than running a piece of work once during start up. I find it really handy. so let's see how it works.

## Use Case

Let's create a simple patch that will create a new authority zone. 

## Java class

`createCustomZonePatch.java`
{% highlight java%}
public class CreateCustomAuthorityZonePatch extends AbstractPatch
{
  private AuthorityService authorityService;

  @Override
  protected String applyInternal() throws Exception
  {
    authorityService.getOrCreateZone(CUSTOM_AUTHORITY_ZONE);
    return "Patch xxxx Create Custom authority zone finished";
  }

  public void setAuthorityService(AuthorityService authorityService)
  {
    this.authorityService = authorityService;
  }
}
{% endhighlight %}

> Simply extends AbstractPath and overrides the method applyInternal.

## Spring context and properties

`custom-context.xml`
{% highlight xml%}
  <bean id="patch.custom.createCustomAuthorityZone" class="com.custom.patch.CreateCustomAuthorityZonePatch" parent="basePatch">
    <property name="id" value="patch.custom.createCustomAuthorityZone"/>
    <property name="description" value="patch.custom.createCustomAuthorityZone.description"/>
    <property name="fixesFromSchema" value="0"/>
    <property name="fixesToSchema" value="${version.schema}"/>
    <property name="targetSchema" value="10000"/>
    <property name="authorityService" ref="AuthorityService"/>
  </bean>
{% endhighlight %}


`i18n properties`
{% highlight yml%}
patch.custom.createCustomAuthorityZone=Create custom authority zone
patch.custom.createCustomAuthorityZone.description=Patch description Create Custom authority zone
{% endhighlight %}

> The patch can be allowed only for a specific version this is why there are properties from/to/target. The values in my example are the defaults when you don't want to specify a version. 
 Thus, the patch will run next time Alfresco repository will start.

## Additional information

The code will be ran in a __transaction__ as __System__ user. The result will be saved in the database in the table __ALF_APPLIED_PATCHES__.

`SQL`
{% highlight sql%}
select 
    * 
from 
    alf_applied_patch 
where 
    id = 'patch.custom.createCustomAuthorityZone'
{% endhighlight %}

`Result`
{% highlight java%}
id: "patch.custom.createCustomAuthorityZone"
description: "Patch description Create Custom authority zone"
fixes_from_schema: 0
fixes_to_schema: 8009
applied_to_schema: 8009
target_schema: 10000
applied_on_date: "2016-03-09 10:38:49.58"
applied_to_server: "5.0.0 (c r91299-b145) - Community"
was_executed: t
succeeded: t
report: "Patch xxxx Create Custom authority zone finished"
{% endhighlight %}