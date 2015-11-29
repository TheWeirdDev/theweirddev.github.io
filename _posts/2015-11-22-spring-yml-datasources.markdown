---
layout: post
title:  "Spring boot: YAML multiple datasources"
date:   2015-11-22 23:56:45
comments: true
toc: false
categories:
- blog
permalink: spring-yml-datasources
description: How to configure multiple datasources in Srping boot using YAML.
---

In this example we will see how to config two datasources on different environments (development, test, production)
using a YAML config file.

## application.yml

In order to select the environment I used profiles. Then, we can see the datasources configured on each
environments.

{% highlight yaml%}
spring:
  profiles.active: development

---
spring:
  profiles: development
datasource:
  base-person:
      url: jdbc:oracle:thin:@db_person_dev
      username: username
      password: pwd
      driver-class-name: oracle.jdbc.OracleDriver
      validationQuery: "SELECT 1 FROM dual"
  base-contrat:
      url: jdbc:oracle:thin:@db_contract_dev
      username: username
      password: pwd
      driver-class-name: oracle.jdbc.OracleDriver
      validationQuery: "SELECT 1 FROM dual"

---

spring:
  profiles: test
datasource:
  base-person:
      url: jdbc:oracle:thin:@db_person_test
      username: username
      password: pwd
      driver-class-name: oracle.jdbc.OracleDriver
      validationQuery: "SELECT 1 FROM dual"
  base-contrat:
      url: jdbc:oracle:thin:@db_contract_test
      username: username
      password: pwd
      driver-class-name: oracle.jdbc.OracleDriver
      validationQuery: "SELECT 1 FROM dual"

---

spring:
  profiles: production
datasource:
  base-person:
      url: jdbc:oracle:thin:@db_person_prod
      username: username
      password: pwd
      driver-class-name: oracle.jdbc.OracleDriver
      validationQuery: "SELECT 1 FROM dual"
  base-contrat:
      url: jdbc:oracle:thin:@db_contract_prod
      username: username
      password: pwd
      driver-class-name: oracle.jdbc.OracleDriver
      validationQuery: "SELECT 1 FROM dual"

---
{% endhighlight %}

## Application.groovy

When configuring multiple datasources we have to define the beans explicitly. Indeed, we need to tell Spring which  
 properties to use for each datasource. Notice that one datasource can be marked as @primary which can save us to use the qualifier
 for the autowiring.

{% highlight groovy%}
import org.springframework.boot.SpringApplication
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.autoconfigure.jdbc.DataSourceBuilder
import org.springframework.boot.context.properties.ConfigurationProperties
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Primary

import javax.sql.DataSource

@SpringBootApplication
class Application {
  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }

  @Bean
  @Primary
  @ConfigurationProperties(prefix="datasource.db_person")
  public DataSource personDataSource() {
    return DataSourceBuilder.create().build();
  }

  @Bean
  @ConfigurationProperties(prefix="datasource.db_contract")
  public DataSource contractDataSource() {
    return DataSourceBuilder.create().build();
  }
}
{% endhighlight %}

## DatabaseService.groovy

Here just an example how to set up jdcTemplates using the datasources.

{% highlight groovy%}
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.beans.factory.annotation.Qualifier
import org.springframework.jdbc.core.JdbcTemplate

import javax.sql.DataSource

abstract class DatabaseService {

  private JdbcTemplate personJdbcTemplate;
  private JdbcTemplate contractJdbcTemplate;

  @Autowired // primary datasource
  DataSource personDataSource

  @Autowired
  @Qualifier("contractDataSource")
  DataSource contractDataSource

  protected personJdbcTemplate() {
    if (personJdbcTemplate == null)
      personJdbcTemplate = new JdbcTemplate(personDataSource);
    return personJdbcTemplate
  }

  protected contractJdbcTemplate() {
    if (contractJdbcTemplate == null)
      contractJdbcTemplate = new JdbcTemplate(contractDataSource);
    return contractJdbcTemplate
  }
}
{% endhighlight %}
