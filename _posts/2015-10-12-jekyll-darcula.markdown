---
layout: post
title:  "Darcula theme for Pygments"
date:   2015-10-12 23:56:45
comments: true
permalink: pygments-darcula
toc: false
description: Pygments is the code highlighter I use with Jekyll for my blog.
---

I've been recently refactoring my blog and I couldn't find a specific dark theme for the code highlighter. There are 
quite a few on GitHub, however I wanted one that fits well with my blog theme. I actually quite like the dark theme in Intellij Idea 
so I created one with the same tones. It has definitely some differences nevertheless the result looks fine.

> I'd like to thank Richeland because I used one of his CSS at [http://richleland.github.io/pygments-css/](http://richleland.github.io/pygments-css)
 to start.

## Examples

I haven't tried all languages to see how it's displayed, so far it performs well with XML, Java, JSON, JavaScript, and CSS.
Let's see some examples.

**`Java + horizontal scroll:`**
{% highlight java %}
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
}
{% endhighlight %}

**`XML:`**
{% highlight xml %}
<!-- ***************************************************** -->
<!-- Effective date aspect. Effective date of the document -->
<!-- ***************************************************** -->
<aspect name="ms:effectiveDate">
 <title>Effective date</title>
 <properties>
   <property name="ms:date">
     <type>d:date</type>
     <mandatory>true</mandatory>
   </property>
 </properties>
</aspect>
{% endhighlight %}

**`JSON + line numbers:`**
{% highlight json linenos %}
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

> You can find more real examples all over my blog [here]({{ site.url }}).

## The theme 

This is the scss for the theme. You can find it on Github as well [_darcula.scss](https://github.com/smasue/pygments/blob/master/_darcula.scss)

{% highlight scss %}
$foreground: #A9B7C6;
$background: #2B2B2B;
$selectionForeground: #A9B7C6;
$selectionBackground: #214283;

$operators: #A9B7C6;
$strings: #A5C25C;
$names: #88be05;
$keywords: #CB772F;
$varAndProp: #A9B7C6;
$numbers: #6897BB;
$tag: #f1c829;
$attributes: #9876AA;
$comments: #75715e;
$linenos: #A9B7C6;

.highlight {
  margin-bottom: 1.5em;
  color: $foreground;
  background-color: $background;
  @include rounded(4px);
  pre {
    position: relative;
    margin: 0;
    padding: 1em;
    overflow-x: auto;
  }
  pre::-webkit-scrollbar {
    height: 10px;
    background-color: #34362e;
    border-radius: 0 0 4px 4px;
  }
  pre::-webkit-scrollbar-thumb{
    background-color: #75715e;
    @include rounded(4px);
  }
  ::selection {
    background-color: $selectionBackground;
    color: $selectionForeground;
  }
   .hll { background-color: #49483e }
   .c { color: $comments } /* Comment */
   .err { color: #960050; background-color: #1e0010 } /* Error */
   .k { color: $keywords } /* Keyword */
   .l { color: $numbers} /* Literal */
   .n { color: $varAndProp } /* Name */
   .o { color: $operators } /* Operator */
   .p { color: $varAndProp } /* Punctuation */
   .cm { color: #75715e } /* Comment.Multiline */
   .cp { color: #75715e } /* Comment.Preproc */
   .c1 { color: #75715e } /* Comment.Single */
   .cs { color: #75715e } /* Comment.Special */
   .ge { font-style: italic } /* Generic.Emph */
   .gs { font-weight: bold } /* Generic.Strong */
   .kc { color: $keywords } /* Keyword.Constant */
   .kd { color: $keywords } /* Keyword.Declaration */
   .kn { color: $operators } /* Keyword.Namespace */
   .kp { color: $keywords } /* Keyword.Pseudo */
   .kr { color: $keywords } /* Keyword.Reserved */
   .kt { color: $keywords } /* Keyword.Type */
   .ld { color: $strings } /* Literal.Date */
   .m { color: $numbers} /* Literal.Number */
   .s { color: $strings } /* Literal.String */
   .na { color: $attributes } /* Name.Attribute */
   .nb { color: $varAndProp } /* Name.Builtin */
   .nc { color: $names } /* Name.Class */
   .no { color: $keywords } /* Name.Constant */
   .nd { color: #f1c829 } /* Name.Decorator */
   .ni { color: $varAndProp } /* Name.Entity */
   .ne { color: $names } /* Name.Exception */
   .nf { color: $names } /* Name.Function */
   .nl { color: $varAndProp } /* Name.Label */
   .nn { color: $varAndProp } /* Name.Namespace */
   .nx { color: $names } /* Name.Other */
   .py { color: $varAndProp } /* Name.Property */
   .nt { color: $tag } /* Name.Tag */
   .nv { color: $varAndProp } /* Name.Variable */
   .ow { color: $operators } /* Operator.Word */
   .w { color: $varAndProp  } /* Text.Whitespace */
   .mf { color: $numbers } /* Literal.Number.Float */
   .mh { color: $numbers } /* Literal.Number.Hex */
   .mi { color: $numbers } /* Literal.Number.Integer */
   .mo { color: $numbers } /* Literal.Number.Oct */
   .sb { color: $strings } /* Literal.String.Backtick */
   .sc { color: $strings } /* Literal.String.Char */
   .sd { color: $strings } /* Literal.String.Doc */
   .s2 { color: $strings } /* Literal.String.Double */
   .se { color: $numbers} /* Literal.String.Escape */
   .sh { color: $strings } /* Literal.String.Heredoc */
   .si { color: $strings } /* Literal.String.Interpol */
   .sx { color: $strings } /* Literal.String.Other */
   .sr { color: $strings } /* Literal.String.Regex */
   .s1 { color: $strings } /* Literal.String.Single */
   .ss { color: $strings } /* Literal.String.Symbol */
   .bp { color: $varAndProp  } /* Name.Builtin.Pseudo */
   .vc { color: $varAndProp } /* Name.Variable.Class */
   .vg { color: $varAndProp } /* Name.Variable.Global */
   .vi { color: $varAndProp } /* Name.Variable.Instance */
   .il { color: $numbers} /* Literal.Number.Integer.Long */

   .gh { } /* Generic Heading & Diff Header */
   .gu { color: #75715e; } /* Generic.Subheading & Diff Unified/Comment? */
   .gd { color: $operators; } /* Generic.Deleted & Diff Deleted */
   .gi { color: $names; } /* Generic.Inserted & Diff Inserted */
   .l-Scalar-Plain {color: $names}

   /* line numbers */
   .lineno{ border-right: solid 1px $linenos; color: $linenos; padding-right: 5px; }
}
{% endhighlight %}

In addition you will need this:

{% highlight scss %}
@mixin rounded($radius:4px) {
  -webkit-border-radius : $radius;
  -moz-border-radius : $radius;
  border-radius : $radius;
}
{% endhighlight %}
