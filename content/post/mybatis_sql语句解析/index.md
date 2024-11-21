---
author: Ryan Yuan
title: 一条 SQL 语句是如何被解析的？
date: 2024-11-21
image: image.png
categories:
    - MyBatis 底层探秘
---

# 环境准备
mybatis.xml
```xml
<?xml version="1.0" encoding="UTF-8" ?>  
<!DOCTYPE configuration   PUBLIC "-//mybatis.org//DTD Config 3.0//EN"  "https://mybatis.org/dtd/mybatis-3-config.dtd">  
<configuration>  
  <environments default="development">  
    <environment id="development">  
      <transactionManager type="JDBC"/>  
      <dataSource type="POOLED">  
        <property name="driver" value="com.mysql.cj.jdbc.Driver"/>  
        <property name="url" value="jdbc:mysql://localhost:3306/user_db"/>  
        <property name="username" value="root"/>  
        <property name="password" value="my-secret-pw"/>  
      </dataSource>  
    </environment>  
  </environments>  
  <mappers>  
    <mapper resource="mapper/UserMapper.xml"/>  
  </mappers>  
</configuration>
```

Mapper 接口
```java
public interface UserMapper {  
  
    List<User> getUserList();  
  
}
```

User 实体类
```java
@Data  
@ToString  
public class User {  
  
    private Integer id;  
  
    private String name;  
  
    private Integer age;  
  
}
```

Mapper.xml
```java
<?xml version="1.0" encoding="UTF-8" ?>  
<!DOCTYPE mapper  
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  
        "https://mybatis.org/dtd/mybatis-3-mapper.dtd">  
<mapper namespace="com.anno.dao.UserMapper">  
    <select id="getUserList" resultType="com.anno.model.User">  
        select * from user_0  
    </select>  
</mapper>
```

# 解析 configuration 中的 mappers 标签
org.apache.ibatis.builder.xml.XMLConfigBuilder#mapperElement(XNode parent)
```java
private void mapperElement(XNode parent) throws Exception {  
  if (parent != null) {  
	// 遍历所有的 mapper 节点
    for (XNode child : parent.getChildren()) {  
      if ("package".equals(child.getName())) {  
        String mapperPackage = child.getStringAttribute("name");  
        configuration.addMappers(mapperPackage);  
      } else {  
        String resource = child.getStringAttribute("resource");  
        String url = child.getStringAttribute("url");  
        String mapperClass = child.getStringAttribute("class");  
        if (resource != null && url == null && mapperClass == null) {  
          ErrorContext.instance().resource(resource);  
          try(InputStream inputStream = Resources.getResourceAsStream(resource)) {  
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());  
            mapperParser.parse();  
          }  
        } else if (resource == null && url != null && mapperClass == null) {  
          ErrorContext.instance().resource(url);  
          try(InputStream inputStream = Resources.getUrlAsStream(url)){  
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());  
            mapperParser.parse();  
          }  
        } else if (resource == null && url == null && mapperClass != null) {  
          Class<?> mapperInterface = Resources.classForName(mapperClass);  
          configuration.addMapper(mapperInterface);  
        } else {  
          throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");  
        }  
      }  
    }  
  }  
}
```
MyBatis 可以使用 XML 作为配置文件，上面的代码是对配置文件中配置的 `<mappers>` 元素的解析。
如果元素是 `<package>`：
	提取 name 属性（一个包名）。
	调用 configuration.addMappers(mapperPackage)，自动扫描该包下的所有 Mapper 接口。
MyBatis 支持三种方式来定义 Mapper，它们是互斥的，逐次尝试来进行解析
	通过 resource 来配置
	通过 url 定义远程 xml
	通过 class 定义，直接加载 Mapper 接口类，通常用于加载通过**注解**配置 XML 的接口。

