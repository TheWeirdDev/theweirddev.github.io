---
layout: post
title:  "Extract metadata from file names"
date:   2015-03-28 23:56:45
comments: true
categories:
- blog
permalink: extract-metadata
description: We will se how to extract metadata from a formatted file name.
---

This post covers the following topic. **How to use files name to send metadata into Alfresco?**

> __Before starting__
> 
> Since this post is the first of a series (I hope), we will go together through the steps needed to start a project that extends/customizes Alfresco. Our project will use the All-in-one Maven
archetype. A
minimum of
knowledge on
Alfresco is
required in order to understand the article. Don't worry there is nothing
complex and you can find the full implementation [here]({{ site.data.blog.urlExamples }}).
For 
those who don't have the
pre-requisite there are many tutorials which introduce to Alfresco.
 For instance I recommend the __Alfresco Developer Tutorials__ from Jeff Potts.
>
>This is my config:
>
> > * Alfresco Community 5.0.a
> > * Fedora 20
> > * JDK 7
> > * Maven 3.2.3
> > * Alfresco Maven SDK 2.0.0 - beta 4
> > * Intellij Idea 14
>

-----------------------------------

To help us in this task I will use a little story. Let's say we work for a fictional company MyCo (sorry about the
name!) and we just received a new project. The medical service of MyCo is full of 
enthusiasm about archiving their documents electronically. Actually everyday this service generates many paper 
documents. Most of the documents are about
employees. Examples of docs are medical certificates, prescriptions, medical visits and so on. Our DB specialist
certified us we have in
our database data which can be interesting for the project. Especially information about
 employees and types of medical documents.

Oh! I forgot to say at MyCo we use `Alfresco` ;).

Based on our experience with Alfresco we know that users are not really keen on fill in
properties for each document. And the success of this project will be partial if half of the properties are missing.
 In 
order to
simplify the work of the medical service and to assure having a maximum of metadata filled we decided to
implements a mechanism which uses *files name* to
retrieve metadata at
upload time. To be a little bit more accurate, we will provide into the files name things like primary keys which
will be used to retrieve full objects from our databases in order to complete the meta-data.

## 1. Content model

The first thing to do is to define the content model. In a document management system it is common to
attach properties to documents and also to classify documents into different types. All of these will allow us to
perform
very accurate treatments and searches.
For our study case, the Medical Service needs for a document to have
information about:


- The person concerned.
- Which kind of document it is.
- At date the document is/was effective.


 Defining the content model that is actually writing a specific XML file that describes the properties and
 types of documents.
 Let's take a look at this file next part.

 > In an advance use of Alfresco we can do much more complex things with the content model. Actually it can be a topic
 for a
 full
 article. However there are already enough good documentations on the web and there is no real need to enter into details here.

### a. Content model XML

I already edited this file after many meetings with the Med. Service discussing which meta-data are relevant for them.
You can find it below -
click the link
 to see the full file on
GitHub.

[/repo-amp/src/main/amp/config/alfresco/module/repo-amp/model/medicalServiceModel.xml]({{ site.data.blog.urlExtractMeta }}/repo-amp/src/main/amp/config/alfresco/module/repo-amp/model/medicalServiceModel.xml)
{% highlight xml%}
<!--		T Y P E   D E F I N I T I O N S		-->

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
       <property name="ms:docTypeCode">
         <type>d:text</type>
         <mandatory>true</mandatory>
       </property>
       <property name="ms:docTypeName">
         <type>d:text</type>
         <mandatory>true</mandatory>
       </property>
       <property name="ms:docTypeDescription">
         <type>d:text</type>
       </property>
     </properties>
   </aspect>
 </aspects>
{% endhighlight %}

This model is quite self-explanatory. As you can see there is one content type and three aspects:

- `ms:content` (type)
- `ms:person` (aspect)
- `ms:effectiveDate` (aspect)
- `ms:documentType` (aspect)

<br/>
I'd just 
like to
highlight
why I created aspects to hold the properties instead of adding properties in `ms:content`. First of all, I find the
aspects much more flexible. Indeed you can easily add or remove properties to a node by manipulating aspects. And 
then aspects are crossed type so
you 
can 
really reuse them.

