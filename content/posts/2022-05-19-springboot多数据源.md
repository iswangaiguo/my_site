---
title: SpringBoot多数据源
date: 2022-05-19T14:01:41+00:00
categories:
  - Java
  - 编程

---

在某些场景下，系统可能不止一个数据库，需要根据实际需要切换数据源访问不同的数据库，在Spring框架下为我们提供了`org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource`类用于设置多数据源和指定切换规则。

## 基本配置

### 数据库配置

首先新建两个数据库，这是直接使用docker启动两个测试数据库

```shell
docker ps
CONTAINER ID   IMAGE                COMMAND                  CREATED        STATUS                PORTS                                     NAMES
98c5b1d4ab69   postgres             "docker-entrypoint.s…"   2 days ago     Up 2 days             0.0.0.0:5432->5432/tcp                    postgres-db
88723934d79a   mysql/mysql-server   "/entrypoint.sh --ch…"   9 months ago   Up 5 days (healthy)   0.0.0.0:3306->3306/tcp, 33060-33061/tcp   mysql
```

分别在Mysql和Postgres新建两个测试用库master和slave，并各自新建test_table表，插入数据

```sql
-- mysql
DROP TABLE IF EXISTS `test_table`;
CREATE TABLE `test_table` (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

INSERT INTO `test_table` VALUES (1, 'master');

-- postgresql
DROP TABLE IF EXISTS "public"."test_table";
CREATE TABLE "public"."test_table" (
  "id" int4 NOT NULL,
  "name" varchar(255) COLLATE "pg_catalog"."default"
);

INSERT INTO "public"."test_table" VALUES (1, 'slave');
```

## 项目配置

项目整体目录结构如下：

```shell
tree
.
├── HELP.md
├── mvnw
├── mvnw.cmd
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── datasource
    │   │           └── demo
    │   │               ├── DemoApplication.java
    │   │               ├── config
    │   │               │   ├── DataSourceConfig.java
    │   │               │   └── DynamicDataSource.java
    │   │               ├── entity
    │   │               │   └── DataEntity.java
    │   │               └── mapper
    │   │                   └── DataMapper.java
    │   └── resources
    │       └── application.yml
    └── test
        └── java
            └── com
                └── datasource
                    └── demo
                        └── DemoApplicationTests.java

15 directories, 11 files
```

pom.xml配置如下，主要引入了springboot-start、数据库驱动、lombok、druid、logback等依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.6.7</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.datasource</groupId>
	<artifactId>demo</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>demo</name>
	<description>Demo project for Spring Boot</description>
	<properties>
		<java.version>8</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>2.2.2</version>
		</dependency>

		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.postgresql</groupId>
			<artifactId>postgresql</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
			<version>1.2.8</version>
        </dependency>
		<dependency>
			<groupId>ch.qos.logback</groupId>
			<artifactId>logback-classic</artifactId>
	  </dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<configuration>
					<excludes>
						<exclude>
							<groupId>org.projectlombok</groupId>
							<artifactId>lombok</artifactId>
						</exclude>
					</excludes>
				</configuration>
			</plugin>
		</plugins>
	</build>

</project>
```

application.yml主要对数据源进行配置

```yaml
spring:
    datasource:
        druid:
            master:
                url: jdbc:mysql://127.0.0.1:3306/master?useUnicode=true&characterEncoding=utf-8&useSSL=true&serverTimezone=UTC&useAffectedRows=true
                username: root
                password: 123456
                driver-class-name: com.mysql.cj.jdbc.Driver
            slave:
                url: jdbc:postgresql://127.0.0.1:5432/slave?useUnicode=true&characterEncoding=utf-8&useSSL=true&serverTimezone=UTC&useAffectedRows=true
                username: root
                password: 123456
                driver-class-name: org.postgresql.Driver
```

接下来在DataSorceConfig对数据源进行注入

```java
package com.datasource.demo.config;

import java.util.HashMap;
import java.util.Map;

import javax.sql.DataSource;

import com.alibaba.druid.pool.DruidDataSource;
import com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceBuilder;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

@Configuration
public class DataSourceConfig {
    
    @Bean
    @ConfigurationProperties("spring.datasource.druid.master")
    public DataSource masterDataSource() {
        return DruidDataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.druid.slave")
    public DataSource slaveDataSource() {
        return DruidDataSourceBuilder.create().build();
    }

    @Bean
    @Primary
    public DynamicDataSource dataSource(Map<String, DruidDataSource> druidDataSourceMap) {
        Map<String, DataSource> dataSourceMap = new HashMap<>(druidDataSourceMap);
        return new DynamicDataSource(dataSourceMap);
    }

}
```

这里设置ConfigurationProperties的时候很多人会新建一个配置类作为参数接受数据库配置，这是没有必要的，这里返回的DataSource对象会自动注入配置信息。我们将DynamicDataSource标注为@Primary，在Autowired等注解注入数据源首先会选择该对象，DynamicDataSource是数据源切换的关键。

DynamicDataSource配置类如下：

```java
package com.datasource.demo.config;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.atomic.AtomicInteger;

import javax.sql.DataSource;

import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@RequiredArgsConstructor
@Slf4j
public class DynamicDataSource extends AbstractRoutingDataSource {


