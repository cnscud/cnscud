---
layout: post
title: MyBatis使用Zookeeper保存数据库的配置,可动态刷新
date:   2021-08-05 15:35
description: MyBatis使用Zookeeper保存数据库的动态配置
categories: mybatis
comments: true
tags:
- mybatis,zookeeper
---

核心关键点: 封装一个DataSource, 重写 getConnection 就可以实现

我们一步一步来看.

环境: Spring Cloud + MyBatis

## MyBatis常规方式下配置数据源: 使用Spring的Configuration
```java


package com.cnscud.cavedemo.fundmain.config;


import com.cnscud.xpower.dbn.SimpleDBNDataSourceFactory;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionFactoryBean;
import org.mybatis.spring.SqlSessionTemplate;
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.transaction.PlatformTransactionManager;

import javax.sql.DataSource;

/**
 * Database Config 多数据源配置: 主数据源.
 *
 * @author Felix Zhang 2021-08-02 17:30
 * @version 1.0.0
 */
@Configuration
@MapperScan(basePackages = {"com.cnscud.cavedemo.fundmain.dao"},
        sqlSessionFactoryRef = "sqlSessionFactoryMainDataSource")
public class MainDataSourceConfig {



    //常规配置: 使用application.yml里面的配置.
    @Primary
    @Bean(name = "mainDataSource")
    @ConfigurationProperties("spring.datasource.main")
    public DataSource mainDataSource() throws Exception {
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean(name = "sqlSessionFactoryMainDataSource")
    public SqlSessionFactory sqlSessionFactoryMainDataSource(@Qualifier("mainDataSource") DataSource mainDataSource) throws Exception {
        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
        //org.apache.ibatis.session.Configuration configuration = new org.apache.ibatis.session.Configuration();
        //configuration.setMapUnderscoreToCamelCase(true);
        //factoryBean.setConfiguration(configuration);
        factoryBean.setConfigLocation(new PathMatchingResourcePatternResolver().getResource("classpath:mybatis-config.xml"));

        // 使用mainDataSource数据源, 连接mainDataSource库
        factoryBean.setDataSource(mainDataSource);

        //下边两句仅仅用于*.xml文件，如果整个持久层操作不需要使用到xml文件的话（只用注解就可以搞定），则不加
        //指定entity和mapper xml的路径
        //factoryBean.setTypeAliasesPackage("com.cnscud.cavedemo.fundmain.model");
        factoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:com/cnscud/cavedemo/fundmain/mapper/*.xml"));
        return factoryBean.getObject();
    }

    @Primary
    @Bean
    public SqlSessionTemplate sqlSessionTemplateMainDataSource(@Qualifier("sqlSessionFactoryMainDataSource") SqlSessionFactory sqlSessionTemplateMainDataSource) throws Exception {

        //使用注解中配置的Factory
        return new SqlSessionTemplate(sqlSessionTemplateMainDataSource);
    }

    @Primary
    @Bean
    public PlatformTransactionManager mainTransactionManager(@Qualifier("mainDataSource") DataSource prodDataSource) {
        return new DataSourceTransactionManager(prodDataSource);
    }
}

```

这里面获取数据源的关键函数是 mainDataSource, 我们自己来实现就好了:

因为这个是一次性的工作, 所以我们无法修改DataSource的指向, 只能在DataSource内部做文章, 所以我们需要自己实现一个DataSource.


其中的步骤比较多, 我们来看看最终结果:

## 最终的DataSourceWrapper 
它完全封装了一个DataSource, 自己并没有任何DataSource的功能:

