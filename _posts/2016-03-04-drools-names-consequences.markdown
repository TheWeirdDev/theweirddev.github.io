---
layout: post
title:  "Drools: example of named consequences"
date:   2016-03-04 23:56:45
comments: true
toc: false
categories:
- blog
permalink: drools-eg-named-consequences
description: Use case, document permission system.
---

There are already theoretical explanations on the internet about named consequences. However, I can't say I was very lucky when looking for practical examples, this is why I decided to share mine afterward.

## Example

Hereunder is a sample of code from a project. It is about permissions on documents. A person asks if he can delete a document. Giving the following instructions:

> * A document contains a list of persons who can delete (same for read, write, etc.), a bit like rights in Linux. A person can only delete if he is in this list. One element of the list is called an ACE (Access Control Entry). 
* A person can't delete his own documents (to avoid any conflict of interest). 
* If the document has been created more than a month ago the document can't be deleted. The permission expired.
* If the person is an admin he can delete any documents at any time.

A person called "requester" is inserted in drools as a fact with a delete permission request for a specific document. ACEs have been generated for this document and this permission. 

{% highlight java%}
rule "An example of named consequences: delete permission"
when
  $requester: Person()
  $ace: Ace(authority == $requester)
  $pr: PermissionRequest(permission == $ace.permission, permission == "delete")
  if($requester == $pr.document.personConcerned) do[DENIED]
  else if(!$requester.isAdmin() && $pr.document.creationDate <= MyDateUtil.getNowPlusOrMinusDays(-30)) do[EXPIRED]
  else do[ALLOW]
then
  logger.debug("Debugging message for instance the Ace{}", $ace);
  // always executed
then [ALLOW]
  $pr.setHasPermission(true);
  update($pr);
  drools.halt();
then [DENIED]
  $pr.setHasPermission(false);
  $pr.setReason("A person can't delete his own documents");
  update($pr);
  drools.halt();
then [EXPIRED]
  $pr.setHasPermission(false);
  $pr.setReason("The delete permission expires after a period of time");
  update($pr);
  drools.halt();
end
{% endhighlight %}