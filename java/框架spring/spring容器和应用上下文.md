#### 1.容器概念

应用中的对象存在与Spring容器(container)中，Spring容器使用ID(依赖注入)负责创建对象、装配并管理他们的整个生命周期。

#### 2.容器分类

spring容器大致可以分为两类：

```java
/**
 * org.springframework.beans.factory.BeanFactory，提供最基本的ID支持，其基本方法如下(一般不会使用)
 */

package org.springframework.beans.factory;

public interface BeanFactory {

	String FACTORY_BEAN_PREFIX = "&";

	Object getBean(String name) throws BeansException;

	<T> T getBean(String name, Class<T> requiredType) throws BeansException;

	boolean containsBean(String name);

	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	String[] getAliases(String name);
}

/**
 * 应用上下文org.springframework.context.ApplicationContext，提供应用框架
 * 级别的服务，例如从属性文件解析文本信息、发布应用事件给感兴趣的事件监听者。
 * 
 */
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
		MessageSource, ApplicationEventPublisher, ResourcePatternResolver {

	String getId();

	String getApplicationName();

	String getDisplayName();

	long getStartupDate();

	ApplicationContext getParent();

	AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException;

} 
```
#### 3. 应用上下文分类(重申：应用上下文是容器的一种)

Spring的应用上下文对象分为两种：从**配置类**中加载应用上下文和从**配置文件**中加载应用上下文：

```java
//配置类AnnotationConfigXXX、Spring上下文和Spring web应用上下文；(配置类使用@Configuration)
1. AnnotationConfigApplicationContext:从基于Java的配置类中加载Spring应用上下文；

2. AnnotationConfigWebApplicationContext:从基于Java的配置类中加载Spring web应用上下文；

ApplicationContext context=new AnnotationConfigApplicationContext(path.to.class.AuthServiceConfig.class);

//配置文件Xml：类路径classPath、文件系统FileSystem和web应用下的XML配置文件
1. ClassPathXmlApplicationContext:从类路径下的XML配置文件中加载上下文定义（把应用上下文定义文件作为类资源);

2. FileSystemXmlapplication:从文件系统下的xml配置文件中加载上下文定义；

3.XmlWebApplicationContext:从web应用下的XML配置文件中加载上下文定义。

ApplicationContext context=new ClassPathXmlApplicationContext("knight.xml");

```

通过以上方式获取Spring的应用上下文以后，就可以通过`getBean("beanName")`方法获取bean——重申：==spring容器/应用上下文==负责创建、装配bean，负责bean的整个生命周期。