>In a real world it
might be better to have a content model at the level of your organisation that holds the common aspects and types
for all your projects. To go further a good practice would be to have a **root type** and all other types would
inherit from it. (E.g. myco:content, myco:folder)

### b. Spring context

Once the content model XML created we have to tell Alfresco about it. For this we have to add a bean in the Spring 
context. For a matter of clarity let's create a dedicated context XML for our project `medical-service-context.xml` 
(or just rename the service-content.xml).
And add this bean:

[/repo-amp/src/main/amp/config/alfresco/module/repo-amp/context/medical-service-context.xml]({{ site.data.blog.urlExtractMeta }}/repo-amp/src/main/amp/config/alfresco/module/repo-amp/context/medical-service-context.xml)
{% highlight xml%}
  <!-- Registration of new models -->
  <bean id="${project.artifactId}_dictionaryBootstrap" parent="dictionaryModelBootstrap"
        depends-on="dictionaryBootstrap">
    <property name="models">
      <list>
        <value>alfresco/module/${project.artifactId}/model/medicalServiceModel.xml</value>
      </list>
    </property>
  </bean>
{% endhighlight %}

Do not forget to register our new context file in the module-context.xml.

[/repo-amp/src/main/amp/config/alfresco/module/repo-amp/module-context.xml]({{ site.data.blog.urlExtractMeta }}/repo-amp/src/main/amp/config/alfresco/module/repo-amp/module-context.xml)
{% highlight xml%}
<beans>
	<import resource="classpath:alfresco/module/${artifactId}/context/service-context.xml" />
	<import resource="classpath:alfresco/module/${artifactId}/context/medical-service-context.xml" />
</beans>
{% endhighlight %}

> The module-context.xml is imported by the application-context.xml which is the main spring context file in Alfresco.
Thus by importing medical-service-context.xml in the module-context.xml we will assure that our custom beans will be
integrated.

At this stage you can run Alfresco with the new content model however Share is not aware of it so you won't see any
differences in the UI. Don't worry this is the goal of the next part.

## 2. Share

So we want our content model to be visible in Share. The modifications will be done in the `share-amp`
folder. I will go through this
part quickly because it is not the crux of the issue.

### a. share custom config XML

To expose our content model through the UI Share we mainly need to modify one file, the well-named
`share-config-custom.xml`. Once again I let you discover by your own this file and anyway the sources are on GitHub.

[/share-amp/src/main/resources/META-INF/share-config-custom.xml]({{ site.data.blog.urlExtractMeta }}/share-amp/src/main/resources/META-INF/share-config-custom.xml)
{% highlight xml%}
<alfresco-config>

  <!-- Document Library config section : Display new aspects and new types-->
  <config evaluator="string-compare" condition="DocumentLibrary" replace="true">
    <aspects>
      <!-- Aspects that a user can see -->
      <visible>
        <aspect name="ms:person"/>
        <aspect name="ms:effectiveDate"/>
        <aspect name="ms:documentType"/>
      </visible>

      <!-- Aspects that a user can add. Same as "visible" if left empty -->
      <addable>
      </addable>

      <!-- Aspects that a user can remove. Same as "visible" if left empty -->
      <removeable>
      </removeable>
    </aspects>

    <types>
      <type name="cm:content">
        <subtype name="ms:document"/>
      </type>
      <type name="ms:document">
      </type>
    </types>

  </config>

  <!-- ################################ TYPES EXISTING NODES ######################## -->

  <config evaluator="node-type" condition="ms:document">
      <forms>
        <form>
          <field-visibility>
            <!-- cm properties -->
            <show id="cm:name" for-mode="view"/>
            <show id="cm:title" for-mode="view"/>
            <show id="size" for-mode="view"/>
            <show id="cm:created" for-mode="view"/>
            <show id="cm:creator" for-mode="view"/>
            <!-- aspect person -->
            <show id="ms:personId"/>
            <show id="ms:firstName" for-mode="view"/>
            <show id="ms:lastName" for-mode="view"/>
            <show id="ms:gender" for-mode="view"/>
            <show id="ms:age" for-mode="view"/>
            <show id="ms:jobTitle" for-mode="view"/>
            <!-- aspect effectiveDate -->
            <show id="ms:date" for-mode="view"/>
            <!-- aspect documentType -->
            <show id="ms:docTypeCode"/>
            <show id="ms:docTypeName" for-mode="view"/>
            <show id="ms:docTypeDescription" for-mode="view"/>
          </field-visibility>
          <appearance>
            <set id="medicalinfoset" appearance="fieldset" label-id="set.ms_medicalinfoset"/>
            <set id="commonpropertiesset" appearance="fieldset" label-id="set.ms_commonpropertiesset"/>

            <field id="ms:personId" label-id="prop.ms_personId" set="medicalinfoset"/>
            <field id="ms:firstName" label-id="prop.ms_firstName" set="medicalinfoset"/>
            <field id="ms:lastName" label-id="prop.ms_lastName" set="medicalinfoset"/>
            <field id="ms:gender" label-id="prop.ms_gender" set="medicalinfoset"/>
            <field id="ms:age" label-id="prop.ms_age" set="medicalinfoset"/>
            <field id="ms:jobTitle" label-id="prop.ms_jobTitle" set="medicalinfoset"/>
            <field id="ms:date" label-id="prop.ms_date" set="medicalinfoset"/>
            <field id="ms:docTypeCode" label-id="prop.ms_docTypeCode" set="medicalinfoset"/>
            <field id="ms:docTypeName" label-id="prop.ms_docTypeName" set="medicalinfoset"/>
            <field id="ms:docTypeDescription" label-id="prop.ms_docTypeDescription" set="medicalinfoset"/>

            <field id="cm:name" set="commonpropertiesset"/>
            <field id="cm:title" set="commonpropertiesset"/>
            <field id="size" set="commonpropertiesset"/>
            <field id="cm:created" set="commonpropertiesset"/>
            <field id="cm:creator" set="commonpropertiesset"/>
          </appearance>
        </form>
      </forms>
    </config>

