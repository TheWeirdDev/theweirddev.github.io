---
layout: post
title:  "Pdf rotation"
date:   2015-07-22 23:56:45
comments: true
categories:
- blog
permalink: rotate-pdf
description: Create a custom action to rotate pdf and add it to share.
---

Let's see how to create a custom action to rotate pdf files. We will use the library pdfBox for which Alfresco
already depends. The result will be display in the Share UI:

![rotate.screen]({{ site.url }}/assets/posts/rotate/rotate-screen.png)

>My config:
>
> > * Alfresco Community 5.0.c
> > * Fedora 20
> > * Maven 3.2.3
> > * Maven SDK 2.0.0
> > * Intellij Idea 14

## Repository

### Create action

The action itself is quite simple. It takes only one parameter to tell which direction you want to rotate.
{% gist 6a33dd8adb60e112ead6 %}

### Spring context
You need to register the bean in the spring context. The id of the bean will re-used later to call the action. Notice
 the parent `action-executer`.
{% highlight xml%}
  <bean id="rotate-pdf" class="somepath/action/RotatePdfAction" parent="action-executer"/>
{% endhighlight %}

## Share

### Share custom config
On the share side, most of the work is done in the `share-config-custom.xml`. We will define two actions, one to rotate
clockwise and
one to rotate counter-clockwise. At the end, we add the actions in the document details page.
{% highlight xml%}
<alfresco-config>

  <config evaluator="string-compare" condition="DocLibActions">
    <actions>
<!-- Action rotate clockwise -->
      <action id="rotate-pdf-clockwise" type="javascript" label="action.rotate-pdf-clockwise" icon="rotate_cw">
        <param name="function">onActionSimpleRepoAction</param>
        <permissions>
          <permission allow="true">Write</permission>
        </permissions>
        <!-- point to the action newly created, here the bean id is used -->
        <param name="action">rotate-pdf</param>
        <!-- param of the action -->
        <param name="clockwise">true</param>
        <param name="failureMessage">message.rotate-pdf.failure</param>
        <evaluator>custom.evaluator.pdf</evaluator>
      </action>
<!-- Action rotate counter-clockwise -->
      <action id="rotate-pdf-anti-clockwise" type="javascript" label="action.rotate-pdf-anti-clockwise" icon="rotate">
        <param name="function">onActionSimpleRepoAction</param>
        <permissions>
          <permission allow="true">Write</permission>
        </permissions>
        <param name="action">rotate-pdf</param>
        <param name="clockwise">false</param>
        <param name="failureMessage">message.rotate-pdf.failure</param>
        <evaluator>custom.evaluator.pdf</evaluator>
      </action>
    </actions>

<!-- Extend the document details action group to add our actions there -->
    <actionGroups>
      <actionGroup id="document-details">
        <action index="600" id="rotate-pdf-clockwise"/>
      </actionGroup>
      <actionGroup id="document-details">
        <action index="601" id="rotate-pdf-anti-clockwise"/>
      </actionGroup>
    </actionGroups>
  </config>

</alfresco-config>
{% endhighlight %}

We will see next how to add labels, evaluator and icons.

### Labels

It looks nicer with labels so I added these ones to my properties:
{% highlight xml%}
# Custom actions labels
message.rotate.failure=Failed to rotate the pdf.
action.rotate-pdf-clockwise=Rotate right
action.rotate-pdf-anti-clockwise=Rotate left
{% endhighlight %}

### Evaluator
We need an evaluator to make the actions appear only for pdf. An is easy way is to re-use an out of the
box evaluator which returns true if it matches one mimetype from a predefined list (for us the list will
 be only pdf).
To do so add this bean in your share spring context.
{% highlight xml%}
<bean id="custom.evaluator.pdf" parent="evaluator.doclib.action.isMimetype">
  <property name="mimetypes">
    <list>
      <value>application/pdf</value>
    </list>
  </property>
</bean>
{% endhighlight %}

### Icons

Finally the icons:
![rotate1]({{ site.url }}/assets/posts/rotate/rotate-16.png)
![rotate2]({{ site.url }}/assets/posts/rotate/rotate_cw-16.png)
You can probably find better. The most important is where to put them, in the right place:
`custom-share-apm/src/main/resources/META-INF/components/documentlibrary/actions`