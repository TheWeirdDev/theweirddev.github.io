---
layout: post
title:  "Automatic fat jar deployment with Bamboo"
date:   2016-11-09 23:56:45
comments: true
permalink: fat-jar-deployment-bamboo
toc: true
description: Deployment project on Bamboo to automate the deployment of fat jar.
---

> Config:  
- Bamboo 5.5  
- Spring Boot  
- Maven  

This article is an example of how to deploy a Spring Boot fat jar on a remote server using Bamboo. As reminder, a fat jar is a self-contained jar, in other words it contains all the dependent libraries. With Spring boot it's also easy to embed an HTTP listen such as Tomcat.  

## Bamboo plan
A __deployment project__ has to be associated with a __Bamboo plan__. So first, let's have a look to it. Usually, it produces an artifact and this is what you want to deploy. Is it without saying, in our case the artifact will be a fat jar.

### Define the artifact
We have to tell Bamboo where to find the artifact. Follow the steps:  
`Plan configuration > Stage > Artifacts > Create definition`

And then fill in the form:

>  
* __Name__: myApp-release-jar  
* __Location__: /target  
* __Copy pattern__: *.jar   
* __Shared__: true (this is important otherwise you won't the artifact from the deployment project)

### Optional - extract maven GAV
You might need to extract the maven GroupId, Artifact and Version from the pom.xml, in order to reuse them in the deployment project:

>  
* __Add task__: Maven POM Value Extractor
* __Variable Type__: Plan
* __Variable Prefix__: "maven"

### Deployment project
To create a deployment project [click on this link]({{here}}https://confluence.atlassian.com/display/BAMBOO055/Creating+a+deployment+environment).

The deployment can be triggered right after then plan succeeded or scheduled at a specific time. Eventually, no trigger might be defined, and then it's up to the user to run it manually.

## Deployment project
I will cover only on environment, If don't have create a new environment. Here the list of tasks covered in the next chapters.

![all-task]({{ site.url }}/assets/posts/bamboo-deploy-jar/all-tasks.png)

The tasks "Init Kerberos auth" and "Destroy Kerberos" are custom commands. It is used to initiate/close the authentication with the remote server. I won't cover the authentication because it might vary for each of your case. Instead, let's focus on the script tasks.

### Clean working directory task
This is one of the default task. Just keep it as it is, or you can write a description if you want to.

### Artifact download
This is the second default task, it's used to get the artifact to deploy. Normally, in the dropdown list the name of the artifact from the Bamboo plan should appear, 
just select it and save. 

If the artifact does not appear in the list, go back to the plan configuration and make sure the "shared option" is checked in the artifact tab.

### Scripts
From now, I will use only "Script" tasks to manage the deployment. It might seem a bit low level, but at the same time it's really flexible.

I'm going to use the following variables, all along the different scripts:

>
- __releaseVersion__ (plan variable or extracted from the maven pom)
- __app.name__
- __app.server__ (user@servername)
- __primary.port__

### Upload jar
New "Script task". Here is the body:

{% highlight sh %}
# At this time, the artifact has been downloaded and is available in the current
# directory (where the script is executed). The name of the jar is the one
# generated during your build process in the target folder. In a normal maven
# project the same should be artifactId-version.jar. In that case, this variable
# should always point to the file:
jarName=${bamboo.app.name}-${bamboo.releaseVersion}.jar

# Location for the upload on the remote server
jarFolder=/folder/on/the/remote/server/jars

# At any time you can use echo to debug
echo "Jar name: $jarName"
echo "Jar folder: $jarFolder"

# Upload the jar on the remote server
scp $jarName ${bamboo.app.server}:$jarFolder/$jarName

# Update/create symlink.
ssh ${bamboo.app.server} "cd $jarFolder; ln -sf $jarName ${bamboo.app.name}-primary.jar"
{% endhighlight %}

### Stop instance
There might be already an instance of your application running on the server. You should stop it before starting a new one.

{% highlight sh %}
# check processes running
nbServices=$(ssh ${bamboo.app.server} "ps aux | grep -c ${bamboo.app.name}")

# there is always noise - E.g. the grep and the ssh
let "nbServices += -2"

# return 1 if the port is already used
portUsed=$(ssh ${bamboo.efiles.env} "/usr/sbin/lsof -i :${bamboo.primary.port} | grep -c LISTEN")

echo "Nb ${bamboo.app.name} running: $nbServices"
echo "Port in use: $portUsed"

if [ "$nbServices" -gt 0 ] || [ "$portUsed" -gt 0 ]
then
    echo "Shuting down the running instance of ${bamboo.app.name}"
    ssh --silent ${bamboo.efiles.env} "curl -X POST http://localhost:${bamboo.primary.port}/shutdown --max-time 5"
    sleep 10
else
    echo "${bamboo.app.name} was not running"
fi

# check the app has stopped
portStillUsed=$(ssh ${bamboo.efiles.env} "/usr/sbin/lsof -i :${bamboo.primary.port} | grep -c LISTEN")
if [ "$portUsed" -gt 0 ]
then
    ssh ${bamboo.efiles.env} "/usr/sbin/lsof -i :${bamboo.primary.port} | grep LISTEN"
    echo "${bamboo.app.name} not stop yet"
fi
{% endhighlight %}

### Start instance
{% highlight sh %}
javaCmd=/some/path/jdk8/bin/java
timeZoneOpt=user.timezone=Europe/Zurich
jarLocation=/some/path/

# Run the jar
ssh ${bamboo.efiles.env} "cd $jarLocation; nohup $javaCmd -jar -D$timeZoneOpt -Dserver.port=${bamboo.primary.port} ${bamboo.app.name}-primary.jar > /dev/null 2>&1 &"
{% endhighlight %}

### Check the instance is running
{% highlight sh %}
sleep 30

nbServices=$(ssh ${bamboo.efiles.env} "ps aux | grep -c ${bamboo.app.name}")
portUsed=$(ssh ${bamboo.efiles.env} "/usr/sbin/lsof -i :${bamboo.primary.port} | grep -c LISTEN")
let "nbServices += -2"

if [ "$nbServices" -lt 0 ]
then
    echo "${bamboo.app.name} is not running. Process not found."
    exit 1
elif [ "$portUsed" -gt 0 ]
then
    echo "${bamboo.app.name} is not running. Port ${bamboo.primary.port} not in use."
else
    echo "${bamboo.app.name} is running"
fi
{% endhighlight %}

This task does not take any decision if the application in not running, you might want to do something instead. In my case it is here only for information.