```java
package com.cnscud.xpower.dbn;

import javax.sql.DataSource;
import java.io.PrintWriter;
import java.sql.Connection;
import java.sql.SQLException;
import java.sql.SQLFeatureNotSupportedException;
import java.util.logging.Logger;

/**
 * Datasource wrapper, 为了方便动态创建DataSource.
 *
 * @author Felix Zhang 2021-08-05 14:14
 * @version 1.0.0
 */
public class DynamicByZookeeperDataSourceWrapper implements DataSource {

    protected SimpleDBNConnectionPool simpleDBNConnectionPool;
    protected String bizName;

    public DynamicByZookeeperDataSourceWrapper(SimpleDBNConnectionPool simpleDBNConnectionPool, String bizName) {
        this.simpleDBNConnectionPool = simpleDBNConnectionPool;
        this.bizName = bizName;
    }

    protected DataSource pickDataSource() throws SQLException{
        return simpleDBNConnectionPool.getDataSource(bizName);
    }

    @Override
    public Connection getConnection() throws SQLException {
        return pickDataSource().getConnection();
    }

    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        return pickDataSource().getConnection(username, password);
    }

    @Override
    public <T> T unwrap(Class<T> iface) throws SQLException {
        return pickDataSource().unwrap(iface);
    }

    @Override
    public boolean isWrapperFor(Class<?> iface) throws SQLException {
        return pickDataSource().isWrapperFor(iface);
    }

    @Override
    public PrintWriter getLogWriter() throws SQLException {
        return pickDataSource().getLogWriter();
    }

    @Override
    public void setLogWriter(PrintWriter out) throws SQLException {
        pickDataSource().setLogWriter(out);
    }

    @Override
    public void setLoginTimeout(int seconds) throws SQLException {
        pickDataSource().setLoginTimeout(seconds);
    }

    @Override
    public int getLoginTimeout() throws SQLException {
        return pickDataSource().getLoginTimeout();
    }

    @Override
    public Logger getParentLogger() throws SQLFeatureNotSupportedException {
        throw new SQLFeatureNotSupportedException();
    }
}

```

## SimpleDBNConnectionPool 

支持多个数据源的暂存池, 可以根据name获取不同的数据库DataSource实例:

这个类负责创建DataSource, 保存在Map里. 并且能监听Zookeeper的变化, 一旦侦听到变化, 就close现有的DataSource.

```java
package com.cnscud.xpower.dbn;

import com.github.zkclient.IZkDataListener;
import com.zaxxer.hikari.HikariDataSource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

import static java.lang.String.format;

/**
 * The simple datasource pool.
 *
 * 根据名字存放多个数据库的DataSource, 并且会监听Zookeeper配置, 动态重建.
 *
 * @author adyliu (imxylz@gmail.com)
 * @since 2011-7-27
 */
public class SimpleDBNConnectionPool {

    final Logger logger = LoggerFactory.getLogger(getClass());


    private Map<String, DataSource> instances = new ConcurrentHashMap<>();
    private final Set<String> watcherSchema = new HashSet<String>();


    public DataSource getInstance(String bizName) {
        try {
            return findDbInstance(bizName);
        }
        catch (SQLException e) {
            e.printStackTrace();
        }
        return null;
    }

    public Connection getConnection(String bizName) throws SQLException {
        DataSource ds = getDataSource(bizName);
        return ds.getConnection();
    }

    public DataSource getDataSource(String bizName) throws SQLException {
        return findDbInstance(bizName);
    }


    protected void destroyInstance(final String bizName) {
        synchronized (instances) {
            DataSource oldInstanceIf = instances.remove(bizName);
            logger.warn(format("destoryInstance %s and %s", bizName, oldInstanceIf != null ? "close datasource" : "do nothing"));
            if (oldInstanceIf != null) {
                closeDataSource(oldInstanceIf);
            }
        }
    }

    protected void closeDataSource(DataSource ds) {
        if (ds instanceof HikariDataSource) {
            try {
                ((HikariDataSource) ds).close();
            }
            catch (Exception e) {
                logger.error("Close datasource failed. ", e);
            }
        }
    }


    private DataSource createInstance(Map<String, String> dbcfg) {
        return new SimpleDataSourceBuilder().buildDataSource(dbcfg);
    }


    private DataSource findDbInstance(final String bizName) throws SQLException {
        DataSource ins = instances.get(bizName);
        if (ins != null) {
            return ins;
        }
        synchronized (instances) {// 同步操作
            ins = instances.get(bizName);
            if (ins != null) {
                return ins;
            }
            boolean success = false;
            try {
                Map<String, String> dbcfg = SchemeNodeHelper.getInstance(bizName);
                if (dbcfg == null) {
                    throw new SQLException("No such datasouce: " + bizName);
                }
                ins = createInstance(dbcfg);
                //log.warn("ins put "+ins);
                instances.put(bizName, ins);


                if (watcherSchema.add(bizName)) {
                    SchemeNodeHelper.watchInstance(bizName, new IZkDataListener() {

                        public void handleDataDeleted(String dataPath) throws Exception {
                            logger.warn(dataPath + " was deleted, so destroy the bizName " + bizName);
                            destroyInstance(bizName);
                        }

                        public void handleDataChange(String dataPath, byte[] data) throws Exception {
                            logger.warn(dataPath + " was changed, so destroy the bizName " + bizName);
                            destroyInstance(bizName);
                        }
                    });
                }
                success = true;
            }
            catch (SQLException e) {
                throw e;
            }
            catch (Throwable t) {
                throw new SQLException("cannot build datasource for bizName: " + bizName, t);
            }
            finally {
                if (!success) {
                    instances.remove(bizName);
                }
            }
        }
        return ins;
    }

}

```

