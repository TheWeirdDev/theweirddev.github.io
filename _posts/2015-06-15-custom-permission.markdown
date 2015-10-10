---
layout: post
title:  "Add custom roles in Alfresco"
date:   2015-06-17 23:56:45
comments: true
categories:
- blog
permalink: add-custom-role
description: Following a simple use case we will see how to add new roles in Alfresco.
---

For some cases, out of the box roles in Alfresco are not enough. However, it's not so hard to
 extend them. We will see how to do it using a simple example.

> Config:
>
> > * Alfresco Community 5.0.c
> > * Fedora 20
> > * Maven 3.2.3
> > * All-in-one Alfresco Maven SDK 2.0.0
> > * Intellij Idea 14
>

## 1. Use case
 I have a folder which contains documents and I would like users to be able to delete and edit the documents but not
 create new ones, in summary I would like these permissions:

* Read
* Edit
* Delete
* But not create

Alfresco does provide the role `Editor` which combines these permissions except that it doesn't
allow us to delete documents (children). So we will try to add this `advanced editor` role but before I would like to
introduce some vocabulary:

> A **base permission** is a single unit and represents an action such as Read, Delete, Edit and so on.

> A **role** is a set of base permissions (contains one or more base permissions). Examples are Collaborator,
Contributor,
Editor, ...

> **Setting a permission** is done by associating a role with an authority (user or group of
users) on a specific node. It's a triplet `role / authority / node`.
The idea is to determine which actions can do a certain user on a specific node.

## 2. Permission definitions

To extend permissions in Alfresco you have to write `permission definitions`. Actually it's only one xml file, the
`permissionDefinitions.xml`. There is a dtd associated with it, `permissionSchema.dtd`, which is well documented and
gives really useful information to start with. This chapter will give details on how to configure
permissions but it really worth it having a look to these files, you can find them in the sources of Alfresco.

### a. permissionSet

Define a new set of permissions (base permissions) and permission groups (roles) for a type or aspect. Sub-element of
 the root element. It's usually where to start.

Sub-elements: `permissionGroup`, `permission`

|type|prefix:name|A permission set is associated with a type or aspect and applies only to that type and sub-types, or aspect and sub-aspects|mandatory|
|expose|all,selected|governs if these permissions are exposed on the permission model. If all, exposes all permission groups. If selected, only the permission groups that say they are explicitly exposed are exposed|default all|

{% highlight xml%}
<permissionSet type="sys:base" expose="all">
<!-- permissions and permissionGroups -->
</permissionSet>
{% endhighlight %}

### b. permissionGroup (role)

Build a new "role" combining existing permissionGroups. If the permissionGroup doesn't include other permissionGroup,
 it means it has been granted to a single permission (see permission below).

Sub-element: `includePermissionGroup`.

|name|string|simple name of the permission. Full name includes the type uri from the outer permissionSet|mandatory|
|type|prefix:name|If the permission group extends another then this attribute can be used to specify the type it extends. Normally it would assume this follows the data dictionary type hierarchy|implied|
|extends|boolean|does this permission group extend one that already exists?|default false|
|expose|boolean|if the the containing permission set does not expose all permission groups specify if this particular permission group is exposed or not|default false|
|allowFullControl|boolean|if true, this permission group effectively grants all permissions|default false|
|requiresType|default|this is useful for permission groups tied to aspects. If false, the permission group applies to all types as they could have the aspect. If true, the permission group only makes sense if the aspect has been applied|default true|

> **includePermissionGroup**
>
|type|prefix:name|the type on which to find the permission group to include|
|permissionGroup|string|the name of the permission group to include as defined on the type|

{% highlight xml%}
<permissionSet type="sys:base" expose="all">
<!-- example extracted from permissionDefinitions.xml (sys:base) -->
  <permissionGroup name="FullControl" expose="true" allowFullControl="true" />

  <permissionGroup name="ReadProperties" expose="true" allowFullControl="false" />

  <permissionGroup name="ReadContent" expose="false" allowFullControl="false" />

  <permissionGroup name="ReadChildren" expose="true" allowFullControl="false" />

  <permissionGroup name="Read"  expose="true" allowFullControl="false">
       <includePermissionGroup type="sys:base" permissionGroup="ReadProperties"/>
       <includePermissionGroup type="sys:base" permissionGroup="ReadChildren"/>
       <includePermissionGroup type="sys:base" permissionGroup="ReadContent"/>
  </permissionGroup>
</permissionSet>
{% endhighlight %}

### c. permission

Define a base permission. The permissionGroup(s) to which this permission applies are defined in the grantedToGroup
element. Other permissions which are required or implied by this permission can be defined using requiredPermission elements.

Sub-element(s): `grantedToGroup`, `requiredPermission`.

|**name**|string|same as permissionGroup|mandatory|
|**expose**|boolean|same as permissionGroup|default false|
|**requiresType**|boolean|same as permissionGroup|default true|

> **grantedToGroup**
>
|type|prefix:name|the type on which the permissionGroup is defined|implied|
|permissionGroup|string|the name of the permission group|mandatory|

