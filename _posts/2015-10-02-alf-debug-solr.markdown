---
layout: post
title:  "Alfresco: how to debug Solr queries"
date:   2015-10-06 23:56:45
comments: true
categories:
- blog
permalink: alf-debug-solr
toc: true
description: A couple of tips about debugging Solr queries with Alfresco.
---

When you start investigating Solr and Alfresco it's not always easy to find a starting point. You might wondering,  where in the
code Alfresco
calls Solr? What are the queries that Alfresco sends to Solr? etc, etc.

In this post, I will drive you in the right direction in order for you to quickly focus on what matters (problem,
extension, and so on).

>Config:
>
> > * Alfresco SDK, All-In-One
> > * Alfresco Community 5.0.c
> > * Solr 4

## 1. Alfresco and Solr queries

It's important to highlight that
 the communication between Alfresco and Solr is not just Alfresco performing queries on Solr. Indeed, the
 communication can be about indexing new content, updating indexes, sending content model, and even more. In this post
 we will only
 talk
 about queries, it actually covers already a lot. For instance, a list of share components which use Solr queries:

 * page "People Finder"
 * the auto-complete box at the top right corner
 * the main search page (obviously)
 * many dashlets (E.g. "Site Content")
 * the page "Site Finder"
 * and others


## 2. SolrQueryHTTPClient

The class **SolrQueryHTTPClient** is one of the most important class when talking about "Alfresco sending queries to Solr". So the first tip is to change the
 logger level on this class, set it to DEBUG.

In your `log4j.properties`:
{% highlight js %}
log4j.logger.org.alfresco.repo.search.impl.solr.SolrQueryHTTPClient=debug
{% endhighlight %}

Once you've change the logger level you will see a lot of things in the logs. In fact, the logs show the Alfresco
queries. We can see it's split in two parts: **sent** and **with**. Let's see that in more details.

### a. Sent
The **sent** part looks like that:

{% highlight rest %}
/solr4/alfresco/afts?wt=json&fl=DBID,score&rows=251&df=keywords&start=0&locale=en_US...
{% endhighlight %}


