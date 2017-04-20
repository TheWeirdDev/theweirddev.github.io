---
layout: post
title:  "Database appender with log4j2 and Spring Boot"
date:   2016-12-13 23:56:45
comments: true
permalink: log4j2-spring-database-appender
toc: false
description: Use the Spring datasource properties in the configuration of a log4j2 database appender.
---

The goal of this short example is to show how to configure a log4j2 database appender, and making it using the database configuration 
properties 
from a Spring properties file.

> Config:  
>
* Spring Boot 1.4.0.RELEASE
* Maven
* Oracle 12g

## Maven dependency
We need to define the dependency to the log4j2 starter.

{% highlight xml %}
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
{% endhighlight %}

The starter is going to import the following libraries:  

- log4j-core
- log4j-slf4j-impl
- log4j-api

## Create the log table
Of course, we have to create a table for the logs. This is a simple example, feel free to adapt it to your needs:
{% highlight sql %}  
CREATE TABLE LOGS
(
    APPLICATION VARCHAR2(20) NULL,
    LOG_DATE   DATE      NOT NULL,
    LOGGER  VARCHAR2(250)    NOT NULL,
    LOG_LEVEL   VARCHAR2(10)    NOT NULL,
    MESSAGE VARCHAR2(4000)  NOT NULL
);
{% endhighlight %}

## Spring Boot
I found it easier to register the appender programmatically during the startup of the application context. Here how it works, I created
 a Spring configuration file dedicated to it:
{% highlight java %}
package some.package.config;

import org.apache.logging.log4j.Level;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.core.Logger;
import org.apache.logging.log4j.core.appender.db.jdbc.ColumnConfig;
import org.apache.logging.log4j.core.appender.db.jdbc.JdbcAppender;
import org.apache.logging.log4j.core.filter.ThresholdFilter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.env.Environment;

import javax.annotation.PostConstruct;
import some.package.logging.JdbcConnectionSource;

@Configuration
public class LogConfig
{
  @Autowired
  private Environment env;

  @PostConstruct
  public void onStartUp()
  {
    String url = env.getProperty("spring.datasource.url");
    String userName = env.getProperty("spring.datasource.username");
    String password = env.getProperty("spring.datasource.password");
    String validationQuery = env.getProperty("spring.datasource.validation-query");
    
    // Create a new connectionSource build from the Spring properties
    JdbcConnectionSource connectionSource = new JdbcConnectionSource(url, userName, password, validationQuery);

    // This is the mapping between the columns in the table and what to insert in it.
    ColumnConfig[] columnConfigs = new ColumnConfig[5];
    columnConfigs[0] =  ColumnConfig.createColumnConfig(null, "APPLICATION", "ACCESS", null, null, "false", null);
    columnConfigs[1] =  ColumnConfig.createColumnConfig(null, "LOG_DATE", null, null, "true", null, null);
    columnConfigs[2] =  ColumnConfig.createColumnConfig(null, "LOGGER", "%logger", null, null, "false", null);
    columnConfigs[3] =  ColumnConfig.createColumnConfig(null, "LOG_LEVEL", "%level", null, null, "false", null);
    columnConfigs[4] =  ColumnConfig.createColumnConfig(null, "MESSAGE", "%message", null, null, "false", null);
    
    // filter for the appender to keep only errors
    ThresholdFilter filter = ThresholdFilter.createFilter(Level.ERROR, null, null);
    
    // The creation of the new database appender passing:
    // - the name of the appender
    // - ignore exceptions encountered when appending events are logged
    // - the filter created previously
    // - the connectionSource, 
    // - log buffer size, 
    // - the name of the table 
    // - the config of the columns.
    JdbcAppender appender = JdbcAppender.createAppender("DB", "true", filter, connectionSource, "1", "LOGS", columnConfigs);
    
    // start the appender, and this is it...
    appender.start();
    ((Logger) LogManager.getRootLogger()).addAppender(appender);
  }
}
{% endhighlight %}

## Connection source

The final step is the connection source. Here are the sources:
{% highlight java %}
package some.package.logging;

import org.apache.commons.dbcp.DriverManagerConnectionFactory;
import org.apache.commons.dbcp.PoolableConnection;
import org.apache.commons.dbcp.PoolableConnectionFactory;
import org.apache.commons.dbcp.PoolingDataSource;
import org.apache.commons.pool.impl.GenericObjectPool;
import org.apache.logging.log4j.core.appender.db.jdbc.ConnectionSource;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;
import java.util.Properties;

public class JdbcConnectionSource implements ConnectionSource
{
  private DataSource dataSource;

  public JdbcConnectionSource(String url, String userName, String password, String validationQuery)
  {
    Properties properties = new Properties();
    properties.setProperty("user", userName);
    properties.setProperty("password", password);

    GenericObjectPool<PoolableConnection> pool = new GenericObjectPool<>();
    DriverManagerConnectionFactory cf = new DriverManagerConnectionFactory(url, properties);
    new PoolableConnectionFactory(cf, pool, null, validationQuery, 3, false, false, Connection.TRANSACTION_READ_COMMITTED);
    this.dataSource = new PoolingDataSource(pool);
  }

  @Override
  public Connection getConnection() throws SQLException
  {
    return dataSource.getConnection();
  }
}
{% endhighlight %}


A datasource is initialized in the constructor of the class, and then the idea is to override the method getConnection.