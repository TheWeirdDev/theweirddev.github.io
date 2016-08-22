---
layout: post
title:  "Spring boot: YAML multiple datasources"
date:   2016-08-21 23:56:45
comments: true
toc: false
categories:
- blog
permalink: spring-yml-datasources
description: How to configure multiple datasources in Srping boot using YAML. (Updated)
---

> This post is an update from the original post created the 22nd of Nov 2015.

In this example we will see how to config two datasources on different environments (development, test, production)
using a YAML config file. Before starting, I added in my __pom.xml__ the __spring-boot-starter-jdbc__.

## application.yml

In order to represent the several environments I used profiles. After that, for each environment I set the properties concerning the datasources. We can 
notice the two datasources each time.

> Profiles are used to group beans together. For a specific profile the beans are created only if the profile is activated. 

{% highlight yaml%}
spring:
  profiles.active: development

---
spring:
  profiles: development
datasource:
  db-person:
      url: jdbc:oracle:thin:@db_person_dev
      username: username
      password: pwd
      driver-class-name: oracle.jdbc.OracleDriver
      test-on-borrow: true
      validation-query: SELECT 1 FROM dual
  db-contract:
      url: jdbc:oracle:thin:@db_contract_dev
      username: username
      password: pwd
      driver-class-name: oracle.jdbc.OracleDriver
      test-on-borrow: true
      validation-query: SELECT 1 FROM dual

---

spring:
  profiles: test
datasource:
  db-person:
      url: jdbc:oracle:thin:@db_person_test
      username: username
      password: pwd
      driver-class-name: oracle.jdbc.OracleDriver
      test-on-borrow: true
      validation-query: SELECT 1 FROM dual
  db-contract:
      url: jdbc:oracle:thin:@db_contract_test
      username: username
      password: pwd
      driver-class-name: oracle.jdbc.OracleDriver
      test-on-borrow: true
      validation-query: SELECT 1 FROM dual

---

spring:
  profiles: production
datasource:
  db-person:
      url: jdbc:oracle:thin:@db_person_prod
      username: username
      password: pwd
      driver-class-name: oracle.jdbc.OracleDriver
      test-on-borrow: true
      validation-query: SELECT 1 FROM dual
  db-contract:
      url: jdbc:oracle:thin:@db_contract_prod
      username: username
      password: pwd
      driver-class-name: oracle.jdbc.OracleDriver
      test-on-borrow: true
      validation-query: SELECT 1 FROM dual

---
{% endhighlight %}

## Application.groovy
 
Usually, Spring boot creates automatically a __datasource__ and a __jdbcTemplate__ when the __jdbc-starter__ is part of the dependencies. 
Behind the scene, it checks in classpath for libraries brought by the starters, based on the presence of certain libraries it will autoconfigure beans.
Spring boot is said as opinionated. Also, it is worth mentioning that Spring Boot reads the application properties to auto-configure the beans. This is how Spring Boot
 can create a datasource, but you have to respect the right names for the properties. 

In the case of multiple datasources Spring Boot can't guess that you actually want multiple datasources. Hopefully, it's possible to override Spring Boot behaviour and 
define these beans ourself.

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
  @ConfigurationProperties(prefix="datasource.db-person")
  public DataSource personDataSource() {
    return DataSourceBuilder.create().build();
  }

  @Bean
  @ConfigurationProperties(prefix="datasource.db-contract")
  public DataSource contractDataSource() {
    return DataSourceBuilder.create().build();
  }
  
  @Bean
  public JdbcTemplate personJdbcTemplate(){
    return new JdbcTemplate(personDataSource());
  }
  
  @Bean
  public JdbcTemplate contractJdbcTemplate(){
    return new JdbcTemplate(contractDataSource());
  }
  
  @Beans
  public PersonRepository jdbcPersonRepository() {
    PersonRepository personRepo = new JdbcPersonRepository();
    personRepo.setJdbcTemplate(personJdbcTemplate());
    return personRepo;
  }
  
  @Beans
  public ContractRepository jdbcContractRepository() {
    ContractRepository contractRepo = new JdbcContractRepository();
    contractRepo.setJdbcTemplate(contractJdbcTemplate());
    return contractRepo;
  }
}
{% endhighlight %}