## 真正创建DataSource的代码:

```java
package com.cnscud.xpower.dbn;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.apache.commons.lang.StringUtils;

import java.util.Map;

/**
 * Hikari DataSource.
 *
 * 思考: 可以根据参数里面的类型来使用不同的库创建DataSource, 例如Druid. (默认为HikariDataSource)
 *
 *
 * @author Felix Zhang 2021-08-05 11:14
 * @version 1.0.0
 */
public class SimpleDataSourceBuilder {


    public HikariDataSource buildDataSource(Map<String, String> args) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(getUrl(args));
        config.setUsername(args.get("username"));
        config.setPassword(args.get("password"));
        config.setDriverClassName(getDriverClassName(args));

        String maximumPoolSizeKey = "maximum-pool-size";
        int maximumPoolSize = 30;
        if(StringUtils.isNotEmpty(args.get(maximumPoolSizeKey))){
            maximumPoolSize = Integer.parseInt(args.get(maximumPoolSizeKey));
        }

        config.addDataSourceProperty("cachePrepStmts", "true"); //是否自定义配置，为true时下面两个参数才生效
        config.addDataSourceProperty("prepStmtCacheSize", maximumPoolSize); //连接池大小默认25，官方推荐250-500
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048"); //单条语句最大长度默认256，官方推荐2048
        config.addDataSourceProperty("useServerPrepStmts", "true"); //新版本MySQL支持服务器端准备，开启能够得到显著性能提升
        config.addDataSourceProperty("useLocalSessionState", "true");
        config.addDataSourceProperty("useLocalTransactionState", "true");
        config.addDataSourceProperty("rewriteBatchedStatements", "true");
        config.addDataSourceProperty("cacheResultSetMetadata", "true");
        config.addDataSourceProperty("cacheServerConfiguration", "true");
        config.addDataSourceProperty("elideSetAutoCommits", "true");
        config.addDataSourceProperty("maintainTimeStats", "false");

        config.setMaximumPoolSize(maximumPoolSize); //
        config.setMinimumIdle(10);//最小闲置连接数，默认为0
        config.setMaxLifetime(600000);//最大生存时间
        config.setConnectionTimeout(30000);//超时时间30秒
        config.setIdleTimeout(60000);

        config.setConnectionTestQuery("select 1");

        return new HikariDataSource(config);
    }

    private String getDriverClassName(Map<String, String> args) {
        return args.get("driver-class-name");
    }

    private String getUrl(Map<String, String> args) {
        return args.get("jdbc-url") == null ? args.get("url"): args.get("jdbc-url");
    }
}

```

