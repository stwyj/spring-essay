# Spring IOC的原理与简单实现

## 概述

&ensp;&ensp;&ensp;&ensp;在Spring IOC中，我们基本都了解，IOC主要是对Bean的管理为主，所以这次，我们需要分析一下IOC是基于什么原理来实现的呢？

## Java反射机制

&ensp;&ensp;&ensp;&ensp;java从业者基本都知道java语言是支持反射的，但是到底什么是反射呢？反射可以用来干什么呢？这些就是需要一起来探讨一下。

### 什么是反射

&ensp;&ensp;&ensp;&ensp;java文件是需要编译之后才能交由jvm去解释运行的，所以当一个.java文件编译成.class文件之后去运行，但是在我们运行过程中，一个对象该如何去`检视自己所拥有的内部成员（成员方法与属性）`，如何获取到某个类的`class信息`、`构造方法`等等呢？这种情况下我们就需要用到反射。

&ensp;&ensp;&ensp;&ensp;对于反射机制，其明确定义为：`java允许程序在运行时可以进行自我检视以及对内部成员进行操作，是java语言的一种特性`。

### 反射可以用来干什么

&ensp;&ensp;&ensp;&ensp;在明白什么是反射之后，我们可以确认反射是在程序运行过程中对类的操作，但是我们有时候不禁要问为什么我们需要用反射？直接创建对象来进行操作不就可以了。

&ensp;&ensp;&ensp;&ensp;这个主要涉及到开发过程中的两种概念，静态编译和动态编译：

- 静态编译——指的是在程序编译前确定类型，后期只能按照该类型来绑定对象，不可变更

- 动态编译——与静态编译相反，动态编译确定类型与绑定对象都可以在运行时确定，这样我们就可以减少垃圾产生、同时可以更加灵活的组合我们的应用。

&ensp;&ensp;&ensp;&ensp;那么经过以上的静态与动态编译我们了解了为什么我们需要用到反射，但是我们可以用反射来干点什么呢？

1. 我们可以判断一个对象所属类信息，也就是我们的`instanceOf`，这样我们可以对对象进行不同的处理；

2. 运行时实例化类的对象，`newInstance`这个方法就可以通过类来创建对象了；

3. 运行时可以访问java对象的类、属性以及方法，例如：`getClass`

## 工厂模式

&ensp;&ensp;&ensp;&ensp;工厂模式是 Java 中最常用的设计模式之一。在使用到工厂模式后，我们在创建对象时不会对客户端暴露创建逻辑，并且是通过使用一个共同的接口来指向新创建的对象。

&ensp;&ensp;&ensp;&ensp;对于工厂模式相信大家都非常非常熟悉，这里不做细讲。

## IOC的实现

&ensp;&ensp;&ensp;&ensp;上篇文章中我们讲解了Spring Bean配置管理是由BeanFactory来管理的，那么从名称上我们可以知道它就是一个工厂模式的实现，加上配置文件中我们定义了Bean的属性以及class，实际上这个就是对反射的一种应用。

&ensp;&ensp;&ensp;&ensp;那么IOC是如何通过反射以及工厂模式来实现的呢？下面我们来简单的实现一下最基础的IOC

## BeanFactory

### 代码


DemoContainer.java

```java
public class DemoContainer{

    private Map<String, PersonFactory> personFactoryMap = new HashMap<>();

    private Properties properties;

    /**
     * 无参构造，实例化时调用init方法来读取配置文件
     */
    public DemoContainer() {
        try {
            init();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 读取配置文件，咱们暂时以properties文件读取为例，与xml相比，仅仅解析不同
     */
    private void init() throws IOException {
        Properties pro = new Properties();

        File f = new File("D:\\idea-workspace\\demo\\src\\main\\java\\com\\example\\demo\\application.properties");

        if(f.exists()){
            pro.load(new FileInputStream(f));
        }else{
            System.out.println("application.properties 文件不存在！");
        }
        this.properties = pro;
    }

    /**
     * 获取bean
     * @param beanId beanId
     * @return bean
     */
    public PersonFactory getBean(String beanId) {
        if(this.personFactoryMap != null && this.personFactoryMap.containsKey(beanId)) {
            System.out.println("读取缓存中的Bean：" + beanId);
        } else {
            System.out.println("初始化Bean：" + beanId);
            getInstance(beanId);
        }

        return this.personFactoryMap.get(beanId);
    }

    /**
     * 根据beanId获取类名
     * @param beanId beanId
     * @return 类名
     * @throws Exception 无对应类名异常
     */
    private String getClassName(String beanId) throws Exception {
        String className = this.properties.getProperty(beanId);

        if(StringUtils.hasText(className)) {
            return className;
        } else {
            throw new Exception("this bean is not in Container");
        }
    }

    /**
     * 根据beanId实例化Bean
     * @param beanId beanid
     */
    private void getInstance(String beanId) {
        try {
            String className = getClassName(beanId);
            PersonFactory personFactory = (PersonFactory) Class.forName(className).newInstance();
            this.personFactoryMap.put(beanId, personFactory);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

## Bean（Lady、Gentleman）

PersonFactory.java

```java
public interface PersonFactory {

