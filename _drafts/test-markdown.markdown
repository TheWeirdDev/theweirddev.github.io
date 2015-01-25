---
layout: post
title:  "Extract metadata from file names"
date:   2015-01-18 23:56:45
categories:
- blog
permalink: extract-metadata
description: We will se how to extract metadata from a formatted file name.
---

Here the idea is to use the names of file to send some metadata into Alfresco. 

Let's take an example. I am developing a small site for the medical service of my company. The documents uploaded should have the metadata about the person concerned and
the type of document (certificate, prescription, ...). To avoid to enter all the information about the person and type of document which we already have in a database.
 We offer the solution to enter a file name that match a pre-defined pattern and then with a simple drag-and-drop into 
 Alfresco they can archive all there documents without editing anything. 
 
 I'll go trough few steps to cover the articles. 
 
 1. File name pattern
 2. content model
 3. 
 4. extra - new behaviour with
 
 ##File name pattern
 
 **salut**
 _test_
 
 __test__
 *test*
 
 Titre de niveau 1
 =================

 Titre de niveau 2
 -----------------
 
# Titre de niveau 1

## Titre de niveau 2

 ### Titre de niveau 3
 
 * Une puce
 * Une autre puce
 * Et encore une autre puce !
 
 * Une puce
 * Une autre puce
     * Une sous-puce
     * Une autre sous-puce
 * Et encore une autre puce !
 
> Ceci est un texte cité. Vous pouvez répondre
> à cette citation en écrivant un paragraphe
> normal juste en-dessous !
 
 > Une citation
 >
 > > Une réponse à la citation
 > >
 > > Réponse qui contient une liste à puces :
 > >
 > > * Puce
 > > * Autre puce
 
 -----------------
 
 A First Level Header
 ====================