## 为了方便读取Zookeeper节点, 还有个SchemeNodeHelper:

支持两种配置文件的方式 json或者Properties格式:


```java
package com.cnscud.xpower.dbn;

import com.cnscud.xpower.configcenter.ConfigCenter;
import com.cnscud.xpower.utils.Jsons;
import com.github.zkclient.IZkDataListener;
import org.apache.commons.lang.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.io.StringReader;
import java.util.HashMap;
import java.util.Map;
import java.util.Properties;

/**
 * 从Zookeeper的 /xpower/dbn节点下读取数据库配置.
 * 内容支持两种格式: json或者properties格式.
 *
 * JSON格式如下:
 * {
 *   "jdbc-url": "jdbc:mysql://127.0.0.1:3306/cavedemo?useUnicode=true&characterEncoding=utf8&useSSL=false&serverTimezone=UTC",
 *   "username": "dbuser",
 *   "password": "yourpassword",
 *   "driver-class-name": "com.mysql.cj.jdbc.Driver"
 * }
 *
 * Properties格式如下:
 *  jdbc-url: jdbc:mysql://127.0.0.1:3306/cavedemo?useUnicode=true&characterEncoding=utf8&useSSL=false&serverTimezone=UTC
 *  username: dbuser
 *  password: password
 *  driver-class-name: com.mysql.cj.jdbc.Driver
 *
 * @author Felix Zhang
 * @since 2021-8-5
 */
public class SchemeNodeHelper {

    static final Logger logger = LoggerFactory.getLogger(SchemeNodeHelper.class);

    //支持两种格式: json, properties
    public static Map<String, String> getInstance(final String instanceName) throws Exception {
        String data = ConfigCenter.getInstance().getDataAsString("/xpower/dbn/" + instanceName);
        if(StringUtils.isEmpty(data)){
            return null;
        }

        data = data.trim();
        if (data.startsWith("{")) {
            //as json
            Map<String, String> swap = Jsons.fromJson(data, Map.class);
            Map<String, String> result = new HashMap<>();

            if (swap != null) {
                for (String name : swap.keySet()) {
                    result.put(name.toLowerCase(), swap.get(name));
                }
            }

            return result;
        }
        else {
            //as properties
            Properties props = new Properties();
            try {
                props.load(new StringReader(data));
            }
            catch (IOException e) {
                logger.error("loading global config failed", e);
            }

            Map<String, String> result = new HashMap<>();

            for (String name : props.stringPropertyNames()) {
                result.put(name.toLowerCase(), props.getProperty(name));
            }

            return result;
        }
    }

    public static void watchInstance(final String bizName, final IZkDataListener listener) {
        final String path = "/xpower/dbn/" + bizName;
        ConfigCenter.getInstance().subscribeDataChanges(path, listener);
    }
}

```


## 实际应用
最后在MyBatis项目中, 替换原有MainDataSource代码为:

```java
    /**
     * 添加@Primary注解，设置默认数据源，事务管理器.
     * 此处使用了一个可以动态重建的DataSource, 如果Zookeeper配置改变,会动态重建.
     */
    @Primary
    @Bean(name = "mainDataSource")
    public DataSource mainDataSource() throws Exception {
        return SimpleDBNDataSourceFactory.getInstance().getDataSource("cavedemo");
    }

```

运行项目, 发现可以连上数据库, 并且不重启项目的情况下, 动态修改数据库配置, 能自动重连.


## 项目代码: 
    https://github.com/cnscud/xpower/tree/main/xpower-main/src/main/java/com/cnscud/xpower/dbn

其中用到的 ConfigCenter 也在这个项目里, 也可以自己实现, 就可以脱离本项目了.

