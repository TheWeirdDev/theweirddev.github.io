---
layout: post
title:  "Extract metadata from file names"
date:   2015-01-18 23:56:45
categories:
- blog
permalink: extract-metadata
description: We will se how to extract metadata from a formatted file name.
---

This post covers the following topic. **How to use documents file name to send metadata into Alfresco?**

> Before starting:
> 
> A minimum of knowledge about Alfresco is required to follow the article. However there is nothing
complex and you can find the full implementation in an allinone project [here]({{ site.data.blog.urlExamples }}).
For 
those who don't have the
pre-requisite there are many tutorials which introduce to Alfresco.
 For instance I recommend the __Alfresco Developer Tutorials__ from Jeff Potts. For this tutorial I used the config 
 below:
>
> > * Alfresco Community 5.0.a
> > * Fedora 20
> > * JDK 7
> > * Maven 3.2.3
> > * Alfresco Maven SDK 2.0.0 - beta 4
> > * Intellij Idea 14
>

-----------------------------------

To help us in this task I will use an illustration. Let's say we work for a fictional company MyCo (sorry about the 
name!) and we just received a new project. The medical service of MyCo is full of 
enthusiasm about archiving their documents electronically. Actually everyday this service generates many paper 
documents. Most of the documents are about
employees. Examples of docs are medical certificates, prescriptions, medical visit and so on. Our DB specialist 
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
upload time.

## 1. Content model

First of all we need to define the content model. The medical service needs to have information for each document 
about the person concerned, the type of the document and the effective date. To start we will create a new
XML file which is a description of the content model. Let's take a look at it now.

### a. Content model XML

I have done the work for you. 

>This is one solution but of course it can be discussed and other implementations are 
possible.