   public void introduce();
}
```

Lady.java

```java
public class Lady implements PersonFactory {
    @Override
    public void introduce() {
        System.out.println("hello,i'm Lily!");
    }
}
```

Gentleman.java

```java
public class Gentleman implements PersonFactory {
    @Override
    public void introduce() {
        System.out.println("hello,i'm Jim");
    }
}
```

## application.properties

&ensp;&ensp;&ensp;&ensp;此处我们对于bean的配置也进行了简化，`=`前参数对应xml中beanId，`=`后参数对应xml中的class

```properties
bean.id.lady=com.example.demo.Lady
bean.id.gentleman=com.example.demo.Gentleman
```
## Test

```java
public class Test {
    public static void main(String[] args) {
        /**
         * 初始化启动容器
         */
        DemoContainer demoContainer = new DemoContainer();

        PersonFactory lady = demoContainer.getBean("bean.id.lady");
        lady.introduce();

        PersonFactory lady1 = demoContainer.getBean("bean.id.lady");
        lady1.introduce();

        PersonFactory gentleman = demoContainer.getBean("bean.id.gentleman");
        gentleman.introduce();
    }
}
```

## 运行结果

```text
初始化Bean：bean.id.lady
hello,i'm Lily!
读取缓存中的Bean：bean.id.lady
hello,i'm Lily!
初始化Bean：bean.id.gentleman
hello,i'm Jim
```

### 分析

&ensp;&ensp;&ensp;&ensp;在上述实现中：

1. 我们定义了一个PersonFactory接口，其中定义了方法introduce()

2. 定义了Lady以及Gentleman两个class，对接口PersonFactory进行了实现

3. 定义了DemoContainer，对配置文件进行了加载，提供了bean的实例化以及管理

4. 在Test中，我们首先实例化了容器DemoContainer，然后根据beanId对对应的bean进行了实例化以及接口的调用。

&ensp;&ensp;&ensp;&ensp;在此示例中，我们实际开发业务也仅仅是需要知道beanId，而无需去控制bean的实例化，同时对bean的可复用性我们并不需要关心

### 分析DemoContainer

&ensp;&ensp;&ensp;&ensp;此示例中，其核心就是PersonFactory和DemoContainer，其就是对IOC中的BeanFactory的功能的简单实现，它首先提供了读取配置文件功能。

- 其无参构造中`public DemoContainer(){}`，我们定义了其实例化启动时需要初始化读取配置文件，也就是调用了`init()`方法，然后将配置文件中的内容加载到`DemoContainer`中的属性`properties`中。

- 当`DemoContainer`启动之后，我们需要获取到一个beanId为`bean.id.lady`的对象实例，可以通过`DemoContailner`的`getBean(String beanId)`方法来获取到对象实例

- `getBean()`方法中，我们分为三个部分：

    1. 查看`personFactoryMap`容器中是否存在`beanId`为`bean.id.lady`的实例，如果存在直接返回；如果不存在，重新实例化一个该bean放入到`personFactoryMap`容器中，并返回。

    2. 在容器初始化时，我们读取了配置文件`application.properties`，并将其内容保存在容器属性`properties`中,这一步我们需要通过`beanId`在配置文件`properties`中查询该`beanId`对应的`className`。

    3. 通过`className`，使用`反射机制`去创建对应的对象实例。

## 总结

&ensp;&ensp;&ensp;&ensp;经过示例我们发现，IOC的基本技术就是`反射`，可以`通过类名来动态的创建对象实例`，而且这种方式可以在对象在需要使用的时候才会去动态创建，而并不需要在一开始就来进行实例化。

&ensp;&ensp;&ensp;&ensp;在实际的使用过程中，我们也仅仅只是需要在配置文件中将beanId与class等等进行配置，来提高实例化对象时的灵活性和可维护性。

&ensp;&ensp;&ensp;&ensp;在Spring IOC容器中，我们可以同时可以把容器看成是一个工厂，在配置文件中我们定义好需要生产的对象，然后利用反射机制，根据配置文件中给出的类名生成相应的对象，`根据bean的Id或者name将bean反转注入到该对象被需求的地方`。

**注意：** 上述例子中并没有对scope进行讲解，scope为Singleton时，Bean在容器初始化时创建，以后每次调用都会从缓存（内存）中读取，对于scope为prototype时，每次调用都会重新去实例化该bean对象。

## 上一篇

[Spring Bean配置及管理](/document/SpringBean配置及管理.md)

## 下一篇

[Spring AOP](/document/SpringAOP.md)
