---
layout: post
title:  "CMIS and aspects in Alfresco"
date:   2015-06-08 23:56:45
comments: true
categories:
- blog
permalink: cmis-and-aspects
description: How to select aspects properties from CMIS.
---

A very short reminder how to use aspects in CMIS 1.1.

Let's use a simple snippet of content model I already used in a previous [post](http://smasue.github
.io/extract-metadata/). Briefly, to refresh our mind,
there is
 a type `ms:document` that inherits from `cm:content` and represents medical documents. Medical documents are linked
 to a
  person using and aspect `ms:person` that holds the person properties. Below is the xml that describes this content
  model, the full file is [here]({{ site.data.blog.urlExtractMeta}}/repo-amp/src/main/amp/config/alfresco/module/repo-amp/model/medicalServiceModel.xml):
{% highlight xml%}
<types>
   <type name="ms:document">
     <title>Medical service document</title>
     <parent>cm:content</parent>
     <mandatory-aspects>
       <aspect>ms:documentType</aspect>
       <aspect>ms:person</aspect>
       <aspect>ms:effectiveDate</aspect>
     </mandatory-aspects>
   </type>
 </types>

 <!--		A S P E C T    D E F I N I T I O N S		-->

 <aspects>
   <!-- ************************************************************************ -->
   <!-- Person aspect -->
   <!-- ************************************************************************ -->
   <aspect name="ms:person">
     <title>Patient</title>
     <properties>
       <!-- The Person ID of the related person -->
       <property name="ms:personId">
         <type>d:long</type>
         <mandatory>true</mandatory>
       </property>
       <property name="ms:firstName">
         <type>d:text</type>
       </property>
       <property name="ms:lastName">
         <type>d:text</type>
       </property>
       <property name="ms:gender">
         <type>d:text</type>
       </property>
       <property name="ms:age">
         <type>d:int</type>
       </property>
       <property name="ms:jobTitle">
         <type>d:text</type>
       </property>
     </properties>
   </aspect>

   ...
{% endhighlight %}


What we want is to select all medical documents for a certain person and show the properties. We want to select the
person based on his ID.
{% highlight sql%}
SELECT *
FROM ms:document AS d JOIN ms:person as p ON d.cmis:objectId = p.cmis:objectId
WHERE p.ms:personId = 11111
{% endhighlight %}

As you can see the there is a join between the type and the aspect - the aspect is actually considered as a separated
object. Then the join is made on the property `cmis:objectId`. That's all!



