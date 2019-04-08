---
layout: post
title:  Springboot加载自定义yml文件配置的方法
date:   2019-04-03 12:08:00 +0800
categories: 技巧
tag: Springboot
---

* content
{:toc}


**Springboot在1.5.x版本之后，去除了@ConfigurationProperties注解中的location参数，因此无法再通过这种方式加载自定义的yml配置文件了。**

我在网上看到很多资料都是这么说明的，我也没去验证，先不管吧，总之我搜集资料后特意总结了以下三种解决方法，亲测有效。

首先先在resources目录下创建一个测试用的yml配置文件，内容如下：

**PS：注意yml文件中key: value的冒号后面是有一个空格的**
```yml
system:
  user:
    name: zjhuang
    password: 123456
    age: 25
```
## **一. @ConfigurationProperties + @PropertySource + @Value注解方式**
* 配置参数类代码：

1. 这种方法必须配置@ConfigurationProperties中的prefix前缀信息，否则无法获取到yml数据。

2. @PropertySource是为了告知springboot加载自定义的yml配置文件，springboot默认会自动加载application.yml文件，如果参数信息直接写在这个文件里，则不需要显式加载。

3. 在@ConfigurationProperties中配置了prefix前缀信息的条件下，@Value指定yml配置文件中的参数项名称。
```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.PropertySource;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "system.user")
@PropertySource(value = "classpath:test.yml")
public class YamlProperties {

    @Value("${name}")
    private String name;
    @Value("${password}")
    private String password;
    @Value("${age}")
    private int age;

    // Setter...
    // Getter...
    // toString...
}
```
* 测试用例代码：

后面的两种方式也共用这个测试用例代码，下面不再列出。
```java
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
public class TestYamlLoader {

    @Autowired
    private YamlProperties yamlProperties;

    @Test
    public void test() {
        System.out.println(yamlProperties.toString());
    }
}
```
* 输出结果：
```
YamlProperties{name='zjhuang', password='123456', age=25}
```

## **二. 实现PropertySourceFactory接口 + @PropertySource + @Value方式**

* PropertySourceFactory实现类代码
```java
import org.springframework.boot.env.PropertySourcesLoader;
import org.springframework.core.env.PropertySource;
import org.springframework.core.io.Resource;
import org.springframework.core.io.support.EncodedResource;
import org.springframework.core.io.support.PropertySourceFactory;
import org.springframework.util.StringUtils;

import java.io.IOException;

public class YamlPropertySourceFactory implements PropertySourceFactory {

    @Override
    public PropertySource<?> createPropertySource(String name, EncodedResource encodedResource) throws IOException {
        return name != null ? new PropertySourcesLoader().load(encodedResource.getResource(), name, null) : new PropertySourcesLoader().load(
                encodedResource.getResource(), getNameForResource(encodedResource.getResource()), null);
    }

    private static String getNameForResource(Resource resource) {
        String name = resource.getDescription();
        if (!StringUtils.hasText(name)) {
            name = resource.getClass().getSimpleName() + "@" + System.identityHashCode(resource);
        }
        return name;
    }
}
```

* 配置参数类代码

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.PropertySource;
import org.springframework.stereotype.Component;

@Component
@PropertySource(value = "classpath:test.yml", factory = YamlPropertySourceFactory.class)
public class YamlProperties {

    @Value("${system.user.name}")
    private String name;
    @Value("${system.user.password}")
    private String password;
    @Value("${system.user.age}")
    private int age;

    // Setter...
    // Getter...
    // toString...
}
```

* 跟第一种方法不同点在于：

1、放弃了@ConfigurationProperties注解，改为实现了PropertySourceFactory接口

2、@PropertySource注解定义了factory属性，值为上一步中
PropertySourceFactory实现类的class

3、@Value注解中使用了参数在yml'文件中的全限定名

* 输出结果：

```
YamlProperties{name='zjhuang', password='123456', age=25}
```

## **三.使用YamlPropertiesFactoryBean + @Value方式**

* PropertySourcesPlaceholderConfigurer类代码

```java
import org.springframework.beans.factory.config.YamlPropertiesFactoryBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.support.PropertySourcesPlaceholderConfigurer;
import org.springframework.core.io.ClassPathResource;
import org.springframework.stereotype.Component;

@Component
public class PropertySourcesPlaceholderConfigurerBean {
    @Bean
    public PropertySourcesPlaceholderConfigurer yaml() {
        PropertySourcesPlaceholderConfigurer configurer = new PropertySourcesPlaceholderConfigurer();
        YamlPropertiesFactoryBean yaml = new YamlPropertiesFactoryBean();
        yaml.setResources(new ClassPathResource("test.yml"));
        configurer.setProperties(yaml.getObject());
        return configurer;
    }
}
```

* 配置参数类代码

这第三种方法去掉了@PropertySource 注解，改为定义一个PropertySourcesPlaceholderConfigurer类型的@Bean来加载配置文件信息。
```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.beans.factory.config.YamlPropertiesFactoryBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.support.PropertySourcesPlaceholderConfigurer;
import org.springframework.core.io.ClassPathResource;
import org.springframework.stereotype.Component;

@Component
public class YamlProperties {
    
    @Value("${system.user.name}")
    private String name;
    @Value("${system.user.password}")
    private String password;
    @Value("${system.user.age}")
    private int age;

    // Setter...
    // Getter...
    // toString...

}
```

* 输出结果：

```
YamlProperties{name='zjhuang', password='123456', age=25}
```

以上就是springboot加载自定义yml配置信息的三种方法，大家按照自己喜好选择使用即可；第三种方法好像只能用于加载一个yml文件的情况，第一和第二种方法可以实现加载多个yml文件，@PropertySources的value参数是支持数组赋值的。

```
@PropertySource(value = {"classpath:test1.yml", "classpath:test2.yml"})
```

另外需要提一点的是，不管是加载几个yml文件，@Value修饰的所有参数都必须在yml文件中定义齐全，yml缺少配置的话会抛出无法解析占位符的异常

```
Caused by: java.lang.IllegalArgumentException: Could not resolve placeholder 'age' in value "${age}"
```

