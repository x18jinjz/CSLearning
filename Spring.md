## IOC & DI 

![1526888208254](.\image\1526888208254.png)

IOC：控制反转，指某一接口具体实现类的选择控制权从调用类中移除，转交给第三方决定 

传统方式：组件向容器请求，容器返回资源反转控制方式：容器直接推送资源给组件，组件以某方式接受依赖注入。

DI是IOC的一种表述方式：组件以一些预定好的方式接受容器资源注入。 

### IOC类型

从注入方法划分，可以分为三种：构造函数注入，属性注入和接口注入。     

* 构造函数注入通过调用类的构造函数，将接口实现类通过构造函数变量传入。    
* 属性注入有选择地通过 Setter 方法完成调用类所需依赖的注入，更加灵活方便。    
* 接口注入（效果和属性注入并无本质区别，不推荐）将调用类所有依赖注入的方法抽取到一个接口中，调用类通过实现该接口提供相应的注入方法。 

### 通过容器完成依赖关系的注入

第三方容器帮助完成类的初始化与装配工作，让开发者从这些底层实现类的实例化、依赖关系装配等工作中脱离出来。

Spring 就是这样的一个容器，它通过配置文件或注解描述类和类之间的依赖关系，自动完成类的初始化和依赖注入的工作。将配置信息作为需要操控目标类的元信息，利用Java的反射机制进行相应的调用操作，实例化Bean并建立Bean间的依赖关系，从容器中即可返回准备就绪的 Bean 实例，后续可直接使用。 

### IOC容器

在读取Bean配置创建Bean实例之前，必须先实例化IOC容器，之后才可以从IOC容器里获取Bean实例并使用。 

IoC容器两种实现：`BeanFactory`（常称IoC容器） 和 `ApplicationContext`（常称Spring容器） 

* Spring 中所说的 Bean 比JavaBean 更宽泛，所有可以被 Spring 容器实例化并管理的 Java 类都可以成为 Bean。
* Bean 工厂（ com.springframework.beans.factory.BeanFactory）是 Spring 框架最核心的接口，它提供了高级 IoC 的配置机制。
* 应用上下文（ com.springframework.context.ApplicationContext）建立在 BeanFactory 基础之上，提供了更多面向应用的功能。
* 对于两者的用途，我们可以进行简单划分： BeanFactory 是 Spring 框架的基础设施，面向 Spring 本身； ApplicationContext 面向使用 Spring 框架的开发者，几乎所有的应用场合我们都直接使用 ApplicationContext 而非底层的 BeanFactory。
* ApplicationContext 的初始化和 BeanFactory 有一个重大的区别： BeanFactory在初始化容器时，并未实例化 Bean，直到第一次访问某个 Bean 时才实例目标 Bean； 而ApplicationContext 则在初始化应用上下文时就实例化所有单实例的 Bean。 

### BeanFactory

* `BeanFactory` 接口是Spring最框架最核心的接口，是类的通用工厂，它可以创建并管理各种类的对象。这些可被创建和管理的对象， Spring 称为 Bean。

* 必须在类路径下提供Log4j才不会报错。
* 一个经典实现类：XmlBeanFactory 。

### ApplicationContext

* `BeanFactory`中大多以编程方式实现，`ApplicationContext`中可以通过配置实现。

* 主要实现类：ClassPathXmlApplicationContext和FileSystemApplicationContext。 

### WebApplicationContext

* 专门为Web应用准备，允许从相对于Web根目录的路径中装载配置文件完成初始化。

* web应用上下文会做为属性放置到ServletContext中，以便Web应用环境可以访问Spring应用上下文。初始化必须要有ServletContext实例，也就是必须在拥有Web容器的前提下才能完成启动的工作。可以在web.xml中配置自启动的Servlet或者定义Web容器监听器（ContextLoaderListener）来启动。

* ```xml
  <!--在启动Web容器时，自动装配Spring applicationContext.xml的配置信息--> 
  <listener> 
  	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class> </listener>  
  <context-param> 
  	<param-name>contextConfigLocation</param-name> 
  	<param-value>/applicationContext.xml</param-value> 
  </context-param> 
  ```

  或者

  ```xml
  <context-param> 
      <param-name>contextConfigLocation</param-name> 
      <param-value>/WEB-INF/baobaotao-dao.xml, /WEB-INF/baobaotao-service.xml </param-value> </context-param> 
  <!--声明自动启动的Servlet --> 
  <servlet> 
      <servlet-name>springContextLoaderServlet</servlet-name> 
      <servlet-class>org.springframework.web.context.ContextLoaderServlet </servlet-class> 
      <!--②启动顺序--> 
      <load-on-startup>1</load-on-startup> 
  </servlet>    
  ```

### Bean配置方式

基于XML、基于注解、基于JAVA类  

* XML通过全类名（反射机制，所以必需有无参数的构造器）

* 基于注解

  在配置文件ApplicationContext中需加上

  ```xml
  <!--扫描基类包下的所有类中的注解以定义bean-->
  
  <context:component-scan base-package="com.czf"/> 
  ```

  * @Component
  * @Repository
  * @Controller
  * @Service
  * @Autowired：实现Bean的依赖注入

### Bean的作用域

