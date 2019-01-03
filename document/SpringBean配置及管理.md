# Spring Bean配置及管理

## 概述

在Spring的核心IOC中，我们主要是对对象以及各种java组件进行管理，我们管这些对象以及java组件叫Bean。所以IOC的核心也就是对于Bean管理的过程，或者叫做管理Bean的生命周期。

Spring使用BeanFactory来实例化、配置以及管理Bean的生命周期。

## BeanFactory

### 代码

```java
    String FACTORY_BEAN_PREFIX = "&";

    Object getBean(String name) throws BeansException;

    <T> T getBean(String name, Class<T> requiredType) throws BeansException;

    Object getBean(String name, Object... args) throws BeansException;

    <T> T getBean(Class<T> requiredType) throws BeansException;

    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

    <T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

    <T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

    boolean containsBean(String name);

    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

    boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

    boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

    @Nullable
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;

    String[] getAliases(String name);
```

### 分析

以上代码为BeanFactory源码，通过源码我们得知

1. `FACTORY_BEAN_PREFIX`，该属性主要用来引用一个实例，或者说把它和实际工厂产生的bean进行区分。例如，一个bean的名称为beanName,那么我们可以得到对应的factory，该factory名为&beanName。

2. `getBean()`，该类中提供了四种不同的方法来获取到bean实例，此方法就是`getBean()`

3. `containsBean(String name)`，该方法用来监测容器中是否包含该bean。

4. `isSingleton(String name)`和`isPrototype(String name)`，这两个方法主要来判断bean对应的作用域`scope`，在BeanFactory中，所有的bean都会有对应的作用域，分为单例(singeton)和原型(prototype)两种，分别对应的就是单实例的singeton以及多个实例的prototype。同时BeanFactory仅仅提供了单例的生命周期管理，对于原型bean，在实例被创建之后便被传给了客户端,容器失去了对它们的引用。

5. `isTypeMatch()`，此方法用来监测bean的名称以及类型是否匹配。

6. `getType(String name)`，该方法用来获取bean的类型

7. `getAliases(String name)`，此方法用来根据实例的名字获取实例的别名，在Spring容器内，我们可以对同一个对象或资源进行多次bean创建时，我们需要针对于定制化的需求，就会用到别名，用来区分同一个bean的不同依赖。

综上，从类的命名以及方法，我们很明显可以看出来这是一个非常典型的工厂模式接口。

## 配置

在BeanFactory中，最常见的配置实现为xml配置，实现类为XmlBeanFactory，可以直接通过file文件系统或者classpath中获取资源来进行实现。

### 代码

- 实现1

```java
File file = new File("file-bean.xml");
Resource resource = new FileSystemResource(file);
BeanFactory beanFactory = new XmlBeanFactory(resource);
```

- 实现2

```java
Resource resource = new ClassPathResource ("classpath-bean.xml");
BeanFactory beanFactory = new XmlBeanFactory(resource);
```

### 解释

通过以上两种方式，XmlBeanFactory可以加载xml的配置文件。

我们通过在applicationContext.xml中配置

```xml
<bean id="demo" class="spring.ioc.demo.Demo" />
```

然后我们可以通过XmlBeanFactory来启动ioc容器，代码如下

```java
public static void main(String[] args) {

    　ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
      Resource res = resolver.getResource("classpath:applicationContext.xml");
      BeanFactory factory = new XmlBeanFactory(res);

      Demo demo = factory.getBean("demo", Demo.class);
}
```

1. XmlBeanFactory，通过Resource装载applicationContext.xml配置信息、启动IoC容器，然后就可以通过factory.getBean从IOC容器中获取Bean了。

2. 在IOC容器启动后，Bean并不会被初始化，只有在调用时，容器才会对其进行初始化。

3. Singleton和Prototype是不同的，BeanFactory会缓存Singleton类型的Bean实例，也就是说在第二次获取该Bean的时候，是直接从容器缓存中读取，并不是重新初始化；Prototype则不同，对于Bean使用完之后会直接被销毁，使用前再次初始化。

## ApplicationContext

在实际的使用过程中，我们往往用到比较多的为ApplicationContext而不是BeanFactory，这是为什么呢？

