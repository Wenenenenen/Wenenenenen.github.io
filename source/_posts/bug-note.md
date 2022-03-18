---
title: BUG笔记
index_img: /img/index_post/bug.jpg # https://s2.loli.net/2022/03/18/xWpOyBb6Zm1R3kd.jpg
date: 2022-03-18 10:32:14
tags: [JAVA,spring boot,spring cloud,easy poi]
categories: JAVA
excerpt: 日常开发过程遇到的bug
sticky: 100
---
## 1. spring boot启动报错@`activatedProperties`@

> 时有时无 -> ERROR `org.springframework.boot.SpringApplication` - Application run failed
> `org.yaml.snakeyaml.scanner.ScannerException`: while scanning for the next token
> found character '@' that cannot start any token. (Do not use @ for indentation)
> in 'reader', line 11, column 13:
>      active: @`activatedProperties`@

#### 解决方法 -> pom.xml 加上以下代码  

```xml
<build>
    <resources>
        <resource>
			<directory>src/main/resources</directory>
            <!--开启过滤，解决@activatedProperties@报错-->
            <filtering>true</filtering>
        </resource>
    </resources>
</build>
```

## 2. spring cloud 2020 引用 `LoadBalancerClient` 启动报错

> `A component required a bean of type 'org.springframework.cloud.client.loadbalancer.LoadBalancerClient' that could not be found.`
>
> 原因：spring cloud 2020版本，移除了以前自动引入的ribbon，换成了`LoadBalancer`，但是需要手动引入包

#### 解决方法 -> pom.xml加上

```xml
<!-- LB 扩展 -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

> 引申错误，找不到服务名的URL `java.net.UnknownHostException: cloud-nacos-provider`
>
> 需要确保`restTemplate`上加有`@LoadBalanced`注解，并且有ribbon引用或者`LoadBalancer`引用包才行

## 3. easy-poi 读取时间列读到的值为44197

> 原因：Excel日期格式自定义读取为44197 就是自1900.1.1以来的天数 手动转换一下  转换失败就按照`EEE MMM dd HH:mm:ss zzz yyyy`格式来读取

#### 解决方法 ->  用Calendar或者改Excel日期格式

```java
Calendar instance = Calendar.getInstance();
for (DimensionTimeTable timeTable : excelList) {
  instance.set(1900, Calendar.JANUARY, 1);
  String timeName = timeTable.getTimeName();
  try {
    int days = Integer.parseInt(timeName);
    instance.add(Calendar.DAY_OF_MONTH, days);
  } catch (NumberFormatException numberFormatException) {
    log.warn(numberFormatException.getMessage());
    Date parse = DateUtil.parse(timeName);
    instance.setTime(parse);
  }
  timeTable.setTimeName(instance.get(Calendar.YEAR) + "年" + (instance.get(Calendar.MONTH)+1) + "月");
}
```

## 4.  spring boot项目报错找不到Tomcat临时文件夹

> 报错：
>
> `Failed to parse multipart servlet request; nested exception is java.io.IOException: The temporary upload location`
> `[/tmp/tomcat.34315()5664888818074.8081/work/Tomcat/localhost/ROOT] is not valid`
>
> 原因：在Linux系统中，spring boot应用服务再启动（java -jar 命令启动服务）的时候，会在操作系统的`/tmp`目录下生成一个tomcat*的文件目录，上传的文件先要转换成临时文件保存在这个文件夹下面。由于临时`/tmp`目录下的文件，在长时间（10天）没有使用的情况下，就会被系统机制自动删除掉。所以如果系统长时间无人问津的话，就可能导致上面这个问题

解决方法 -> 三种

1、重启服务

2、项目的配置文件中，手动给这个临时文件夹设定目录，这样子就不会被Linux删除了

```yml
server:
  tomcat:
    basedir: /usr/local/tmp/  #修改内置Tomcat临时文件夹 避免在tmp里被定时清理
```

3.写个配置类，通过@Bean的方式配置目录：

```java
@Configuration
public class MultipartConfig {
    /**
     * 文件上传临时路径
     */
    @Bean
    MultipartConfigElement multipartConfigElement() {
        MultipartConfigFactory factory = new MultipartConfigFactory();
        String location = System.getProperty("user.home") + "/.xiangmu/file/tmp";
        File tmpFile = new File(location);
        if (!tmpFile.exists()) {
            tmpFile.mkdirs();
        }
        factory.setLocation(location);
        return factory.createMultipartConfig();
    }
 
}
```
