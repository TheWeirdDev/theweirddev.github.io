---
layout: post
title:  "Extract metadata from file names"
date:   2015-01-18 23:56:45
categories:
- blog
permalink: extract-metadata
description: We will se how to extract metadata from a formatted file name.
---

This post will cover the topic, how to use the file names to send metadata into Alfresco?

To illustrate the article I'll take an example. Let's imagine I have got a new project in my company, the medical
service wants to
 archive documents electronically. These documents concern persons and have a special type (certificate, 
 prescription, ...). We have
already have some data in our database, especially about the persons and the type of documents. In order to simplify 
the work of the medical 
service we decided to define a mechanism that uses a *files names* to retrieve metadata at
upload time. The idea is avoiding post-edition of documents.
 
 I'll go trough the following steps : 
 
1. File name pattern
2. content model
3. behaviour
4. extra - new behaviour with
 
## 1. File name pattern

First we need to define a pattern for our files names. In our database we have the tables **PERSON** and
**MEDICAL_DOCUMENT**. To create our 
pattern we will use the
person_id and the medical_document_code from these tables. We also need to specify the effective date of the document 
which is an
important information for our medical service.

Here the pattern we will use:

`personId;documentCode;effectiveDate`

>the person id is a number, document code a string and the effective date should respect the format dd-MM-yyyy

One example would be

`100;CERT;19-01-2015`

>In this example the document concerns the person with the id 100, it is a certificate made the 19th of January 2015.

## 2. Content model

In this second part I created a basic content model which gives the definition of our metadata according to the 
medical service requirement. 

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<!-- Medical service model -->
<model name="ms:medicalService" xmlns="http://www.alfresco.org/model/dictionary/1.0">
  <!-- Optional meta-data about the model -->
  <description>Medical service Content Model</description>
  <author>Sam</author>
  <version>1.0</version>

  <!-- Imports are required to allow references to definitions in other models -->
  <imports>
    <!-- Import Alfresco Dictionary Definitions -->
    <import uri="http://www.alfresco.org/model/dictionary/1.0" prefix="d"/>
    <!-- Import Alfresco Content Domain Model Definitions -->
    <import uri="http://www.alfresco.org/model/content/1.0" prefix="cm"/>
  </imports>

  <!-- Introduction of new namespaces defined by this model -->
  <namespaces>
    <namespace uri="http://www.someco.ch/model/medicalservice/1.0" prefix="ms"/>
  </namespaces>

  <!--		T Y P E   D E F I N I T I O N S		-->

  <types>
    <type name="ms:content">
      <title>medical service document</title>
      <parent>cm:content</parent>
      <mandatory-aspects>
        <aspect>ef:typed</aspect>
      </mandatory-aspects>
    </type>

    <type name="ms:folder">
      <title>medical service folder</title>
      <parent>cm:folder</parent>
      <mandatory-aspects>
        <aspect>ef:typed</aspect>
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
        <property name="ms:personStatus">
          <type>d:text</type>
        </property>
        <property name="ms:orgUnit">
          <type>d:text</type>
        </property>
      </properties>
    </aspect>

    <!-- ************************************************************************ -->
    <!-- Effective date aspect. Effective date of the document -->
    <!-- ************************************************************************ -->
    <aspect name="ms:effectiveDate">
      <title>Effective date</title>
      <properties>
        <property name="ms:date">
          <type>d:date</type>
          <mandatory>true</mandatory>
        </property>
      </properties>
    </aspect>

    <!-- ************************************************************************ -->
    <!-- The type of the document. E.g certificate, prescription, ... -->
    <!-- ************************************************************************ -->
    <aspect name="ms:documentType">
      <title>Type</title>
      <properties>
        <property name="ms:typeCode">
          <type>d:text</type>
          <mandatory>true</mandatory>
        </property>
        <property name="ms:typeName">
          <type>d:text</type>
          <mandatory>true</mandatory>
        </property>
        <property name="ms:typeDescription">
          <type>d:text</type>
        </property>
      </properties>
    </aspect>
  </aspects>
</model>
{% endhighlight %}

This model is quite self-explanatory in terms of content. I'd just like to
highlight 
why I created aspects to hold the properties instead of putting everything in `ms:content`. First of all, I find the
 aspects much more flexible. Indeed if you want to add properties you just add a new aspect and if you want to remove
  properties you just remove the aspect from the document. Then the aspect is crossed type. If I had put the
  information about the 
person directly in the ms:content I would not be able to use the same properties with another type (E.g. HR documents). 
In a real case it
 might better to have a common content model at the level of your organization which will hold the common aspects such
  as the person aspect.