* 默认 singleton：容器初始化时创建bean实例，在容器生命周期内只创建这一个bean，单例的，即每次请求IOC容器获取该bean时都是同一个bean实例。
* prototype：即容器初始化时不创建bean实例，每次向容器请求时才创建新的bean实例并返回。
* **WebApplication新增3个scope**：
* request：每次请求创建一个新的Bean。
* session：同一个HTTP Session共享一个Bean。
* globalSession：同一个全局Session共享一个Bean。 



## AOP

主要用于将性能监测、访问控制、日志记录等非业务逻辑从业务代码中横向抽取出来，为这些无法通过纵向继承体系进行抽象的重复性代码提供了解决方案。

**切点**：用于定位连接点，使用类和方法作为连接点的查询条件（准确的说应该是执行点，连接点等于执行点加方位）。

**增强**：织入到目标类连接点上的一段程序代码。还包含着和连接点相关的信息：执行点的方位。

**引介**：一种特殊的增强，为类添加一些属性和方法。

**织入**：将增强添加到目标类具体连接点上的过程。包括编译器织入、类装载期织入、动态代理织入。Spring采用动态代理织入。

**代理**：一个类被织入增强后，成为了融合原类和增强逻辑的代理类。代理类可能是和原类具有相同接口的类，也可能是原类的子类，所以可以采用调用原类相同的方式调用代理类。

**切面**：由切点和增强（引介）组成，包括横切逻辑的定义和连接点的定义，Spring AOP就是负责实施切面的框架，将却面所定义的横切逻辑织入到切面指定的连接点。 

### JDK动态代理

只能对接口进行代理。

### CGLib动态代理

可以为一个类创建子类，但是因为是动态创建子类的方式生成代理对象，所以无法对final方法代理。

### 增强

* 前置增强：实现`MethodBeforeAdvice`，重写`before`方法。
* 后置增强：实现`AfterReturningAdvice`，重写`afterReturning`方法。
* 环绕增强：实现`MethodInterceptor`，重写`invoke`方法，拦截目标方法并加入增强代码。
* 异常抛出增强：实现`ThrowsAdvice`，重写`afterThrowing`方法。
* 引介增强：为目标类创建新的属性和方法。继承`DelegatingIntroductionInterceptor`并实现所需要增加的接口，再重写接口中的方法，以及重写`invoke`方法拦截目标方法调用并添加增强代码。

### 切点、切面

单独使用增强时增强会被织入目标类的所有方法，结合切点切面则可以有选择的织入到特定方法上。

**切点**：由`Pointcut`类描述，`Pointcut`由`ClassFilter`和`MethodMatcher`构成，通过`ClassFilter`定位到特定类上，通过`MethodMatcher`定位到特定方法。

**切面**：可以只包括增强，也可以是增强+切点。由`Advisor`描述，分为`Advisor`（一般切面，只包含一个增强）、`PointcutAdvisor`（具有切点的切面）和 IntroductionAdvisor （引介切面）。

* 静态方法匹配切面：实现`StaticMethodMatcherPointcutAdvisor`，重写`matches`和`getClassFilter`方法，再配合一个增强使用。

* 静态正则表达式方法匹配切面：直接在配置中使用`RegexpMathodPointcutAdvisor`，定义匹配的正则表达式。

* 动态切面：通过`DefaultPointcutAdvisor`和`DynamicMethodMatcherPointcut`实现。继承`DynamicMethodMatcherPointcut`并重写`getClassFilter`、`matches（Method method,Class clazz）`和`matches（Method method,Class clazz, Object[] args）`方法，再在配置文件中配合`DefaultPointcutAdvisor`和一个增强使用。

* 流程切面：为某个方法A中调用的其他方法织入增强。通过`DefaultPointcutAdvisor`和`ControllFlowPointcut`实现。在配置文件中配置`DefaultPointcutAdvisor`并指定号目标方法A，再配置`ControllFlowPointcut`以及相应增强。

* 复合切点切面：`ComposablePointcut`提供了将两个切点组合的运算。

  ```java
  ComposablePointcut cp = new ComposablePointcut();
  Pointcut p1 = new ControllFlowPointcut(WaiterDelegate.class,"service"); 
  NameMatchMethodPointcut p2 = new NameMatchMethodPointcut();
  p2.addMethodName("greetTo");
  // 两个切点进行交集运算
  return cp.intersection(p1).intersection(p2);
  ```

  

* 引介切面：在配置文件中配置一个`DefaultIntroductionAdvisor`并给其配置一个引介增强。

## SpringMVC

### 体系架构

![1526976936064](.\image\1526976936064.png)

### DispatcherServlet

负责接收HTTP请求并协调SpringMVC各个组件完成处理请求的工作，在web.xml中配置。

```xml
    ......
	<!--启动业务层和持久层的Spring配置文件，被父Spring容器所使用-->
	<listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/applicationContext.xml</param-value>
    </context-param>
	
	<!--默认加载/WEB-INF/<servletname>-servlet.xml配置文件，启动Web层的Spring容器-->
	<!--Web层的Spring容器将作为业务层Spring容器的子容器，因此可以访问业务层容器中的Bean-->
    <servlet>  
        <servlet-name>dispatcher</servlet-name>  
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>  
        <load-on-startup>1</load-on-startup>  
    </servlet>  
    <servlet-mapping>  
        <servlet-name>dispatcher</servlet-name>  
        <url-pattern>/</url-pattern>  
    </servlet-mapping>  

```

可以配置多个`DispatcherServlet`，`DispatcherServlet`会装配好一套默认的可用组件。

