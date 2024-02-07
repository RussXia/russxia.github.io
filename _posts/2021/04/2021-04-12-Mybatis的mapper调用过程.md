---
layout: blog
title: "Mybatis的mapper调用过程"
catalog: true
tag: [Java,2021]
---
# Mybatis的初始化和Mapper调用过程
1. 加载全局配置xml
2. 加载指定的mapper(xml或者注解扫描)并解析,会将mapper添加到configuration中的`mapperRegistry`
  1. 解析过程中，所有的方法，会被解析成一个个的MappedStatement并保存到configuration中的`mappedStatements`中
  2. MappedStatemnt的id，xml中id是 namespace+"."+id，注解时的为 interfaceName+"."+methodName
  3. 保存MappedStatement的`mappedStatements`(Map)以id为key
3. 构建SqlSessionFactory
4. 通过SqlSessionFactory获取SqlSession
5. 通过SqlSession获取mapper(从mapperRegistry中获取，通过MapperProxyFactory一个MapperProxy,jdk动态代理)
6. 通过动态代理对象，调用接口方法
7. 拿到要执行的mappermethod
8. 拿到对应(类名+method)和语句类型(SqlCommand),使用sqlSession查询
9. 使用(类名+method)查询对应的MappedStatement
10. 使用executor查询
  1. mybatis主要有三种执行器，SimpleExecutor、ReuseExecutor、BatchExecutor
    + SimpleExecutor每次开启一个Statement，用完立即关闭
    + ReuseExecutor会以sql为key缓存Statement，如果有就复用
    + BatchExecutor在执行update时，将所有sql都添加到批处理中(addBatch())，等待统一执行(executeBatch())
11. executor使用StatementHandler查询
12. parameterHandler设置参数
13. statement查询结果



#### 1-3步初始化

1-3步主要是mybatis的初始化，包括构建Configuration、Environment、TransactionFactory、SqlSessionFactory等，

在xml构建过程中，在加载Configuration时会加载mybatis-config.xml配置，在mybatis-config.xml中会指定mapper文件地址，

configuration会添加这些mapper的MapperProxyFacotry代理对象，并将xml中的`<select>`、`<update>`等标签解析成MappedStatement，保存到mappedStatements(Map)中。

todo 调用时序图



在`mybatis-spring-boot-starter`加载过程中，`org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration`



#### 4-13步mapper调用查询

4-13步是整个mybatis调用mapper的整个过程。



todo 调用时序图



## Mybatis中的一些关键类解读

#### Configuration





#### Environment





#### SqlSessionFactory





#### SqlSession



#### MappedStatement





#### Executor



#### TypeHandler



#### ParameterHandler





#### ResultHandler







#### RowBounds





#### Interceptor







## Mybatis中的缓存



#### 一级缓存

`SESSION`、`STATEMENT`







#### 二级缓存

`CachingExecutor`