[/repo-amp/src/main/amp/config/alfresco/module/repo-amp/model/medicalServiceModel.xml](https://github.com/smasue/blog-examples/blob/master/extract-metadata/repo-amp/src/main/amp/config/alfresco/module/repo-amp/model/medicalServiceModel.xml)
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
       <property name="ms:status">
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

This model is quite self-explanatory. We can see there are one content type and three aspects:

- `ms:content` (type)
- `ms:person` (aspect)
- `ms:effectiveDate` (aspect)
- `ms:documentType` (aspect)

<br/>
I'd just 
like to
highlight
why I created aspects to hold the properties instead of adding properties in `ms:content`. First of all, I find the
aspects much more flexible. Indeed you can easily add or remove properties to a node by manipulating aspects. And then
 an 
aspect is crossed type so 
you 
can 
really reuse it.

>In a real world it
might be better to have a content model at the level of your organisation that holds the generic aspects and types 
for all your projects. To go further a good practice would be to have one let's say **root type**
 for your organisation and all other types will inherit from it. This abstract doesn't even need any properties.

### b Spring context

Once the content model XML created we have to tell Alfresco about it. For this we have to add a bean in the Spring 
context. For a matter of clarity let's create a dedicated context XML for our project `medical-service-context.xml` 
(or just rename the service-content.xml).
And add this bean
bean:

[/repo-amp/src/main/amp/config/alfresco/module/repo-amp/context/medical-service-context.xml](https://github.com/smasue/blog-examples/blob/master/extract-metadata/repo-amp/src/main/amp/config/alfresco/module/repo-amp/context/medical-service-context.xml)
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

[/repo-amp/src/main/amp/config/alfresco/module/repo-amp/module-context.xml](https://github.com/smasue/blog-examples/blob/master/extract-metadata/repo-amp/src/main/amp/config/alfresco/module/repo-amp/module-context.xml)
{% highlight xml%}
<beans>
	<import resource="classpath:alfresco/module/${artifactId}/context/service-context.xml" />
	<import resource="classpath:alfresco/module/${artifactId}/context/medical-service-context.xml" />
</beans>
{% endhighlight %}

> The module-context.xml is used to define beans for the AMP module. Actually this context will
be imported by the application-context.xml. At this stage you can run Alfresco with the new content model however 
Share is not aware of it. We will modify it soon.

## 2. File name pattern

Now we have a content model we can define the pattern for our files names. In our database we have
the tables **PERSON** and
**MEDICAL_DOCUMENT** which contain more or less the properties of our aspects. To create our
pattern we will use the
**person_id** and the **medical_document_code** primary keys of the tables. We also need to specify the effective date
 of 
the 
document 
which is an
important information for our medical service.

Below one pattern which looks reasonable:

`personId;documentCode;effectiveDate`

>The **personId** is a number, **documentCode** is a string and the **effectiveDate** should respect the format dd-MM-yyyy

An example of file name:

`100;CERT;19-01-2015`

>This document concerns the person with the id 100. It is a certificate made the 19th of January 2015.

### 3. Share and content model

Let's go back to our content model. The modifications will be done in the `share-amp` folder. I will go through this 
part quickly because it is not the heart of the problem.

### a. share custom config XML

We need now to expose our model in the UI Share. Indeed share needs to know about the our model to allow the user to
select our types and aspects from the UI.


[/share-amp/src/main/resources/META-INF/share-config-custom.xml](https://github.com/smasue/blog-examples/blob/master/extract-metadata/share-amp/src/main/resources/META-INF/share-config-custom.xml)
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

For those who are familiar we this file, there is nothing really special here. 

### b. Messages

`medicalService.properties`

The `share-config-custom` usually comes the properties.

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

#sets
set.ms_commonpropertiesset=General properties
set.ms_medicalinfoset=Medical information
{% endhighlight %}

And then we need to tell Spring to integrate this file 

`medical-service-context.xml`

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

## 3. Repository - Let's structure our project

 But first of all we need to structure a bit our project. 

### a. Constants for content model

This part is not mandatory but can it be considered as a good practice. The idea is tp 
create a Java Interface to list (creating constants) all "items" of our content model. It allows to reference them 
latter. And it is even possible to make a simple Maven project holding this Interface to externalize and share it.
You can create a package java/org/myco/medical/constant and then the interface `medicalServiceModel`. Here the final 
file. You can distinguish two different kind of constants. The first ones are just String we all items name. The 
second are Qnames corresponding to the items.

> QName represents the qualified name of a Repository item (cf. alfresco javadoc LINK). Which means a QName is a 
unique identifier for a property or a type, aspect and so on. QName is built for the namespaceURI and the local name 
of the item.

You can find this file here. 

#### b. Business entities 

A bit of OOP now with the business model. In our case we will work with persons and document types the information we
 have in our database.  .Let's
create a package 
under 
java/org/myco/medical/bean and the following 
classes :

- Person 
- DocumentType 
- MedicalDocument

Really nothing special!  just basic Java.

### c. External master data access

Next step is to configure a data source for our external database that contains data about persons and medical 
document types. Once the data source configured we will need DAO or to use ORM. 

For this article I only created fake DAO with hardcoded object. This part is not purely Alfresco dev. So I 
will skip it. The Fake DAOs are in the foler java/org/myco/medical/dao.

### b. Services

Alfresco provides the `Java Foundation API Reference` (https://wiki.alfresco.com/wiki/Java_Foundation_API). It is 
basically a set a spring beans that provide services to access the capabilities of the Alfresco repository. E.g. you 
can find the - 
 - NodeService
 - TransactionService
 - NamespaceService
 - ContentService
 - SearchService
 - ActionService
 
 etc. etc. 
 
 All these services are available throw spring dependency injections. For our case, we will use these services but as
  a best practice we will create a "Medical API", some services for the medical documents. It will an additional 
  layer on top of Alfresco API.
  
Let's start with the NodeService. The NodeService allows operations at the node level. For example accesser for 
properties. Add aspect. Remove aspect and so on. The idea for our `MedicalNodeService` is to add a layer. Let's take 
an example: in the NodeService you have the method addAspect(nodeRef, aspectTypeQName, aspectProperties) which is a bit low level for 
us. A kind of method we will create in service will be like setPersonAspect(nodeRef, person)
person aspect : 

{% highlight xml%}
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
    <property name="ms:status">
      <type>d:text</type>
    </property>
    <property name="ms:orgUnit">
      <type>d:text</type>
    </property>
  </properties>
</aspect>
{% endhighlight %}

and Person POJO is 

{% highlight java%}
public class Person
{
  private Long id;
  private String firstName;
  private String lastName;
  private String status; 
  private String orgUnit;
  
  // getters and setters ...
}
{% endhighlight %}

This abstraction will allow us to link the content model the business classes and the operations to make in Alfresco.
 Obvisously we are not going to create all services but only the ones we will use. 

For this exercice we will create a new package java/org/myco/medical/service and create the `MedicalNodeService.java`. 
The full class is here. As you can notice we use the NodeService from Alfresco Foundation API. We use Spring 
autowiring to retrieve this bean into our class. 

### c. Spring context

To use our services we will declare our beans into the Spring context. The 
`module-context.xml` is launched by Alfresco when starting. Indeed, if you take a look to the `application-context
.xml` we can notice the import to all `module-context.xml`. This file is used to declare and override module specific
 files. If you open it we will see another import `service-context.xml`. There are two declared for the demo. 
 
 Let's create a new spring context. let's call it `medical-service-context.xml`. We will declare 
 
 A mettre dans le content model. 
 
 --------
 
## 4. Repository - extract metadata
 
 Finally we can start with the most interesting part, coding logic of our feature.
 
 To put into place the mechanism we will create a custom `action` which we will use in a `rule`. 
 
### a. Custom action

An action in Alfresco is a unit of work executed upon a node. In practice it is a Java class that extends
ActionExecuterAbstractBase
 and overrides the executeImpl method. Actions can be called from a rule or can be called from the ActionService 
 (Foundation API). Or even can be configured to be call from the UI in share. In our case we will call the action 
 from a rule apply to a folder. The Upload folder. 

> A rule is applied to `folder` and contain `condition` to execute an `action`. 
We will see later an example of rule.

Let's start! 

- New package java/org/myco/medical/action
- New class ExtractMetadataFromFileName 
- Extends ActionExecuterAbstractBase
- Implements methods:
   - executeImpl(Action action, NodeRef actionedUponNodeRef)
   - addParameterDefinitions(List<ParameterDefinition> paramList)
    
At this step the class looks like this:
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

The addParameterDefinitions method is used to define additional variable for the action. In our case we don't really 
need. So we will leave it empty. Then let's focus on the executeImpl method. This method will contain the all logic 
of our feature. Here the node will the documents the users will drag-and-drop into Alfresco. The node should have a 
file name that follows the pattern we defined previously. 

The logic is :

- Read the node name.
- Parse the name in order to extract personId, documentCode and effectiveDate.
- Call DAOs to retrieve the full person and document type.
- Apply the type ms:document to the node.
- Set all medical properties (metadata).

If we translate this into Java it can look like this
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

Because the logic is a bit too complex for one method it is clearer to split it. To respect some OOP principles which
 are re-usability and modularity and to follow Alfresco architecture we will create new services in our medical API. 
 So I am introducing to you the `medicalDocumentService` and the`medicalFileNameService`. I will detail the 
 implementation in a moment. Actually this code is quite
  self explanatory just to give now the DocumentIdentifiers
call which is a simple POJO that represents the file name parsed. 

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

But one important I haven't mentioned yet is
 we need to tell Alfresco about this new action. For this we
 have to declare a new in our medical-service-context.xml 

{% highlight xml%}
<!--medical actions-->
<bean id="extract-metadata-from-file-name" class="org.myco.medical.action.ExtractMetadataFromFileNameAction" 
parent="action-executer"/>
{% endhighlight %}

I usually define two constants in my actions. Name and description 

{% highlight java%}
public static final String NAME = "extract-metadata-from-file-name";
public static final String DESCRIPTION = "Extract metadata from file name following the pattern personId;documentCode;effectiveDate";
{% endhighlight %}

The NAME should be the same as bean id. We are going to use this constants in our example but it is a good practice 
to have them defined because if we want to call our action with the ActionService we will need to specify the NAME. 

To finalize our Action we need to run this piece of into a transaction. Thanks to the Alfresco Foundation API and the 
TransactionService we can easily do it. 

How it's work :

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

The of dependencies injections: 
{% highlight java%}
// Alfresco foundation API
@Autowired
@Qualifier("NodeService")
private NodeService nodeService;
@Autowired
@Qualifier("TransactionService")
private TransactionService transactionService;

// Myco Medical API
@Autowired
private MedicalFileNameService medicalFileNameService;
@Autowired
private MedicalNodeService medicalNodeService;
@Autowired
private MedicalDocumentService medicalDocumentService;
{% endhighlight %}

## b. MedicalFileNameService 

I have introduce in our action a new medical service. Indeed, since we start to have some treatments around file 
names it is clearer to divide our logic. 
 
Here the class corresponding. 
Basically the main method so far for this service is to parse a file name and return a DocumentIdentifiers.

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

This method can be improved. It is a really simple example. And in a real case you should treat error of format and 
non formatted file name. 

## c. MedicalDocumentService 

It is the second service I've introduced in our Action. This service contains two methods.

- void createMedicalDocumentFromExistingNode(NodeRef nodeRef, MedicalDocument medicalDocument)
- MedicalDocument buildMedicalDocumentFromIdentifiers(DocumentIdentifiers documentIdentifiers)

I'm not going to explain it because there nothing no complexity in it. But you can take a look here to this service.

# 5. The Rule 

I will show you how to manually create the rule using Share. It is possible to create it with the RuleService 
(Alfresco foundation API once again!).

# 6. Unit tests 

# Conclusion
The feature we implemented today is very basic but I saw much more than this. 
We actually saw how to structure All-in-one. How to create a custom content model. How to create custom services. Hoe
 to unit test our customizations. How to create Action. Hot to use Alfresco Foundation API and so on.

 





