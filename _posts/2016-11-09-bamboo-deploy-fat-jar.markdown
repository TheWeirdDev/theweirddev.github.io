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

This article shows an example of how to deploy a Spring Boot fat jar on a remote server using Bamboo. As reminder, a fat jar is a self-contained jar, 
in other words it contains all the dependent libraries. With Spring boot it's also easy to embed an HTTP listen such as Tomcat.  

## Bamboo plan
A `deployment project` has to be associated with a `Bamboo plan`. Usually, it produces an artifact and this is what you want to deploy. 
Is it without saying that in our case the artifact will be a fat jar.

### Define the artifact
You have to tell Bamboo where to find the artifact. Just follow the steps:  

>Plan configuration > Stage > Artifacts > Create definition

![artifact-definition]({{ site.url }}/assets/posts/bamboo-deploy-jar/artifact-definition.png)

And then fill in the form:

>  
* __Name__: myApp-release-jar  
* __Location__: /target  
* __Copy pattern__: *.jar   
* __Shared__: true (this is important otherwise you won't the artifact from the deployment project)

### Extract maven GAV - optional
You might need to extract the maven GAV (GroupId, Artifact, Version) from the pom.xml, in order to reuse these variables in the deployment project:

>  
* __Add task__: Maven POM Value Extractor
* __Variable Type__: Plan
* __Variable Prefix__: "maven"

### Create deployment project
If you do not know how to create a deployment project [click on this link]({{here}}https://confluence.atlassian.com/display/BAMBOO055/Creating+a+deployment+environment).

The deployment can be triggered right after the plan succeed or it can be scheduled at a specific time. Eventually, no trigger is defined, 
and then it's up to the user to run it manually.

## Deployment project
You must first create an environment, like this one:

![environment]({{ site.url }}/assets/posts/bamboo-deploy-jar/environment.png)

And then we will focus on the `Edit tasks`. Here are the tasks needed:

![all-task]({{ site.url }}/assets/posts/bamboo-deploy-jar/all-tasks.png)

The tasks "Init Kerberos auth" and "Destroy Kerberos" are custom commands. It is used to initiate/close the authentication with the remote server. 
I won't cover the authentication because it might vary for each of your cases.

### Clean working directory task
This is one of the default task. Just keep it as it is.

### Artifact download
This is the second default task, it's used to get the artifact to deploy. Normally in the dropdown list appears the name of the artifact from the Bamboo plan, 
just select it and save. 

If the artifact is not listed, go back to the plan configuration and make sure the `shared option` is checked in the artifact tab.

### Scripts
From now, I will use only `Script tasks` to manage the deployment. It might seem a bit low level, but at the same time it's really flexible.

I'm going to use the following variables, all along the different scripts:  

>
- __releaseVersion__ (plan variable or extracted from the maven pom)
- __app.name__ (name of the application to deploy - also the maven artifact id)
- __app.server__ (user@servername)
- __primary.port__ (the port on which to run the app. In the future there might be more than one app running so this one will be the primary one.)

### Upload jar
The first script task is used to upload the jar on the remote machine. (the bash script goes in the body of the task)

{% highlight sh %}
# At this time, the artifact has been downloaded and is available in the current
# directory (where the script is executed). The name of the jar is the one
# generated during your build process in the target folder. In a normal maven
# project the name should be artifactId-version.jar. In that case, this variable
# should always point to the file:
jarName=${bamboo.app.name}-${bamboo.releaseVersion}.jar

# Location where to upload the jar on the remote server
jarFolder=/folder/on/the/remote/server/jars

# At any time you can use echo to debug
echo "Jar name: $jarName"
echo "Jar folder: $jarFolder"

# Upload the jar on the remote server using a simple scp command
scp $jarName ${bamboo.app.server}:$jarFolder/$jarName

# Update/create symlink.
ssh ${bamboo.app.server} "cd $jarFolder; ln -sf $jarName ${bamboo.app.name}-primary.jar"
{% endhighlight %}

### Stop instance
There might be already an instance of your application running on the server. You should stop it before starting a new one.

{% highlight sh %}
# Check if there are processes running with the app.name - This is just for information
nbServices=$(ssh ${bamboo.app.server} "ps aux | grep -c ${bamboo.app.name}")

# Remove in the count the processes for the grep and the ssh
let "nbServices += -2"

# Check if the port is already used
portUsed=$(ssh ${bamboo.app.name} "/usr/sbin/lsof -i :${bamboo.primary.port} | grep -c LISTEN")

echo "Nb ${bamboo.app.name} running: $nbServices"
echo "Port in use: $portUsed"

if [ "$portUsed" -gt 0 ]
then
    echo "Shuting down the running instance of ${bamboo.app.name}"
    # Shutdown the app using spring actuator shutdown end point.
    ssh --silent ${bamboo.app.name} "curl -X POST http://localhost:${bamboo.primary.port}/shutdown --max-time 5"
    sleep 10
else
    echo "${bamboo.app.name} was not running"
fi
{% endhighlight %}

I could have done something much smarter. But in my case I know this is the only app which runs on this port. 

### Start instance
{% highlight sh %}
javaCmd=/some/path/jdk8/bin/java
timeZoneOpt=user.timezone=Europe/Zurich
jarLocation=/some/path/

# Run the jar - make sure it runs in background and it keeps running after closing the session. 
ssh ${bamboo.app.name} "cd $jarLocation; nohup $javaCmd -jar -D$timeZoneOpt -Dserver.port=${bamboo.primary.port} ${bamboo.app.name}-primary.jar > /dev/null 2>&1 &"
{% endhighlight %}

### Check the instance is running
{% highlight sh %}
sleep 30

nbServices=$(ssh ${bamboo.app.name} "ps aux | grep -c ${bamboo.app.name}")
portUsed=$(ssh ${bamboo.app.name} "/usr/sbin/lsof -i :${bamboo.primary.port} | grep -c LISTEN")
let "nbServices += -2"

if [ "$nbServices" -lt 0 ]
then
    echo "${bamboo.app.name} is not running. Process not found."
    exit 1
elif [ "$portUsed" -gt 0 ]
then
    # it might take a bit more time to show up in lsof, so I do not exit. 
    echo "${bamboo.app.name} is not running. Port ${bamboo.primary.port} not in use."
else
    echo "${bamboo.app.name} is running"
fi
{% endhighlight %}