</alfresco-config>
{% endhighlight %}

For those who are familiar with this file, it is pretty basic. The result should look like this:

![Properties]({{ site.url }}/assets/posts/extract-meta/result-properties.png)

### b. Messages

The `share-config-custom` usually comes with properties used for the labels in the UI.

[/share-amp/src/main/amp/config/alfresco/module/share-amp/messages/medicalService.properties]({{ site.data.blog.urlExtractMeta }}/share-amp/src/main/amp/config/alfresco/module/share-amp/messages/medicalService.properties)
{% highlight xml%}
#types
type.ms_document=Medical document

#aspects
aspect.ms_person=Person
aspect.ms_typeDocument=Medical document type
aspect.ms_effectiveDate=Effective date

#properties
prop.ms_personId=Person id
prop.ms_firstName=First name
prop.ms_lastName=Last name
prop.ms_date=Effective date
prop.ms_docTypeCode=Code
prop.ms_docTypeName=Type document
prop.ms_docTypeDescription=Description
prop.ms_gender=Gender
prop.ms_age=Age
prop.ms_jobTitle=Job title

#sets
set.ms_commonpropertiesset=General properties
set.ms_medicalinfoset=Medical information
{% endhighlight %}

Spring needs to know about this file.

[/share-amp/src/main/amp/config/alfresco/web-extension/medical-serivce-context.xml]({{ site.data.blog.urlExtractMeta }}/share-amp/src/main/amp/config/alfresco/web-extension/medical-serivce-context.xml)
{% highlight xml%}
<?xml version='1.0' encoding='UTF-8'?><!DOCTYPE beans PUBLIC '-//SPRING//DTD BEAN//EN'
  'http://www.springframework.org/dtd/spring-beans.dtd'>

<beans>
  <!-- Add medical service messages -->
  <bean id="${project.artifactId}_resources"
        class="org.springframework.extensions.surf.util.ResourceBundleBootstrapComponent">
    <property name="resourceBundles">
      <list>
        <value>alfresco.module.${project.artifactId}.messages.medicalService</value>
      </list>
    </property>
  </bean>
</beans>
{% endhighlight %}

And we are done with share.

## 3. File name pattern

Since we have a content model we can define the pattern for our files name. Let's say in our database we have
the tables **PERSON** and
**MEDICAL_DOCUMENT** which contain more or less the properties of our aspects. To create our
pattern we will use the
**person_id** and the **medical_document_code** primary keys of the tables. We also need to specify the effective date
 of 
