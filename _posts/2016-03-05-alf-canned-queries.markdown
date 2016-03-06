---
layout: post
title:  "Alfresco canned queries"
date:   2016-03-06 23:56:45
comments: true
toc: false
categories:
- blog
permalink: alf-canned-queries
description: Introduction to Alfresco canned queries.
---

Canned queries are SQL queries executed on the Alfresco database and not on Solr. So If you want to perform a full-text search or having spelling correction or facets then you're not in the right place. One main reason to use canned
 queries
 is that Solr is not transactional, it's near real time, but if you want to assure having exactly what is in the DB then you will need
 canned queries. Behind the scene, Alfresco uses a persistence framework called Ibatis which replaced hibernate some time ago.

## Use Case

Let's study a simple use case. You want to get all nodeRefs for a custom type. And since you millions nodes you want to be able to use
paging.

## Ibatis XML files

`custom-sqlMap.xml`
{% highlight xml%}
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="alfresco.query.custom">

  <resultMap id="result_NodeRef" type="Node">
    <result property="id" column="id" jdbcType="BIGINT" javaType="java.lang.Long"/>
    <result property="store.protocol" column="protocol" jdbcType="VARCHAR" javaType="java.lang.String"/>
    <result property="store.identifier" column="identifier" jdbcType="VARCHAR" javaType="java.lang.String"/>
    <result property="uuid" column="uuid" jdbcType="VARCHAR" javaType="java.lang.String"/>
  </resultMap>

  <select id="select_GetCustomNodesCQ" parameterType="Node" resultMap="result_NodeRef">
     select
      node.id             as id,
      store.protocol      as protocol,
      store.identifier    as identifier,
      node.uuid           as uuid
    from
      alf_node node
      join alf_store store on (store.id = node.store_id)
      join alf_transaction txn on (txn.id = node.transaction_id)
    where
      store.protocol = #{store.protocol} and
      store.identifier = #{store.identifier} and
      node.type_qname_id = #{typeQNameId}
  </select>
</mapper>
{% endhighlight %}

> * This xml file is used to defined the SQL stored procedures, that's the way Ibatis works. The file is called a __mapper__.
* We can see there is a new __namespace__ declared.
* For our case there is only one simple select. It has:
    * an id.
    * a parameter type "Node".
    * a result Map "result_NodeRef".
* The mapping of the "result_NodeRef" has been defined at the begin of the file.
* The parts of the where clause like this #{parameter} are properties taken from the parameter. Node is an alias to a java
class.

`custom-sqlMapConfig.xml`
{% highlight xml%}
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

  <typeAliases>
    <typeAlias alias="Node" type="org.alfresco.repo.domain.node.NodeEntity"/>
  </typeAliases>

  <mappers>
    <mapper resource="alfresco/module/custom/ibatis/custom-sqlMap.xml"/>
  </mappers>

</configuration>
{% endhighlight %}

> * This configuration file is used to "reference" the mapper files, we can see our.
* Aliases are defined here as well. You can see the parameter type "Node" from our select.

## Java classes: canned query and factory

`GetCustomNodesCQ.java`
{% highlight java%}
public class GetCustomNodesCQ extends AbstractCannedQuery<NodeEntity>
{
  public static final String CUSTOM_QUERY_NAMESPACE = "alfresco.query.custom";
  public static final String QUERY_SELECT_CUSTOM_NODES = "select_GetCustomNodesCQ";
  private CannedQueryDAO cannedQueryDAO;

  protected GetCustomNodesCQ(CannedQueryDAO cannedQueryDAO, CannedQueryParameters parameters)
  {
    super(parameters);
    this.cannedQueryDAO = cannedQueryDAO;
  }

  @Override
  protected List<NodeEntity> queryAndFilter(CannedQueryParameters parameters)
  {
    Object paramBeanObj = parameters.getParameterBean();

    // execute the query
    List<NodeEntity> results = cannedQueryDAO.executeQuery(CUSTOM_QUERY_NAMESPACE, QUERY_SELECT_CUSTOM_NODES, paramBeanObj, 0, Integer.MAX_VALUE);

    return results;
  }
}
{% endhighlight %}

> * This class allows to execute the canned query we wrote in the XML. You have to pass the namespace and the id of the query to the
executeQuery method, and voilà.
* There is another abstract class AbstractCannedQueryPermissions you can extend if you want to apply permissions.

`GetCustomNodesCQFactory.java`
{% highlight java%}
public class GetCustomNodesCQFactory extends AbstractCannedQueryFactory<NodeEntity>
{
  protected CannedQueryDAO cannedQueryDAO;

  protected QNameDAO qnameDAO;

  @Override
  public CannedQuery<NodeEntity> getCannedQuery(CannedQueryParameters parameters)
  {
    return new GetCustomNodesCQ(cannedQueryDAO, parameters);
  }

  public CannedQuery getCannedQuery(PagingRequest pagingRequest)
  {
    NodeEntity param = new NodeEntity();
    // Be aware that getQName will return null if there is no node of the custom type.
    param.setTypeQNameId(qnameDAO.getQName(CustomModel.CUSTOM_TYPE).getFirst());
    StoreEntity storeEntity = new StoreEntity();
    storeEntity.setIdentifier(StoreRef.STORE_REF_WORKSPACE_SPACESSTORE.getIdentifier());
    storeEntity.setProtocol(StoreRef.STORE_REF_WORKSPACE_SPACESSTORE.getProtocol());
    param.setStore(storeEntity);

    // Page details
    CannedQueryPageDetails pageDetails = createCQPageDetails(pagingRequest);

    // create query params holder
    int requestTotalCountMax = pagingRequest.getRequestTotalCountMax();
    CannedQueryParameters params =
      new CannedQueryParameters(param, pageDetails, null, requestTotalCountMax, pagingRequest.getQueryExecutionId());

    // return canned query instance
    return getCannedQuery(params);
  }

