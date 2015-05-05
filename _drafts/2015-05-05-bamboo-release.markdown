---
layout: post
title:  "One click release procedure on Bamboo"
date:   2015-05-05 23:56:45
comments: true
categories:
- blog
permalink: one-click-release
description: Configure Bamboo to be able to release a Maven project in one click (more or less).
---

Nowadays it is common to automate as much as possible development steps of a  project such as releasing, deploying,
testing and so on. This is part of what we call Continuous Integration. In that post I will talk about one technical
point - how on Bamboo to configure a plan in such a way we can release a project in one click. Maven is used as a build
tool, the version control system I use is git (hosted on Stash).

We want a branch to be created each time we do a release. Another way could be do have one release branch, then when
you configure the repository you make sure to do the checkout on the specific branch.

## Setup the new plan

We should start to create a new plan. Select a repository
![create plan]({{ site.url }}/assets/posts/release-bamboo/create-plan.png)

Make sure the option `Use shallow clones` is disable in the repository settings.
![create plan]({{ site.url }}/assets/posts/release-bamboo/create-plan.png)

In the newly created plan keep the default checkout task.

## Create a git branch from bamboo

Before going forward we need Bamboo to be able to connect to your git repo. This is different than the config we did
previously on the `Repositories` section in. In fact we want Bamboo itself to have permissions in read-write. To do
this
you need to add the ssh
key from
your Bamboo machine to your Stash repo.

The only way I found to create branches with Bamboo is with the script task. So I created a new script task, `inline`
 with the following content.
{% highlight sh%}
git remote set-url origin ssh://git@somemachine/efiles.git
git checkout -b release/${bamboo_releaseVersion}
git push origin release/${bamboo_releaseVersion}
{% endhighlight %}

I had to explicitly set the url of the origin and then I do a basic `checkout -b` which create a branch and then do a
 checkout on it. The name of the branch use a variable in Bamboo we will in a couple of line. Notice to call variable
  from script task you have to prefix it with `bamboo_` and then the name of the variable.
  Finally the script push the new branch to the origin.

## Release variables

I lied to you when I said a release procedure in one click because you still need to specific variable before running
 it. I suppose it can be fully automated however I find it a bit more flexible like this. In my case I use only two
 variables.

 The releaseVersion which is used to create the next branch, to release and create the tag. The second one is the
 nextVersion used to set the next development version (SNAPSHOT).

![variables]({{ site.url }}/assets/posts/release-bamboo/variables.png)

## Maven release

To do the release itself, I use the maven release plugin and all the magic happens here. Create a new Maven task. I
run a goal more or less similar to this one.

{% highlight sh%}
release:prepare
release:perform
-DreleaseProfiles=prod
-DautoVersionSubmodules=true
-DdevelopmentVersion=${bamboo.nextVersion}
-DreleaseVersion=${bamboo.releaseVersion}
-Dtag=${bamboo.releaseVersion}
{% endhighlight %}

> Remarks:
>
> > It's only one command.
> > I did not available the batch mode because it is automatically called with Bamboo.

To call the maven release plugin you have to call the command release:prepare release:perform. These commands can
take parameters:
- releaseProfiles: you cannot call -PprofileName here, but if you want to set a profile then you need to pass it in
this option. You can pass more than one profile.
- autoVersionSubmodules: in my case I have sub-projects but I want them all with the same version. By default this
option is false.
- developmentVersion: the next snapshot version. I use the Bamboo variable, you can see the prefix `bamboo.` to get
the variable in opposition to the `bamboo_` in the script task.
- releaseVersion: to set the version of the release.
- tag: name of the tag created

## Maven pom

For the release plugin to work and the deploy to be done on your Maven artifact repository you need to provide your
pom with these information. Of course you should replace with real values.
{% highlight xml%}
<distributionManagement>
  <repository>
    <id>releases</id>
    <name>Internal Release Repository</name>
    <url>https://yourrepo/releases</url>
  </repository>
  <snapshotRepository>
    <id>snapshots</id>
    <uniqueVersion>false</uniqueVersion>
    <name>Internal Snapshot Repository</name>
    <url>https://yourrepo/snapshots</url>
  </snapshotRepository>
</distributionManagement>

<scm>
  <developerConnection>scm:git:ssh://git@yourrepo/efiles.git</developerConnection>
  <connection>scm:git:ssh://git@yourrepo/efiles.git</connection>
  <url>https://yourrepourl/efiles</url>
  <tag>HEAD</tag>
  </scm>
{% endhighlight %}

## Useful Tips

If you created "by mistake" some tags, you can get ride of them running these commands:
{% highlight sh%}
git tag -d taganme
git push origin :refs/tags/tagname
{% endhighlight %}

During the phase of setting up the plan and trying the maven release plugin, you can use the option `-DdryRun` such
that
 no
changes will
 be apply.
{% highlight sh%}
release:prepare
release:perform
-DreleaseProfiles=prod
-DautoVersionSubmodules=true
-DdevelopmentVersion=${bamboo.nextVersion}
-DreleaseVersion=${bamboo.releaseVersion}
-Dtag=${bamboo.releaseVersion}
-DdryRun
{% endhighlight %}

I got trouble with an old version of the release plugin, I had to set explicitly in my pom.xml a newer version for
this plugin. The problem was the plugin could not commit and the logs were silent about it.

## To go further

There is actually an issue with this plan. In fact if you find a bug in your release and you fix it but you don't
want to merge on master yet - Then this plan won't work because each time a new branch is created
from master. I am still working on a better solution however you've got good basics to play on your own and you can
still create a second plan you will call when you will fix a release.