the 
document 
which is an
important information for our medical service.

Here an example of pattern which looks reasonable:

`personId;documentCode;effectiveDate`

>The **personId** is a number, **documentCode** is a string and the **effectiveDate** should respect the format dd-MM-yyyy

Example of file name:

`100;CERT;19-01-2015`

>This document concerns the person with the id 100. It is a certificate made the 19th of January 2015.

To summarize, a person working in the Medical Service will follow this convention to name his
files such that once the files are uploaded in Alfresco all other data about the person and document type
will be retrieved from the database.

## 4. Repository - Let's structure our project

 We are almost ready to go with the heart of the task. But before coding like crazy we need to structure a
 bit our project.

### a. Constants for content model

This part is optional but can be considered as a good practice. The idea is to
create a Java Interface to list all "items" of our content model. It allows to reference them 
later instead of having them hard coded everywhere. You can distinguish two different kind of constants.
The first ones are just strings set with local items names. The second are Qnames associated to the items.

[/repo-amp/src/main/java/org/myco/medical/constant/MedicalServiceModel.java]({{ site.data.blog.urlExtractMeta }}/repo-amp/src/main/java/org/myco/medical/constant/MedicalServiceModel.java)

> QName represents the qualified name of a Repository item. In other words a QName is
unique identifier for a content model property or a type, aspect, ... QNames are built with the namespace uri and the
local name
of the item.

> These constants can be placed in a separated maven project in order to easily share them between projects.

### b. Business entities

A bit of OOP now. Such as with a typical Java project we have business entities and we need to represent them in our
project. Here POJOs will do the job. A new package is created
[/repo-amp/src/main/java/org/myco/medical/bean]({{ site.data.blog.urlExtractMeta}}/repo-amp/src/main/java/org/myco/medical/bean) containing the following
classes :

- [Person]({{ site.data.blog.urlExtractMeta}}/repo-amp/src/main/java/org/myco/medical/bean/Person.java)
- [DocumentType]({{ site.data.blog.urlExtractMeta}}/repo-amp/src/main/java/org/myco/medical/bean/DocumentType.java)
- [MedicalDocument]({{ site.data.blog.urlExtractMeta}}/repo-amp/src/main/java/org/myco/medical/bean/MedicalDocument.java)

### c. External master data access objects (DAO)

Next step would be to configure a data source for the external database. However I'm not going to create a
real
database just for it but instead I created fake DAOs with hardcoded
objects [/repo-amp/src/main/java/org/myco/medical/dao]({{ site.data.blog.urlExtractMeta}}/repo-amp/src/main/java/org/myco/medical/dao).

### d. Services