### 代码

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory, MessageSource, ApplicationEventPublisher, ResourcePatternResolver
```

### 解释

这个得从ApplicationContext来说，我们阅读源码可知，ApplicationContext该接口集成了BeanFactory的功能之外，还对其进行了扩展，相较于BeanFactory，它还提供了以下功能

1. `MessageSource`，提供国际化的消息访问

2. `ApplicationEventPublisher`,应用事件发布，这个也为对AOP特性提供了支持

3. `ResourcePatternResolver`，提供了资源访问，例如URL

4. `EnvironmentCapable`，只是定义了一个方法`getEnvironment()`，该方法用来获取环境变量

## 对比总结

1. BeanFactory提供了Spring的Bean的配置、管理、实例化，同时声明了Bean的生命周期，维护Bean之间的依赖关系。

2. BeanFactory是延迟加载的方式来注入Bean的，在容器启动时并不会初始化Bean，只有档Bean需要使用时，才会进行初始化，针对于此，我们在进行复杂的Bean配置之后，启动项目就很难去发现一些配置方面的错误；而ApplicationContext就刚好相反，在容器启动时，它就会一次性的创建所有的Bean，这样我们就能检查到配置错误，但是正是这样，我们在启动时需要占用大量内存、时间去创建这些Bean，导致启动比较慢。

    - 例如：Bean A初始化是需要依赖于其构造函数中参数a，但是在配置文件中并没有配置，这种情况下启动容器，容器是能正常启动，但是在运行时会导致很多错误；

综上，我们会发现很多情况下，相较而言ApplicationContext使用场景更广，所以很多情况会选择ApplicationContext。

### ApplicationContext配置加载

1. FileSystemXmlApplicationContext——（通过文件系统指定配置文件地址读取）

```java
ApplicationContext context=new FileSystemXmlApplicationContext("D:/applicationContext.xml");
```

2. ClassPathXmlApplicationContext——（从编译后的classpath中读取配置文件）

```jva
ApplicationContext context = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
```

3. XmlWebApplicationContext——（从web根目录读取配置文件，得先在web.xml中配置监听器或者servlet）

```xml
<listener>
<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

或者

```xml
<servlet>
    <servlet-name>context</servlet-name>
    <servlet-class>org.springframework.web.context.ContextLoaderServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
```

以上方式的默认配置文件位置为:`/WEB-INF/applicationContext.xml`，我们也可以在web.xml中手动配置，配置如下：

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/applicationContext.xml</param-value>
</context-param>
```

### **Bean配置(重点讲解xml)**

#### xml配置得三种方式：

1. 类的无参构造，当类中没有无参构造时，会出现无默认构造异常，异常信息为：`No default constructor found`

```xml
<bean id="demo" class="com.demo.Demo" scope="singleton" />
```

2. 工厂静态方法，在Bean的工厂类中，定义静态方法返回类对象,可以直接通过`工厂.方法()`调用

```java
public class DemoFactory {
    public static Demo getDemo() {
        return new Demo();
    }
}
```

```xml
<bean id="demo" class="com.demo.DemoFactory" factory-method="getDemo" />
```

3. 实例工厂，首先需要定义工厂Bean，然后定义工厂的返回类对象，可以通过`对象.方法()`调用

```java
public class DemoFactory {
    public Demo getDemo() {
        return new Demo();
    }
}
```

```xml
<bean id="demoFactory" class="com.demo.DemoFactory" />
<bean id="demo" factory-bean="demoFactory" factory-method="getDemo" />
```

#### bean标签常用属性

- id  指定该bean的id，不能包含特殊字符，且唯一

- class  指定该bean对应的类路径

- name 指定该bean的name，与id功能类似，但是可以包含特殊字符

- scope  指定该bean的作用域，常用的为singleton的单例，以及prototype的多例，默认为singleton

#### 属性注入

使用有参构造来初始化bean

1. 构造参数`constructor-arg`标签，此方法需要类中有对应属性的有参构造

```xml
<bean id="demo" class="com.demo.Demo">
    <constructor-arg name="username" value="Jim" />
</bean>
```

2. 属性`property`标签，此方法需要类中有对应属性的set方法

```xml
<bean id="demo" class="com.demo.Demo">
    <property name="username" value="Jim" />
</bean>
```

3. 属性`property`标签，此方法需要类中有对应属性的set方法，复杂类型注入，例如数组、list、map

```xml
<bean id="demo" class="com.demo.Demo">
    <!-- 数组与list注入相同 -->
    <property name="list">
        <list>
            <value>12</value>
            <value>13</value>
            <value>14</value>
        </list>
    </property>
    <!-- map注入 -->
    <property name="map">
        <map>
            <entry key="id"  value="12" />
            <entry key="name"  value="Jim" />
        </map>
    </property>
</bean>
```

#### 对象注入

在有时候，我们某一个Bean的初始化需要依赖于其他的对象或者bean时，我们可以如一下去进行定义

```xml
<bean id="user" class="com.demo.User" />
<bean id="student" class="com.demo.Student">
    <property name="user" ref="user" />
</bean>
```

## 上一篇

[Spring IOC/DI](/document/spring控制反转与依赖注入.md)

## 下一篇

[Spring IOC的原理与简单实现](/document/SpringIOC的原理与简单实现.md)