    private final AtomicInteger count = new AtomicInteger();
    private final Map<String, DataSource> targetDataSources;

    @Override
    protected Object determineCurrentLookupKey() {
        int i = count.incrementAndGet();
        List<String> list = new ArrayList<>(targetDataSources.keySet());
        String dataSource = list.get(i % list.size());
        // String dataSource = list.get(1);
        log.info(">>>>>> 当前数据库: {} <<<<<<", dataSource);
        return dataSource;
    }
    
    @Override
    public void afterPropertiesSet() {
        super.setTargetDataSources(new HashMap<>(targetDataSources));
        super.afterPropertiesSet();
    }
    
}
```

该类主要继承AbstractRoutingDataSource类，并重写determineCurrentLookupKey和afterPropertiesSet两个方法，afterPropertiesSet将候选数据源设置到成员变量，determineCurrentLookupKey选择返回的数据源，我们选择了一个简单的轮询方式返回数据源。

## 测试

```java
package com.datasource.demo;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@MapperScan("com.datasource.demo.mapper")
@SpringBootApplication
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

}
```

在启动类配置Mapper扫描路径，自动装配Mapper，这里为了简化操作，使用注解的方式绑定SQL

```java
package com.datasource.demo.mapper;

import com.datasource.demo.entity.DataEntity;

import org.apache.ibatis.annotations.Select;

public interface DataMapper {

    @Select("select * from test_table where id = 1")
    DataEntity getData();
    
}
```

上述代码只做测试，一般在实际应用中很少采用这种轮询的方式，更多的是通过注解的方式指定数据源进行返回。

```java
package com.datasource.demo;

import java.util.stream.IntStream;

import com.datasource.demo.mapper.DataMapper;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import lombok.extern.slf4j.Slf4j;

@SpringBootTest
@Slf4j
class DemoApplicationTests {

	@Autowired
	private DataMapper dataMapper;

	@Test
	void testDataSource() throws Exception {
		IntStream.range(0, 10).forEach(i -> log.info(dataMapper.getData().toString()));
	}

}

//输出
2022-05-19 21:57:39.737  INFO 46758 --- [           main] c.datasource.demo.DemoApplicationTests   : DataEntity(id=1, name=slave)
2022-05-19 21:57:39.737  INFO 46758 --- [           main] c.d.demo.config.DynamicDataSource        : >>>>>> 当前数据库: masterDataSource <<<<<<
2022-05-19 21:57:39.740  INFO 46758 --- [           main] c.datasource.demo.DemoApplicationTests   : DataEntity(id=1, name=master)
2022-05-19 21:57:39.740  INFO 46758 --- [           main] c.d.demo.config.DynamicDataSource        : >>>>>> 当前数据库: slaveDataSource <<<<<<
2022-05-19 21:57:39.743  INFO 46758 --- [           main] c.datasource.demo.DemoApplicationTests   : DataEntity(id=1, name=slave)
2022-05-19 21:57:39.743  INFO 46758 --- [           main] c.d.demo.config.DynamicDataSource        : >>>>>> 当前数据库: masterDataSource <<<<<<
2022-05-19 21:57:39.745  INFO 46758 --- [           main] c.datasource.demo.DemoApplicationTests   : DataEntity(id=1, name=master)
2022-05-19 21:57:39.745  INFO 46758 --- [           main] c.d.demo.config.DynamicDataSource        : >>>>>> 当前数据库: slaveDataSource <<<<<<
2022-05-19 21:57:39.747  INFO 46758 --- [           main] c.datasource.demo.DemoApplicationTests   : DataEntity(id=1, name=slave)
2022-05-19 21:57:39.747  INFO 46758 --- [           main] c.d.demo.config.DynamicDataSource        : >>>>>> 当前数据库: masterDataSource <<<<<<
2022-05-19 21:57:39.749  INFO 46758 --- [           main] c.datasource.demo.DemoApplicationTests   : DataEntity(id=1, name=master)
2022-05-19 21:57:39.758  INFO 46758 --- [ionShutdownHook] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closing ...
2022-05-19 21:57:39.760  INFO 46758 --- [ionShutdownHook] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closed
2022-05-19 21:57:39.760  INFO 46758 --- [ionShutdownHook] com.alibaba.druid.pool.DruidDataSource   : {dataSource-2} closing ...
2022-05-19 21:57:39.761  INFO 46758 --- [ionShutdownHook] com.alibaba.druid.pool.DruidDataSource   : {dataSource-2} closed
```

## 通过注解指定数据源

新建一个@DataSouce注解，使用该注解指定数据源

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface DataSource {
   
   DataSourceType value() default DataSourceType.MASTER;

}
```

