---
layout: post
title: 升级spring-jdbc后发现的mysql驱动问题及解决 
---

近日，在一个变更里面把spring-jdbc从4.2.9升级到4.3.11，但升级后随即发现日志里大量出现报错：

```
Caused by: java.sql.SQLException: Streaming result set com.mysql.jdbc.RowDataDynamic@1f325975 is still active. No statements may be issued when any streaming result sets are open and in use on a given connection. Ensure that 
you have called .close() on any active streaming result sets before attempting more queries.
        at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:868) ~[mysql-connector-java-5.1.40.jar:5.1.40]
        at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:864) ~[mysql-connector-java-5.1.40.jar:5.1.40]
        at com.mysql.jdbc.MysqlIO.checkForOutstandingStreamingData(MysqlIO.java:3211) ~[mysql-connector-java-5.1.40.jar:5.1.40]
        at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:2443) ~[mysql-connector-java-5.1.40.jar:5.1.40]
        at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:2677) ~[mysql-connector-java-5.1.40.jar:5.1.40]
        at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2545) ~[mysql-connector-java-5.1.40.jar:5.1.40]
        at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2503) ~[mysql-connector-java-5.1.40.jar:5.1.40]
        at com.mysql.jdbc.StatementImpl.executeQuery(StatementImpl.java:1369) ~[mysql-connector-java-5.1.40.jar:5.1.40]
        at com.mysql.jdbc.Field.getCollation(Field.java:446) ~[mysql-connector-java-5.1.40.jar:5.1.40]
        at com.mysql.jdbc.ResultSetMetaData.isCaseSensitive(ResultSetMetaData.java:552) ~[mysql-connector-java-5.1.40.jar:5.1.40]
        at com.alibaba.druid.filter.FilterChainImpl.resultSetMetaData_isCaseSensitive(FilterChainImpl.java:4571) ~[druid-1.0.25.jar!/:na]
        at com.alibaba.druid.filter.FilterAdapter.resultSetMetaData_isCaseSensitive(FilterAdapter.java:2742) ~[druid-1.0.25.jar!/:na]
        at com.alibaba.druid.filter.FilterChainImpl.resultSetMetaData_isCaseSensitive(FilterChainImpl.java:4568) ~[druid-1.0.25.jar!/:na]
        at com.alibaba.druid.proxy.jdbc.ResultSetMetaDataProxyImpl.isCaseSensitive(ResultSetMetaDataProxyImpl.java:59) ~[druid-1.0.25.jar!/:na]
        at com.sun.rowset.CachedRowSetImpl.initMetaData(CachedRowSetImpl.java:722) ~[na:1.8.0_112]
        at com.sun.rowset.CachedRowSetImpl.populate(CachedRowSetImpl.java:639) ~[na:1.8.0_112]
        at org.springframework.jdbc.core.SqlRowSetResultSetExtractor.createSqlRowSet(SqlRowSetResultSetExtractor.java:82) ~[spring-jdbc-4.3.11.RELEASE.jar:4.3.11.RELEASE]
        at org.springframework.jdbc.core.SqlRowSetResultSetExtractor.extractData(SqlRowSetResultSetExtractor.java:65) ~[spring-jdbc-4.3.11.RELEASE.jar:4.3.11.RELEASE]
        at org.springframework.jdbc.core.SqlRowSetResultSetExtractor.extractData(SqlRowSetResultSetExtractor.java:46) ~[spring-jdbc-4.3.11.RELEASE.jar:4.3.11.RELEASE]
        at org.springframework.jdbc.core.JdbcTemplate$1.doInPreparedStatement(JdbcTemplate.java:697) ~[spring-jdbc-4.3.11.RELEASE.jar:4.3.11.RELEASE]
        at org.springframework.jdbc.core.JdbcTemplate.execute(JdbcTemplate.java:633) ~[spring-jdbc-4.3.11.RELEASE.jar:4.3.11.RELEASE]
        ... 8 common frames omitted
```

赶紧回滚，由于变更内容本身比较小（只改了前端vm模板），怀疑是升级spring-jdbc引起的问题，于是就先降级回来，果然问题消失。但秉着不挖坑及时填坑的原则，开始查原因。

首先把两个版本的spring-jdbc源代码扒下来做diff，果然发现有疑点，[JdbcTemplate#setFetchSize](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/jdbc/core/JdbcTemplate.html#setFetchSize-int-)这里写着：

> Note: As of 4.3, negative values other than -1 will get passed on to the driver, since e.g. MySQL supports special behavior for Integer.MIN_VALUE.

刚好出问题的代码调用的jdbcTemplate确实[设为了Integer.MIN_VALUE来做streaming](https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-reference-implementation-notes.html)，那就来验证一下，在测试环境，把setFetchSize的调用屏蔽掉，重启后，果然没再报错了。

不过这依然是个坑，只不过从spring-jdbc转移到了mysql驱动上，为啥streaming的时候就会报错，既然提供了支持，那就应该是可用的才对，于是继续找原因。在异常栈里找到核心判断代码，加上断点，果然抓到问题，在请求完正常的SELECT后，居然驱动内部又立刻用同一个连接发了一个`SHOW FULL COLUMNS FROM table`的查询，这当然是报错啦。

于是搜了下这个行为，发现遇到的人不少，例如[这里](http://ouyangshixiong.iteye.com/blog/1242050)和[这里](https://stackoverflow.com/questions/29451069/tons-of-generated-show-full-columns-from-queries-in-mysql-connector-j)，似乎只会在用到了CachedRowSetImpl时才会有这个问题，正常的解决方案是jdbcUrl里添加参数`useDynamicCharsetInfo=false`就好了。

然而，我们公司的jdbcUrl是远程推送下来的，并没有办法改，当然换成mybatis也是可以的，但这个jdbcTemplate有点特殊，当初就是要和已有的mybatis隔离开，才用jdbcTemplate来处理的。于是得找找别的解决方案。

把整个链路都翻了一遍，公司提供的DataSource没有任何接口可以修改url参数，mysql-connector也没有，但用到的[druid](https://github.com/alibaba/druid)连接池似乎有filter的能力，而且从断点的栈来看，也是在filter里调用到了判断大小写敏感的地方，最终调用到了问题代码。

![screenshot](/img/2017-09-22_131529.png)

而且filter可以通过System.properties里选择使用哪些filter，那就简单了，在web.xml里新增一个listener，在这个listener里设置一下`System.setProperty("druid.filters", "com.company.DruidMetaDataCaseInsensitiveFilter");`，创建对应的Filter类继承com.alibaba.druid.filter.FilterAdapter，并覆盖resultSetMetaData_isCaseSensitive方法，让其始终返回false（用到的数据库是ci的），再发布一次，果然在spring-jdbc 4.3.11和streaming场景下也不报错了。


