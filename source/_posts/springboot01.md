---
title: SpringBoot是如何进行自动配置的？
date: 2019-06-04 23:20:19
categories: Spring
tags: [SpringBoot 自动配置]
---

> 平时一直在使用SpringBoot，但是没有进入深入的了解，公司需要对原有的项目改造升级，借着这个机会了解下SpringBoot的自动加载机制。<!--more-->

&emsp;&emsp;因为公司向对dubbo进行升级，解决dubbo优雅停机等bug问题，借着这个机会，升级改造项目使用SpringBoot。

#### 常用注解

| 注解                     | 作用                                                         |
| ------------------------ | ------------------------------------------------------------ |
| @Value                   | @Value 就相当于传统 xml 配置文件中的 value 字段。            |
| @ConfigurationProperties | 对标记有@ConfigurationProperties的类所有属性和配置文件中相关的配置项进行绑定。（默认从全局配置文件中获取配置值），绑定之后我们就可以通过这个类去访问全局配置文件中的属性值了。 |
| @Import                  | @Import注解支持导入普通Java类，并将其声明成一个bean。主要用于将对各分散的java config配置类融合成一个更大的config类。 |
| @Conditional             | @Conditional注释可以实现只有在特定条件满足时才启用一些配置。 |

#### SpringBoot自动配置原理

&emsp;&emsp;SpringBoot的自动配置源头一切都要从@SpringBootApplication这个注解开始说起。

@SpringBootApplication 标注在某个类上说明：

- 这个类是 SpringBoot 的主配置类。
- SpringBoot 就应该运行这个类的 main 方法来启动 SpringBoot 应用。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
	// ...
}
```

&emsp;&emsp;通过源码可知，@SpringBootApplication是一个组合注解，由三个注解组成：

@SpringBootConfiguration：该注解表示这是一个SpringBoot的配置类，其实就是一个Configuration注解而已。

@EnableAutoConfiguration：开启自动配置。

@ComponentScan：开启组件扫描。

在查看下开启自动配置的的注解类：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	String[] excludeName() default {};

}

```

@EnableAutoConfiguration注解由@AutoConfigurationPackage和@Import两个注解组成。

@AutoConfigurationPackage：由@Import(AutoConfigurationPackages.Registrar.class)可知，其注解主要是将主配置类（@SpringBootConfiguration标注的类）的所在包及下面所有子包里面的所有组件扫描到Spring容器中。

@Import自动导入了AutoConfigurationImportSelector.class配置类。

&emsp;&emsp;通过AutoConfigurationImportSelector#selectImports方法返回需要导入的组件的全类名数组的。

```java
public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
  //1、加载META-INF/spring-autoconfigure-metadata.properties文件
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
		AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(autoConfigurationMetadata,
				annotationMetadata);
  //3. 将EnableAutoConfiguration的值封装在一个数组中返回。
		return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
	}
```

通过 getAutoConfigurationEntry() -> getCandidateConfigurations() -> loadFactoryNames()查找全类名。

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
				getBeanClassLoader());
		Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
				+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
```

在loadFactoryNames方法中，从当前项目的路径中获取所有MATA-INF/spring.factories文件中的信息。将获取到的信息封装为一个Map返回。在从map中通过传入的 EnableAutoConfiguration.class参数，获取该key下的所有值。

对获取的AutoConfigurationEntry进行过滤，过滤的依据是条件匹配。通过org.springframework.boot.autoconfigure.AutoConfigurationImportFilter过滤器进行过滤，分为configurations，exclusions两个集合。

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
			AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return EMPTY_ENTRY;
		}
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
		List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
		configurations = removeDuplicates(configurations);
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		checkExcludedClasses(configurations, exclusions);
		configurations.removeAll(exclusions);
		configurations = filter(configurations, autoConfigurationMetadata);
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return new AutoConfigurationEntry(configurations, exclusions);
	}
```

#### 总结

&emsp;&emsp;SpringBoot的自动配置体现了约定大于配置的设计理念。将SpringBoot需要的jar在Spring.factories中配置。通过SPI的设计模式，在服务启动时，用ServiceLoader进行加载。