```xml
  <mappers>  
    <mapper resource="mapper/UserMapper.xml"/>  
  </mappers>  
```
其中 resource 方式是使用最多的一种，也就是在本地去编写 `*mapper.xml` 的方式，当获取到 `resource` 中定义的资源后，将输入流作为参数来构造 `XMLMapperBuilder`。
构造完成后，调用 `XMLMapperBuilder` 的 `parse` 方法解析接口映射 XML 文件。（上面解析的是 MyBatis 的配置文件）
关于 Mapper 配置文件的元素内容，可以看官方文档关于这一部分的说明：[https://mybatis.org/mybatis-3/sqlmap-xml.html](https://mybatis.org/mybatis-3/sqlmap-xml.html)
# 解析 Mapper.xml
org.apache.ibatis.builder.xml.XMLMapperBuilder#parse()
```java
public void parse() {  
// 检测该资源是否已经被加载
  if (!configuration.isResourceLoaded(resource)) {  
    configurationElement(parser.evalNode("/mapper"));  
    configuration.addLoadedResource(resource); // 标明该资源已经被加载，防止重复加载 
    bindMapperForNamespace();  
  }  
  
  parsePendingResultMaps();  
  parsePendingCacheRefs();  
  parsePendingStatements();  
}
```
## 解析 Mapper.xml 中的语句标签
### 具体源码
具体负责解析和存储 SQL 语句的是这个方法，其又创建了一个 `XMLStatementBuilder` 对象来解析语句。
org.apache.ibatis.builder.xml.XMLMapperBuilder#buildStatementFromContext
```java
private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {  
  for (XNode context : list) {  
    final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);  
    try {  
      statementParser.parseStatementNode();  
    } catch (IncompleteElementException e) {  
      configuration.addIncompleteStatement(statementParser);  
    }  
  }  
}
```
到目前为止，我们已经遇到了三个 `XML xxx Builder`，分别是 `XMLConfigBuilder`、`XMLMapperBuilder`、`XMLStatmentBuilder`，这样可以有效的降低代码的耦合度。

org.apache.ibatis.builder.xml.XMLStatementBuilder#parseStatementNode
```java
  public void parseStatementNode() {
    // ......

    // 配置处理 selectKey 和 sql 标签

    // 构建 SqlSource
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String resultType = context.getStringAttribute("resultType");
    Class<?> resultTypeClass = resolveClass(resultType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultSetType = context.getStringAttribute("resultSetType");
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
    if (resultSetTypeEnum == null) {
      resultSetTypeEnum = configuration.getDefaultResultSetType();
    }
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    String resultSets = context.getStringAttribute("resultSets");

    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered,
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
  }
```
上面的方法中，解析的是一个具体的语句节点，比如：
```xml
<insert id="insertUser" parameterType="User">
  INSERT INTO users (id, name) VALUES (#{id}, #{name})
</insert>
```
### 构建 SqlSource 对象
通过 `langDriver.createSqlSource` 将其转化为一个 SqlSource 对象：
org.apache.ibatis.scripting.xmltags.XMLLanguageDriver#createSqlSource
=> org.apache.ibatis.scripting.xmltags.XMLScriptBuilder#parseScriptNode
```java
public SqlSource parseScriptNode() {  
  MixedSqlNode rootSqlNode = parseDynamicTags(context);  
  SqlSource sqlSource;  
  if (isDynamic) {  
    sqlSource = new DynamicSqlSource(configuration, rootSqlNode);  
  } else {  
    sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);  
  }  
  return sqlSource;  
}
```
会根据语句中是否含有动态标签（例如 `<if>`、`<where>` 等）来决定构建动态的 SQLSource 还是静态的 SqlSource。
SqlSource 是构建语句中的第一个重要接口，用于表示和封装原始的 SQL 语句以及与之相关的信息：
```java
public interface SqlSource {  
  BoundSql getBoundSql(Object parameterObject);  
}
```
其具体的实现有这么几个：
```java
StaticSqlSource：用于处理静态 SQL，直接将 SQL 原样传递。
DynamicSqlSource：用于处理动态 SQL，支持 <if>、<foreach> 等标签。
RawSqlSource：用于处理原始 SQL，在编译时解析动态内容，运行时效率较高。
ProviderSqlSource：用于支持 SQL 提供者（如 @SelectProvider 注解）。
```
SqlSource 提供了一个核心方法，这个方法会返回一个 `BoundSql` 对象；
BoundSql 是 MyBatis 中用于封装 **最终生成的 SQL** 和相关信息的对象，通常由 SqlSource 的 getBoundSql 方法生成，BoundSql 有这么几个关键属性：
```java
private final String sql;  
private final List<ParameterMapping> parameterMappings;  
private final Object parameterObject;  
private final Map<String, Object> additionalParameters;  
private final MetaObject metaParameters;
```
- sql：最终生成的 SQL 字符串，例如：SELECT * FROM users WHERE id = ?。
- parameterMappings：表示 SQL 中的占位符参数与实际参数的映射关系（如 #{} 对应的 Java 对象属性）。
- parameterObject：原始的参数对象（如传入的实体类或 Map）。
- 用于存储动态 SQL 生成过程中引入的额外参数（如 foreach 标签的索引变量）。
### 构建 MappedStatement 对象
上面提到的 SqlSource 提供的是一个构建好的 SQL 语句，除此之外，一个 SQL 语句标签还有这些属性：
```java
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String resultType = context.getStringAttribute("resultType");
    Class<?> resultTypeClass = resolveClass(resultType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultSetType = context.getStringAttribute("resultSetType");
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
    if (resultSetTypeEnum == null) {
      resultSetTypeEnum = configuration.getDefaultResultSetType();
    }
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    String resultSets = context.getStringAttribute("resultSets");
```
这些属性和 SqlSource 一起被封装成一个 MappedStatment 对象并添加到 Configuration 中。
```java
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered,
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
```

所以，一个 SQL 标签最后会被解析成一个 MappedStatment 对象，其以 ID 为 KEY，存储在 Configuration 的 mappedStatements 属性中。
```java
public void addMappedStatement(MappedStatement ms) {  
  mappedStatements.put(ms.getId(), ms);  
}
```
id 的具体格式为 Mapper 的全类名 + 方法名，之后 Mapper 对象如果需要执行 SQL 语句，就可以通过全类名 + 方法名的方式获取到 MappedStatement 对象。
org.apache.ibatis.session.defaults.DefaultSqlSession#selectList
```java
private <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {  
  try {  
    MappedStatement ms = configuration.getMappedStatement(statement);  // 通过 id 获取 MappedStatment
    return executor.query(ms, wrapCollection(parameter), rowBounds, handler);  
  } catch (Exception e) {  
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);  
  } finally {  
    ErrorContext.instance().reset();  
  }  
}
```