> Alfresco provides the [Java Foundation API Reference](https://wiki.alfresco.com/wiki/Java_Foundation_API) which is
 a set of Spring beans that provides services to access the capabilities of the Alfresco repository. Examples
 of services:
>
> > * NodeService
> > * TransactionService
> > * NamespaceService
> > * ContentService
> > * SearchService
> > * ActionService
>
 
All these services are available with Spring dependency injections. For our case, we will obviously use these 
services but to go a bit further we will create a custom API. It will be an
additional layer on top of Alfresco API.
  
Let's start with the NodeService. The NodeService allows operations at the node level. For instance to get or set 
properties to a node,  to add aspects, remove aspects, and so on. All of these methods are a bit low level for us. In
 fact it would be easier to work directly with our business entities. E.g. `setPersonAspect(nodeRef, person)`.
 
Remember the person aspect : 

{% highlight xml%}
<aspect name="ms:person">
  <title>Patient</title>
  <properties>
    <!-- The Person ID of the related person -->
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
{% endhighlight %}

And the Person POJO: 

{% highlight java%}
public class Person
{
   private Long id;
   private String firstName;
   private String lastName;
   private String gender;
   private String jobTitle;
   private Integer age;
  
  // getters and setters ...
}
{% endhighlight %}

So let's create the custom service `MedicalNodeService`:

[/repo-amp/src/main/java/org/myco/medical/service/MedicalNodeService.java]({{ site.data.blog.urlExtractMeta }}/repo-amp/src/main/java/org/myco/medical/service/MedicalNodeService.java)
{% highlight java%}
/**
 * set the aspect ms:person to the node.
 *
 * @param nodeRef ref of the node.
 * @param person  properties for the aspect.
 */
public void setPersonAspect(NodeRef nodeRef, Person person)
{
  Map<QName, Serializable> aspectValues = Maps.newHashMap();

  aspectValues.put(QNAME_PROP_PERSON_ID, person.getId());
  aspectValues.put(QNAME_PROP_FIRST_NAME, person.getFirstName());
  aspectValues.put(QNAME_PROP_LAST_NAME, person.getLastName());
  aspectValues.put(QNAME_PROP_STATUS, person.getStatus());
  aspectValues.put(QNAME_PROP_ORG_UNIT, person.getOrgUnit());
  nodeService.addAspect(nodeRef, QNAME_ASPECT_PERSON, aspectValues);

  logger.debug("Set person aspect. Node id: " + nodeRef.getId() + ", person id: " + person.getId());
}
{% endhighlight %}


> As you can noticed we used Spring autowiring to get Foundation API beans into our class.

We will create more services like this later.

### e. Spring annotations

With our custom services it is easy to use Spring annotations. For example, by using the annotation `Service` on a
class
Spring we
 can declare a spring bean.
{% highlight java%}
/**
 * Medical node service
 * Additional layer on top of alfresco NodeService
 */
@Service
public class MedicalNodeService implements MedicalServiceModel
{
{% endhighlight %}

However we have to tell spring to allow these annotations and which packages to scan:
[/repo-amp/src/main/amp/config/alfresco/module/repo-amp/context/medical-service-context.xml]({{ site.data.blog.urlExtractMeta }}/repo-amp/src/main/amp/config/alfresco/module/repo-amp/context/medical-service-context.xml)
{% highlight xml%}
<?xml version='1.0' encoding='UTF-8'?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
                           http://www.springframework.org/schema/context
                           http://www.springframework.org/schema/context/spring-context.xsd">
  <context:annotation-config/>

  <context:component-scan base-package="org.myco.medical"/>
</beans>
{% endhighlight %}
 
## 5. Repository - extract metadata
 
 Finally we can start with the most interesting part, I mean the extraction of the meta-dada from the file names.
 Concretely the idea is to create a custom
 `action` which we will be used in a `rule`...
 
### a. Custom action

> Action: in Alfresco is a unit of work executed upon a node. In practice it is a Java class that extends
`ActionExecuterAbstractBase` and overrides the `executeImpl` method. Actions can be called:
>
> > - From a rule
> > - From the ActionService (Foundation API).
> > - Or even can be configured to be call from the UI in share. 

> Rule: is applied to a `folder` which determines the scope of the rule. Only nodes inside the folder or nodes that
enters the folder will be concerned  (rules can be applied to sub-folders as well). Then a rule executes an `action`
upon the nodes under certain `conditions`. The condition is important, it is like a filter to select upon which
nodes we want to execute the action. For example, new nodes, newly updated nodes, with a specific type or aspect, ...

In our case we will call the action from a rule applied to a folder. Let's start creating a new
class in a new package.

[/repo-amp/src/main/java/org/myco/medical/action]({{ site.data.blog.urlExtractMeta }}/repo-amp/src/main/java/org/myco/medical/action)

{% highlight java%}
public class ExtractMetadataFromFileName extends ActionExecuterAbstractBase{
  @Override
  protected void executeImpl(Action action, NodeRef actionedUponNodeRef)
  {
      // TODO fill in implementation
  }

  @Override
  protected void addParameterDefinitions(List<ParameterDefinition> paramList)
  {
      // TODO fill in action parameter definitions
  }
}
{% endhighlight %}

The `addParameterDefinitions` method is used to pass additional variables to the action, we can leave it empty for our
case. Let's focus on the `executeImpl` method which will contain the logic
of our feature. The parameter `actionedUponNodeRef` will reference the medical documents.

The strategy is:

- Read the node name.
- Parse the name in order to extract personId, documentCode and effectiveDate.
- Call DAOs to retrieve the full person and document type.
- Apply the type ms:document to the node.
- Set all medical properties (metadata).

If we translate this into Java it might look something like this
{% highlight java%}
// get file name from nodeRef
String fileName = medicalNodeService.getName(actionedUponNodeRef);
// parse file name and return documentIdentifier
DocumentIdentifiers docIds = medicalFileNameService.parseFormattedFileName(fileName);
// build complete medical document from the identifiers - call DAOs
MedicalDocument medicalDocument = medicalDocumentService.buildMedicalDocumentFromIdentifiers(docIds);
// apply medical document type and set all properties
medicalDocumentService.createMedicalDocumentFromExistingNode(actionedUponNodeRef, medicalDocument);
{% endhighlight %}


The DocumentIdentifiers is a simple POJO that represents the file name parsed.

[/repo-amp/src/main/java/org/myco/medical/bean/DocumentIdentifiers.java]({{ site.data.blog.urlExtractMeta }}/repo-amp/src/main/java/org/myco/medical/bean/DocumentIdentifiers.java)
{% highlight java%}
public class DocumentIdentifiers
{
  private Long personId;
  private String documentTypeCode;
  private Date effectiveDate;
  private String fileName;

  // getters and setters ...
}
{% endhighlight %}

Because the logic is a bit too complex for one method it's clearer to split it. And to follow our good practices we 
will create new services. So I am introducing to you the `medicalDocumentService` and the`medicalFileNameService` but
 we will come back it in a moment.


One thing I haven't mentioned yet - we have to tell Alfresco about this new action. For this we
 have to declare a new bean in our [medical-service-context.xml]({{ site.data.blog.urlExtractMeta }}/repo-amp/src/main/amp/config/alfresco/module/repo-amp/context/medical-service-context.xml)

{% highlight xml%}
<!--medical actions-->
<bean id="extract-metadata-from-file-name" class="org.myco.medical.action.ExtractMetadataFromFileNameAction" 
parent="action-executer"/>
{% endhighlight %}

And finally we need to run this into a transaction. Thanks to the Alfresco Foundation API and the
TransactionService:

{% highlight java%}
transactionService.getRetryingTransactionHelper().doInTransaction(new RetryingTransactionHelper.RetryingTransactionCallback<Void>()
{
  public Void execute() throws Throwable
  {
    // code to execute in a transaction
    return null;
  }
});
{% endhighlight %}

The full action is here: [ExtractMetadataFromFileNameAction.java]({{ site.data.blog.urlExtractMeta }}/repo-amp/src/main/java/org/myco/medical/action/ExtractMetadataFromFileNameAction.java)

### b. MedicalFileNameService

I have introduced a new medical service. Indeed, since we start to have some treatments around file
names I found nice to have a service dedicated.

Basically the main method of this service is to parse a file name and return the equivalent DocumentIdentifiers.

[/repo-amp/src/main/java/org/myco/medical/service/MedicalFileNameService.java]({{ site.data.blog.urlExtractMeta }}/repo-amp/src/main/java/org/myco/medical/service/MedicalFileNameService.java)
{% highlight java%}
public DocumentIdentifiers parseFormattedFileName(String formattedFileName)
  {
    ParameterCheck.mandatory("formattedFileName", formattedFileName);

    DocumentIdentifiers documentIdentifiers = new DocumentIdentifiers();

    String[] tokens = formattedFileName.split(SEPARATOR);

    if (tokens.length != 3)
    {
      logger.info("The file name [" + formattedFileName + "] does not respect the format: " + EXPECTED_FORMAT);
      return documentIdentifiers;
    }

    setPersonId(tokens[0], documentIdentifiers);
    setDocumentTypeCode(tokens[1], documentIdentifiers);
    setEffectiveDate(tokens[2], documentIdentifiers);
    documentIdentifiers.setFileName(getGenerateName());

    return documentIdentifiers;
  }

  private void setDocumentTypeCode(String token, DocumentIdentifiers documentIdentifiers)
  {
    documentIdentifiers.setDocumentTypeCode(token);
  }

  private void setEffectiveDate(String token, DocumentIdentifiers documentIdentifiers)
  {
    try
    {
      documentIdentifiers.setEffectiveDate(DATE_FORMAT.parse(token));
    }
    catch (ParseException e)
    {
      logger.debug("Wrong date format: " + token + "Expected " + STRING_DATE_FORMAT, e);
    }
  }

  private void setPersonId(String token, DocumentIdentifiers documentIdentifiers)
  {
    try
    {
      documentIdentifiers.setPersonId(Long.valueOf(token));
    }
    catch (NumberFormatException e)
    {
      logger.debug("Wrong person id format: " + token, e);
    }
  }
{% endhighlight %}

This is a really simple example and in a real case should be improved. At least handle  non formatted file names.

### c. MedicalDocumentService

It's the second service I've introduced in our Action. This new service contains two methods:
The first one `buildMedicalDocumentFromIdentifiers` uses a documentIdentifier to retrieve business objects and
build an object [MedicalDocument]({{ site.data.blog.urlExtractMeta}}/repo-amp/src/main/java/org/myco/medical/bean/MedicalDocument.java). The second method
`createMedicalDocumentFromExistingNode` sets the type ms:content and all medical properties to an existing node.

[/repo-amp/src/main/java/org/myco/medical/service/MedicalDocumentService.java]({{ site.data.blog.urlExtractMeta }}/repo-amp/src/main/java/org/myco/medical/service/MedicalDocumentService.java)
{% highlight java%}
@Service
public class MedicalDocumentService
{

  private Logger logger = Logger.getLogger(MedicalDocumentService.class);

  // Medical service API
  @Autowired
  private MedicalPersonDao medicalPersonDao;
  @Autowired
  private MedicalDocumentTypeDao medicalDocumentTypeDao;
  @Autowired
  private MedicalNodeService medicalNodeService;

  /**
   * Builds a MedicalDocument from a DocumentIdentifier. Retrieves all data from DAO.
   * @param documentIdentifiers
   * @return medical document corresponding to the identifiers
   */
  public MedicalDocument buildMedicalDocumentFromIdentifiers(DocumentIdentifiers documentIdentifiers)
  {
    MedicalDocument medicalDocument = new MedicalDocument();
    medicalDocument.setPerson(medicalPersonDao.getPerson(documentIdentifiers.getPersonId()));
    medicalDocument.setDocumentType(medicalDocumentTypeDao.getDocumentType(documentIdentifiers.getDocumentTypeCode()));
    medicalDocument.setName(documentIdentifiers.getFileName());
    medicalDocument.setEffectiveDate(documentIdentifiers.getEffectiveDate());
    return medicalDocument;
  }

  /**
   * Creates a medical document from an existing node.
   *
   * @param nodeRef ref of the node.
   * @param medicalDocument object used to set the properties.
   */
  public void createMedicalDocumentFromExistingNode(NodeRef nodeRef, MedicalDocument medicalDocument)
  {
    medicalNodeService.setMedicalDocumentType(nodeRef);
    medicalNodeService.setPersonAspect(nodeRef, medicalDocument.getPerson());
    medicalNodeService.setDocumentTypeAspect(nodeRef, medicalDocument.getDocumentType());
    medicalNodeService.setEffectiveDateAspect(nodeRef, medicalDocument.getEffectiveDate());
    medicalNodeService.setName(nodeRef, medicalDocument.getName());
  }
}
{% endhighlight %}

## 6. The Rule

Let's see how to manually create the rule using Share. First create a new folder somewhere in Alfresco.
![New folder]({{ site.url }}/assets/posts/extract-meta/new-folder.png)

Then on this folder click on manage rules.
![Manage rules]({{ site.url }}/assets/posts/extract-meta/manage-rules.png)

Create the new rule like this: 
![Manage rules]({{ site.url }}/assets/posts/extract-meta/new-rule.png)

Let's try to upload a document with the name `1234;PR;19-01-2015.pdf`
![Manage rules]({{ site.url }}/assets/posts/extract-meta/result-properties.png)

>For your information, it's possible to create a rule and applying it to a folder via the RuleService (Alfresco
foundation
API).

## 7. Unit tests

Proper unit tests should be developed for this feature. If I have time I will add this part.

## Conclusion
The feature we implemented today is very basic but we saw much more than just this feature.
We actually saw how to structure All-in-one. How to create a custom content model. How to create custom services. How
 to create Action. How to use Alfresco Foundation API etc.


To finish, it's not obvious the mechanism of file names that respect a certain convention to get metadata into Alfresco
 is the better way for end users. Indeed it should be use only with "power users" for other population of users
 advance
  UI components in the edit metadata screen would be better. I mean by advance components something like a suggest
  field that can suggest persons in your database while you are writing.