数据源枚举类如下

```java
public enum DataSourceType {
    
    MASTER,

    SLAVE
}
```

注解指定数据源需要对DynamicDataSouce进行改造

```java
@RequiredArgsConstructor
public class DynamicDataSource extends AbstractRoutingDataSource {

    private final Map<String, DataSource> targetDataSources;

    @Override
    protected Object determineCurrentLookupKey() {
        return DynamicDataSourceContextHolder.getDataSourceType();
    }
    
    @Override
    public void afterPropertiesSet() {
        super.setDefaultTargetDataSource(targetDataSources.get(DataSourceType.MASTER.name()));
        super.setTargetDataSources(new HashMap<>(targetDataSources));
        super.afterPropertiesSet();
    }
    
}
```

这里使用DynamicDataSourceContextHolder持有数据源，该类使用LocalThread对数据源进行保存，保证线程上下文使用同一数据源

```java
@Slf4j
public class DynamicDataSourceContextHolder {

    /**
     * 使用ThreadLocal维护变量，ThreadLocal为每个使用该变量的线程提供独立的变量副本，
     *  所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。
     */
    private static final ThreadLocal<String> CONTEXT_HOLDER = new ThreadLocal<>();

    /**
     * 设置数据源的变量
     */
    public static void setDataSourceType(String dsType)
    {
        log.info("切换到{}数据源", dsType);
        CONTEXT_HOLDER.set(dsType);
    }

    /**
     * 获得数据源的变量
     */
    public static String getDataSourceType()
    {
        return CONTEXT_HOLDER.get();
    }

    /**
     * 清空数据源变量
     */
    public static void clearDataSourceType()
    {
        CONTEXT_HOLDER.remove();
    }
}
```

### 设置切面

最后就是要使用切面对注解进行解析，进而动态设置数据源，切面如下

```java
@Aspect
@Order(1)
@Component
public class DataSourceAspect {
    
    @Pointcut("@annotation(com.datasource.demo.annotation.DataSource)"
            + "|| @within(com.datasource.demo.annotation.DataSource)")
    public void dsPointCut() {
        
    }

    @Around("dsPointCut()")
    public Object around(ProceedingJoinPoint pointcut) throws Throwable {

        DataSource dataSource = getDataSource(pointcut);
        DynamicDataSourceContextHolder.setDataSourceType(dataSource.value().name());

        try {
            return pointcut.proceed();
        } finally {
            DynamicDataSourceContextHolder.clearDataSourceType();
        }
    }

    private DataSource getDataSource(ProceedingJoinPoint pointcut) {
        MethodSignature signature = (MethodSignature) pointcut.getSignature();
        DataSource dataSource = AnnotationUtils.findAnnotation(signature.getMethod(), DataSource.class);
        if (Objects.nonNull(dataSource)) {
            return dataSource;
        }
        return AnnotationUtils.findAnnotation(signature.getDeclaringType(), DataSource.class);
    }
}
```

### 测试

设置DataSource注解

```java
public interface DataMapper {

    @Select("select * from test_table where id = 1")
    @DataSource(DataSourceType.SLAVE)
    DataEntity getData();
    
}
```

运行单元测试

```java
@SpringBootTest
@Slf4j
class DemoApplicationTests {

	@Autowired
	private DataMapper dataMapper;

	@Test
	void testDataSource() throws Exception {
		log.info(dataMapper.getData().toString());
	}

}

//输出
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.6.7)

2022-06-04 13:45:38.500  INFO 82721 --- [           main] c.datasource.demo.DemoApplicationTests   : Starting DemoApplicationTests using Java 1.8.0_302 on Mac-mini.local with PID 82721 (started by wag in /Users/wag/projects/for_blog)
2022-06-04 13:45:38.500  INFO 82721 --- [           main] c.datasource.demo.DemoApplicationTests   : No active profile set, falling back to 1 default profile: "default"
2022-06-04 13:45:39.726  INFO 82721 --- [           main] c.datasource.demo.DemoApplicationTests   : Started DemoApplicationTests in 1.507 seconds (JVM running for 1.977)
2022-06-04 13:45:39.846  INFO 82721 --- [           main] c.d.d.d.DynamicDataSourceContextHolder   : 切换到SLAVE数据源
2022-06-04 13:45:39.894  INFO 82721 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2022-06-04 13:45:40.043  INFO 82721 --- [           main] c.datasource.demo.DemoApplicationTests   : DataEntity(id=1, name=slave)
2022-06-04 13:45:40.054  INFO 82721 --- [ionShutdownHook] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closing ...
2022-06-04 13:45:40.056  INFO 82721 --- [ionShutdownHook] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closed
2022-06-04 13:45:40.056  INFO 82721 --- [ionShutdownHook] com.alibaba.druid.pool.DruidDataSource   : {dataSource-0} closing ...