  protected CannedQueryPageDetails createCQPageDetails(PagingRequest pagingReq)
  {
    int skipCount = pagingReq.getSkipCount();
    if (skipCount == -1)
    {
      skipCount = CannedQueryPageDetails.DEFAULT_SKIP_RESULTS;
    }

    int maxItems = pagingReq.getMaxItems();
    if (maxItems == -1)
    {
      maxItems = CannedQueryPageDetails.DEFAULT_PAGE_SIZE;
    }

    // page details
    CannedQueryPageDetails pageDetails = new CannedQueryPageDetails(skipCount, maxItems);
    return pageDetails;
  }

  public void setCannedQueryDAO(CannedQueryDAO cannedQueryDAO)
  {
    this.cannedQueryDAO = cannedQueryDAO;
  }

  public void setQnameDAO(QNameDAO qnameDAO)
  {
    this.qnameDAO = qnameDAO;
  }
}
{% endhighlight %}

> A canned query can be used only once. So each time you want to execute it then you need a new instance. That is the job of the canned
 query factory.

## Java custom service

`CustomCannedQueryService.java`
{% highlight java%}
@Service
public class CustomCannedQueryServiceImpl implements CustomCannedQueryService
{
  @Autowired
  @Qualifier("getCustomNodeCannedQueryFactory")
  private GetCustomNodesCQFactory customNodesCQFactory;

  @Override
  public CannedQueryResults<NodeEntity> getCustomNodes(int skipCount, int maxItemPerPage)
  {
    GetCustomNodesCQ cannedQuery = (GetCustomNodesCQ) customNodesCQFactory.getCannedQuery(new PagingRequest(skipCount, maxItemPerPage));

    return cannedQuery.execute();
  }
}
{% endhighlight %}

> As we can see it is a bit complex, all of these xml files and java classes. However to work better with it you can encapsulate the
call to the factory and the execution of the query in a nice service.

## How to use it

{% highlight java%}
int skip = 0;
int maxItemPerPage = 5;
CannedQueryResults<NodeEntity> pagingResults = null;
List<NodeRef> toReturn = Lists.newArrayList();
do
{
  pagingResults = customCannedQueryService.getCustomNodes(skip, maxItemPerPage);

  for (NodeEntity nodeEntity : pagingResults.getPage())
  {
    NodeRef nodeRef = nodeEntity.getNodeRef();
    toReturn.add(nodeRef);
  }

  if(pagingResults.hasMoreItems()){
    skip += maxItemPerPage;
  }
} while (pagingResults.hasMoreItems());
{% endhighlight %}

> This example shows how to use the paging feature. It's possible to get all nodes in a single page if
"skip" is set to 0 and "maxItemPerPage" to Integer.MAX_VALUE.

## Spring context

`custom-ibatis-context.xml`
{% highlight xml%}
<?xml version='1.0' encoding='UTF-8'?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">

  <!-- Custom session factory to load the Ibatis configuration file -->
  <bean id="customSqlSessionFactory" class="org.alfresco.ibatis.HierarchicalSqlSessionFactoryBean">
    <property name="resourceLoader" ref="dialectResourceLoader"/>
    <property name="dataSource" ref="dataSource"/>
    <property name="configLocation">
      <value>classpath:alfresco/module/custom/ibatis/custom-sqlMapConfig.xml</value>
    </property>
  </bean>

  <!-- Custom session template to be used in the canned query dao -->
  <bean id="customSqlSessionTemplate" class="org.mybatis.spring.SqlSessionTemplate">
    <constructor-arg index="0" ref="customSqlSessionFactory"/>
  </bean>

  <!-- Your custom dao which is used to execute the stored procedures -->
  <bean id="customCannedQueryDAO" class="org.alfresco.repo.domain.query.ibatis.CannedQueryDAOImpl"
        init-method="init">
    <property name="sqlSessionTemplate" ref="customSqlSessionTemplate"/>
    <property name="controlDAO" ref="controlDAO"/>
  </bean>

  <!-- Create a new registry to contain your custom canned queries  -->
  <bean id="customCannedQueryRegistry" class="org.alfresco.util.registry.NamedObjectRegistry">
    <property name="storageType" value="org.alfresco.query.CannedQueryFactory"/>
  </bean>

  <!-- Custom Canned Query Factory -->
  <bean name="getCustomNodeCannedQueryFactory" class="com.custom.ibatis.GetCustomNodesCQFactory">
    <property name="registry" ref="customCannedQueryRegistry"/>
    <property name="qnameDAO" ref="qnameDAO"/>
    <property name="cannedQueryDAO" ref="customCannedQueryDAO"/>
  </bean>

</beans>
{% endhighlight %}

## End

I hope this was useful. It's a basic example but at the same time you have now a nice overview of what are canned queries. Also, I
think there is enough vocabulary in this page for you to look for other examples in the Alfresco source code.