It can be really long. This part is actually the url,  it's long because it can have a lot parameters especially
 for
 the facets. When I work on this part I usually use a tool to make it more readable. (The tool I use is a Chrome
 extension called "Advanced Rest Client". There are plenty of similar tools, I'm sure you will find something)

![Readable url]({{ site.url }}/assets/posts/solr-debug/readable-url.png)

Seen like that it's much easier to understand what's happening. Now, I'm going to explain really briefly some
parameters such that you can understand the overall query.

* **wt**: writer type, to tell Solr to return Json object as response (could be xml, other formats)
* **fl**: field listing, Solr will return these fields for each document. We can see Solr doesn't return much things
here just the DBID and the score (kind of select clause in a SQL query)
* **rows**: maximum number of rows returned
* **df**: default field, to tell the default field to search
* **fq**: filter query, used to filter, here it's used to filter for authorities and tenants, (kind of where clause
in a SQL query)
* **facet**: to allow facets
* **facet.field**: to add a new facet field
* **f.@{http://www.alfresco.org/model/content/1.0}modifier.facet.limit**: the maximum of hits for the field (on the
screenshot it's one of the params with 100 as value)
* **spellcheck**: to allow the spellcheck feature
* **spellcheck.q**: the query for the solr spellcheck component

There are more parameters, this is just a quick overview.

### b. With
The **with** part is a Json object so you can easily indent it:
{% highlight json linenos %}
{
  "queryConsistency": "DEFAULT",
  "textAttributes": [],
  "allAttributes": [],
  "templates": [
    {
      "template": "%(cm:name cm:title cm:description ia:whatEvent ia:descriptionEvent lnk:title\n  lnk:description TEXT TAG)",
      "name": "keywords"
    }
  ],
  "authorities": [
    "GROUP_EVERYONE",
    "GROUP_FOO_BAR",
    "ROLE_AUTHENTICATED",
    "foobar"
  ],
  "tenants": [""],
  "query": "(Foo  AND (+TYPE:\"cm:content\" OR +TYPE:\"cm:folder\")) AND -TYPE:\"cm:thumbnail\" AND
  -TYPE:\"cm:failedThumbnail\" AND -TYPE:\"cm:rating\" AND -TYPE:\"st:site\" AND -ASPECT:\"st:siteContainer\" AND -ASPECT:\"sys:hidden\" AND -cm:creator:system AND -QNAME:comment\\-*",
  "locales": ["en_US"],
  "defaultNamespace": "http://www.alfresco.org/model/content/1.0",
  "defaultFTSFieldOperator": "AND",
  "defaultFTSOperator": "AND",
  "anyDenyDenies": true
}
{% endhighlight %}

This Json object is sent in the body of the http post request. I would like to highlight here that the main query is
actually located in this Json object (line 18).
{% highlight sh%}
"query": "(Foo  AND (+TYPE:\"cm:content\" OR +TYPE:\"cm:folder\")) AND"...
{% endhighlight %}

To conclude, a search query is a http post request that contains parameters in the url and a Json object in the body
. All this information (parameters, json content) is used by Solr to perform the query. It's actually not a pure Solr
 query, indeed Alfresco extended Solr.

> In addition to set the logger to DEBUG, you can actually launch your application in debug mode and add breakpoints in this class. The method **postSolrQuery** is a good start.

## 3. Http client

It can be handy to run (re-run) "Alfresco-Solr" queries independently to Alfresco. To do so I use the same chrome
extension I already mentioned as http client. It's really easy to build the http request using the logs we've just
seen. Use the **sent** part to build the url by adding the host:
{% highlight rest %}
http://localhost:8080/solr4/alfresco/afts?wt=json&fl=DBID,score...
{% endhighlight %}
You have to make sure the http request is a POST request. Then copy the **with** path in the body of your request.
You have to set the "Content-Type" of the request to application/json.

If Solr is running, it should work.

![Http client]({{ site.url }}/assets/posts/solr-debug/http-client.png)

The response will be a Json object which could be quite big as well according to the type of query you are performing.
If you are searching for documents, the Json object will contain something like this (more or less):

{% highlight json %}
{
  "responseHeader": {
    "status": 0,
    "QTime": 114
  },
  "_original_parameters_": "many things...",
  "response": {
    "numFound": 2,
    "start": 0,
    "maxScore": 0.0075914357,
    "docs": [
      {
        "DBID": 972,
        "score": 0.0075914357
      },
      {
        "DBID": 984,
        "score": 0.0075914357
      }
    ]
  }
}
{% endhighlight %}

We can see number of documents found, and then the result set.

## 4. Search webscript

In the `alfresco/templates/webscripts/org/alfresco/slingshot/search` you will find quite a few interesting files.
You can find there how the Json object (**with** part) is built

 * search.lib.js (this file contains all the logic of building the query, cf "with" -> query)
 * search.get.config (if you want to change the default operator AND/OR or add your custom metadata in the search
  template, cf "with" -> template)
 * search.get.js

## 5. Solr Admin page

You can access the Solr Admin page by accessing this url:

{% highlight sh %}
http://localhost:8080/solr4/#/
{% endhighlight %}


I'm not going to give much information about this page here because it's not the purpose of this article. And then, you
 can
 try
yourself to play with this UI. In most cases, you will want to select as Solr Core (the dropdown list named **Core
Selector**) "Alfresco". Last thing, you should be aware that Alfresco extended Solr, it's not a vanilla installation of
Solr. They added request handlers, parsers, search components, and so on. The admin page might not always be designed
 for
"Alfresco-Solr extension". However behind the scene it remains Solr and if you want to know more how it works,
 it
really
 worth it to learn Solr.