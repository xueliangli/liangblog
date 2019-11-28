---
layout: post
title:  "Spring Cloud Sleuth 进阶实战"
categories: SpringCloud
tags:  SpringCloud
---

* content
{:toc}

## 为什么需要Spring Cloud Sleuth


<!--more-->



 Spring Cloud Sleuth采用的是Google的开源项目Dapper的专业术语。
 

## 案例实战


在UserController类建一个“/user/hi”的API接口，对外提供服务，代码如下：

## 使用spring-cloud-starter-stream-rabbit进行链路通讯

在上述的案例中，最终gateway-service收集的数据，是通过Http上传给zip-server的，在Spring Cloud Sleuth中支持消息组件来通讯的，在这一小节使用RabbitMQ来通讯。首先来改造zipkin-server，在pom文件将zipkin-server的依赖去掉，加上spring-cloud-sleuth-zipkin-stream和spring-cloud-starter-stream-rabbit，代码如下：

```

	<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-sleuth-zipkin-stream</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-stream-rabbit</artifactId>
		</dependency>

```

在application.yml配置上RabbitMQ的配置，包括host、端口、用户名、密码，如下：

```
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

```

在程序的启动类ZipkinServerApplication上@EnableZipkinStreamServer注解，开启ZipkinStreamServer。代码如下：

```
@SpringBootApplication
@EnableEurekaClient
@EnableZipkinStreamServer
public class ZipkinServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(ZipkinServerApplication.class, args);
	}
}

```

现在来改造下Zipkin Client（包括gateway-service、user-service），在pom文件中将spring-cloud-starter-zipkin以来改为spring-cloud-sleuth-zipkin-stream和spring-cloud-starter-stream-rabbit，代码如下：

```

<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-sleuth-zipkin-stream</artifactId>

		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-stream-rabbit</artifactId>
		</dependency>

```

同时在applicayion.yml文件加上RabbitMQ的配置，同zipkin-server工程。

这样，就将链路的上传数据从Http改了为用消息代组件RabbitMQ。

## 将链路数据存储在Mysql数据库

在上述的例子中，Zipkin Server是将数据存储在内存中，一旦程序重启，之前的链路数据全部丢失，那么怎么将链路数据存储起来呢？Zipkin支持Mysql、Elasticsearch、Cassandra存储。这一小节讲述用Mysql存储，下一节讲述用Elasticsearch存储。

首先，在zipkin-server工程加上Mysql的连接依赖mysql-connector-java，JDBC的起步依赖spring-boot-starter-jdbc，代码如下：

```
<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-jdbc</artifactId>
		</dependency>

```

在配置文件application.yml加上数据源的配置，包括数据库的Url、用户名、密码、连接驱动，另外需要配置zipkin.storage.type为mysql，代码如下：

```
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/spring-cloud-zipkin?useUnicode=true&characterEncoding=utf8&useSSL=false
    username: root
    password: 123456
    driver-class-name: com.mysql.jdbc.Driver
zipkin:
  storage:
    type: mysql

```

另外需要在Mysql数据库中初始化数据库脚本，数据库脚本地址：https://github.com/openzipkin/zipkin/blob/master/zipkin-storage/mysql/src/main/resources/mysql.sql

```
CREATE TABLE IF NOT EXISTS zipkin_spans (
  `trace_id_high` BIGINT NOT NULL DEFAULT 0 COMMENT 'If non zero, this means the trace uses 128 bit traceIds instead of 64 bit',
  `trace_id` BIGINT NOT NULL,
  `id` BIGINT NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `parent_id` BIGINT,
  `debug` BIT(1),
  `start_ts` BIGINT COMMENT 'Span.timestamp(): epoch micros used for endTs query and to implement TTL',
  `duration` BIGINT COMMENT 'Span.duration(): micros used for minDuration and maxDuration query'
) ENGINE=InnoDB ROW_FORMAT=COMPRESSED CHARACTER SET=utf8 COLLATE utf8_general_ci;

ALTER TABLE zipkin_spans ADD UNIQUE KEY(`trace_id_high`, `trace_id`, `id`) COMMENT 'ignore insert on duplicate';
ALTER TABLE zipkin_spans ADD INDEX(`trace_id_high`, `trace_id`, `id`) COMMENT 'for joining with zipkin_annotations';
ALTER TABLE zipkin_spans ADD INDEX(`trace_id_high`, `trace_id`) COMMENT 'for getTracesByIds';
ALTER TABLE zipkin_spans ADD INDEX(`name`) COMMENT 'for getTraces and getSpanNames';
ALTER TABLE zipkin_spans ADD INDEX(`start_ts`) COMMENT 'for getTraces ordering and range';

CREATE TABLE IF NOT EXISTS zipkin_annotations (
  `trace_id_high` BIGINT NOT NULL DEFAULT 0 COMMENT 'If non zero, this means the trace uses 128 bit traceIds instead of 64 bit',
  `trace_id` BIGINT NOT NULL COMMENT 'coincides with zipkin_spans.trace_id',
  `span_id` BIGINT NOT NULL COMMENT 'coincides with zipkin_spans.id',
  `a_key` VARCHAR(255) NOT NULL COMMENT 'BinaryAnnotation.key or Annotation.value if type == -1',
  `a_value` BLOB COMMENT 'BinaryAnnotation.value(), which must be smaller than 64KB',
  `a_type` INT NOT NULL COMMENT 'BinaryAnnotation.type() or -1 if Annotation',
  `a_timestamp` BIGINT COMMENT 'Used to implement TTL; Annotation.timestamp or zipkin_spans.timestamp',
  `endpoint_ipv4` INT COMMENT 'Null when Binary/Annotation.endpoint is null',
  `endpoint_ipv6` BINARY(16) COMMENT 'Null when Binary/Annotation.endpoint is null, or no IPv6 address',
  `endpoint_port` SMALLINT COMMENT 'Null when Binary/Annotation.endpoint is null',
  `endpoint_service_name` VARCHAR(255) COMMENT 'Null when Binary/Annotation.endpoint is null'
) ENGINE=InnoDB ROW_FORMAT=COMPRESSED CHARACTER SET=utf8 COLLATE utf8_general_ci;

ALTER TABLE zipkin_annotations ADD UNIQUE KEY(`trace_id_high`, `trace_id`, `span_id`, `a_key`, `a_timestamp`) COMMENT 'Ignore insert on duplicate';
ALTER TABLE zipkin_annotations ADD INDEX(`trace_id_high`, `trace_id`, `span_id`) COMMENT 'for joining with zipkin_spans';
ALTER TABLE zipkin_annotations ADD INDEX(`trace_id_high`, `trace_id`) COMMENT 'for getTraces/ByIds';
ALTER TABLE zipkin_annotations ADD INDEX(`endpoint_service_name`) COMMENT 'for getTraces and getServiceNames';
ALTER TABLE zipkin_annotations ADD INDEX(`a_type`) COMMENT 'for getTraces';
ALTER TABLE zipkin_annotations ADD INDEX(`a_key`) COMMENT 'for getTraces';
ALTER TABLE zipkin_annotations ADD INDEX(`trace_id`, `span_id`, `a_key`) COMMENT 'for dependencies job';

CREATE TABLE IF NOT EXISTS zipkin_dependencies (
  `day` DATE NOT NULL,
  `parent` VARCHAR(255) NOT NULL,
  `child` VARCHAR(255) NOT NULL,
  `call_count` BIGINT,
  `error_count` BIGINT
) ENGINE=InnoDB ROW_FORMAT=COMPRESSED CHARACTER SET=utf8 COLLATE utf8_general_ci;

ALTER TABLE zipkin_dependencies ADD UNIQUE KEY(`day`, `parent`, `child`);

```
## 将链路数据存储在ElasticSearch

