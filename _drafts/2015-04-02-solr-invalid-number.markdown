---
layout: post
title:  "Solr invalid number exception"
date:   2015-04-02 23:56:45
categories:
- blog
permalink: solr-invalid-number
description: We will se how to extract metadata from a formatted file name.
---

{% highlight javascript%}
<import resource="classpath:/alfresco/templates/webscripts/org/alfresco/slingshot/search/search.lib.js">

 function main()
 {
   var params =
   {
     siteId: args.site,
     containerId: args.container,
     repo: (args.repo !== null) ? (args.repo == "true") : false,
     term: buildCustomTerm(args.term),
     tag: args.tag,
     query: args.query,
     rootNode: args.rootNode,
     sort: args.sort,
     maxResults: (args.maxResults !== null) ? parseInt(args.maxResults, 10) : DEFAULT_MAX_RESULTS,
     pageSize: (args.pageSize !== null) ? parseInt(args.pageSize, 10) : DEFAULT_PAGE_SIZE,
     startIndex: (args.startIndex !== null) ? parseInt(args.startIndex, 10) : 0,
     facetFields: args.facetFields,
     filters: args.filters,
     spell: (args.spellcheck !== null) ? (args.spellcheck == "true") : false
   };

   model.data = getSearchResults(params);
 }

main();

/**
 * ############################################################################################################
 * Due to Alfresco SolrException (Invalid number) after adding in the "default query template" (search.get.config.xml)
 * custom properties which are of type of number. We had to remove these properties from the template tough we couldn't
 * search automatically on all the metadata we wanted. The idea in order to fix that is to augment the query by adding
 * manually our custom properties (the ones which are numbers).
 *
 * This is a patch and should be reverted as soon as as a fix is released.
 * ############################################################################################################
 * /

 /**
 * Build a query snippet that allows to search for our custom properties (needed only for properties of type of number).
 * E.g. number = "75000" -> (ef:personId:75000 OR ef:cernId:75000 OR 75000).
 *
 * Ours custom properties are ef:personId and ef:cernId. Feel free to add more in the list if required
 *
 * @param number
 * @returns query snippet
 */
function buildCustomQuerySnippetForNumber(number)
{
  var replacementPattern = "(ef:personId:number OR ef:cernId:number OR number)";
  return replacementPattern.replace(new RegExp('number', 'g'), number);
}

/**
 * Got through all the single terms of stringTerm and then for each numbers we build a new query snippet that allows
 * to search for our fields.
 * E.g. stringTerm = "Mars 75000" -> Mars (ef:personId:75000 OR ef:cernId:75000 OR 75000)
 *
 * @param stringTerm
 * @returns augmented term
 */
function buildCustomTerm(stringTerm)
{
  var terms = stringTerm.split(" ");
  var augmentedTerm = stringTerm;
  for (var index = 0; index < terms.length; index++)
  {
    var singleTerm = terms[index];
    // if singleTerm is a number
    if (!isNaN(singleTerm))
    {
      augmentedTerm = augmentedTerm.replace(singleTerm, buildCustomQuerySnippetForNumber(singleTerm));
    }
  }
  return augmentedTerm;
}
{% endhighlight %}