> **requiredPermission**
>
|name|string|the name of the required permission or permission group|mandatory|
|type|string|the type of the required permission or permission group|implied|
|on|node, parent, children|if required permission must be present|mandatory|
|implies|boolean|if false the permission will be checked. If true, the permission is effectively granted along with this one|default false|
>
> More about implies: This will normally be false. For example, to require read permission on the parent to be able
to read the node. This requirement would be recursive, as read on any node would require read on the parent of that
node. If true this is the case where this permission allows the user to take another action as it is required to
carry out the first. Normally you would protect the method call to require both permissions. This does really grant the other permission. If a permission A is defined that requires another permission B, with implies true, then granting someone permission A will also grant permission B. If implies is false, then granting A will not allow A until permission B is also available.

{% highlight xml%}
<!-- example extracted from permissionDefinitions.xml (sys:base) -->
<permission name="_ReadProperties" expose="false" >
   <grantedToGroup permissionGroup="ReadProperties" />
   <!-- Commented out parent permission check ...
   <requiredPermission on="parent" name="_ReadChildren" implies="false"/>
   -->
</permission>
                                                                            -->
<permission name="_ReadChildren" expose="false" >
   <grantedToGroup permissionGroup="ReadChildren" />
   <!-- Commented out parent permission check ...
   <requiredPermission on="parent" name="_ReadChildren" implies="false"/>
   -->
</permission>
{% endhighlight %}

### d. globalPermissions

A global permission assignment.

|authority|string|the string representation of the authority|implied|
|permission|string|the permission that is granted (name or full name)|mandatory|

We are now ready to customize the permissions.

## 3. Add custom role

### a. custom permission definitions

Let's start adding our new role **Advanced Editor**. In a new file `customPermissionDefintion.xml` located in
`{repo...}/amp/config/alfresco/module/{your-module}/model` (replace the curly brackets with your module name). Start to
 add these
lines. I added as well my custom namespace
containing my content model.

{% highlight xml%}
<?xml version='1.0' encoding='UTF-8'?>
<!DOCTYPE permissions >

<permissions>

  <!-- Namespaces used in type references -->

  <namespaces>
    <namespace uri="http://www.alfresco.org/model/system/1.0" prefix="sys"/>
    <namespace uri="http://www.alfresco.org/model/content/1.0" prefix="cm"/>
    <namespace uri="http://www.alfresco.org/model/site/1.0" prefix="st"/>
    <namespace uri="http://www.custom.ch/model/custom/1.0" prefix="cu"/>
  </namespaces>

</permissions>
{% endhighlight %}

Then we will add a new `permissionSet` because I want the new role (permissionGroup) to be specific of my content
model.

{% highlight xml%}
<permissionSet type="cu:folder" expose="selected">
  <!-- permission and permissionGroup-->
</permissionSet>
{% endhighlight %}

Add the new permissionGroup in the permissionSet:
{% highlight xml%}
<!-- Advanced Editor role -->
<permissionGroup name="AdvancedEditor" allowFullControl="false" expose="true">
  <includePermissionGroup permissionGroup="Editor" type="cm:cmobject" />
  <includePermissionGroup permissionGroup="Delete" type="sys:base" />

    <!-- Keep permissionGroups from cm:folfer -->

    <permissionGroup name="Coordinator" extends="true" expose="true"/>
    <permissionGroup name="Collaborator" extends="true" expose="true"/>
    <permissionGroup name="Contributor" extends="true" expose="true"/>
    <permissionGroup name="Editor" extends="true" expose="true"/>
    <permissionGroup name="Consumer" extends="true" expose="true"/>
    <permissionGroup name="RecordAdministrator" extends="true" expose="false"/>
</permissionGroup>
{% endhighlight %}

If you want to keep the permissions and permissions groups from cm:folder you need to copy/past them. I did find yet
a better solution.

### b. boostrap the custom permission definition

Add in your spring context file this bean definition:

{% highlight xml%}
 <!-- Bootstrap the permission model -->
  <bean id="custom_permissionDefinition" parent="permissionModelBootstrap">
    <property name="model"
              value="alfresco/module/{your-module}/model/{custom}PermissionDefinitions.xml"/>
  </bean>
{% endhighlight %}

My context file is located in `{repo...}/amp/config/alfresco/module/{your-module}/context/{custom}-context.xml` and is
imported
 by the module-context.xml.

### c. Label

If you want to set this new role from the UI share you might want to give it a proper label. You must have a file

`{share..}/amp/config/alfresco/module/{your-module}/messages/{custom}.properties`

Add this line:
{% highlight xml%}
# Custom roles
roles.advancededitor=Advanced editor
{% endhighlight %}

After this to use the role you can either use share "Manage Permissions" or pragmatically.

## 4. Further

### a. site permission definitions
There is another example of permission definitions, `sitePermissionDefinitions.xml`, which contains the configurations
 for the
sites permissions.

### b. Permission service Java API

Using the permission service from Alfresco foundation API it's really easy to set a role to a node for an authority.

{% highlight java%}
permissionService.setPermission(nodeRef, "customGroupOrUser", "AdvancedEditor", true);
{% endhighlight %}

The boolean parameter is used to allow the permission.