使用Mysql存储链路数据，在并发高的情况下，显然不合理，这时可以选择使用ElasticSearch存储。读者需要自行安装ElasticSearch、Kibana（下一小节使用），下载地址为https://www.elastic.co/products/elasticsearch。安装完成后并启动它们，其中ElasticSearch的默认端口为9200，Kibana的端口为5601。

安装的过程可以参考我的这篇文章：http://blog.csdn.net/forezp/article/details/71189836

本小节的案例在上上小节的案例的基础上进行改造。首先在pom文件，加上zipkin的依赖和zipkin-autoconfigure-storage-elasticsearch-http的依赖，代码如下：

```

		<dependency>
			<groupId>io.zipkin.java</groupId>
			<artifactId>zipkin</artifactId>
			<version>1.28.0</version>
		</dependency>
		<dependency>
			<groupId>io.zipkin.java</groupId>
			<artifactId>zipkin-autoconfigure-storage-elasticsearch-http</artifactId>
			<version>1.28.0</version>
		</dependency>

```

在application.yml文件加上Zipkin的配置，配置了zipkin的存储类型为elasticsearch，使用的StorageComponent为elasticsearch。然后需要配置elasticsearch，包括hosts，可以配置多个，用“，”隔开；index为zipkin等，具体配置如下：

```

zipkin:
  storage:
    type: elasticsearch
    StorageComponent: elasticsearch
    elasticsearch:
      cluster: elasticsearch
      max-requests: 30
      index: zipkin
      index-shards: 3
      index-replicas: 1
      hosts: localhost:9200

```

## 在kibana上展示

上一小节讲述了如何将链路数据存储在ElasticSearch，ElasticSearch可以和Kibana结合，将链路数据展示在 Kibana上。安装完Kibana，并启动，它默认会向本地的9200端口的ElasticSearch读取数据，它默认的端口为5601。访问http://localhost:5601，显示的界面如下：

![image.png](http://upload-images.jianshu.io/upload_images/2279594-68c5313593f90086.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上述的界面点击"Management"按钮，然后点击“Add New”，添加一个index，在上节我们在ElasticSearch中写入链路数据的index配置为“zipkin”,那么在界面填写为“zipkin-*”，点击“Create”按钮。


![image.png](http://upload-images.jianshu.io/upload_images/2279594-effd2c517fe1884f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 创建完index之后，点击Discover，就可以在界面上展示链路数据了。

![image.png](http://upload-images.jianshu.io/upload_images/2279594-13d602df46b014f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 源码下载

最原始的工程：

https://github.com/forezp/SpringCloudLearning/tree/master/chapter-sleuth

采用RabbitMq通讯的工程：

https://github.com/forezp/SpringCloudLearning/tree/master/chapter-sleuth-stream

采用Mysql存储的工程：

https://github.com/forezp/SpringCloudLearning/tree/master/chapter-sleuth-stream-mysql

采用ES存储的工程：

https://github.com/forezp/SpringCloudLearning/tree/master/chapter-sleuth-stream-elasticsearch

## 参考资料

http://cloud.spring.io/spring-cloud-sleuth/spring-cloud-sleuth.html

https://github.com/openzipkin/zipkin


## 关注我的公众号

精彩内容不能错过！

![forezp.jpg](http://upload-images.jianshu.io/upload_images/2279594-0805748d92bba033.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/200)
