---
layout: post
title: "BootstrapServiceRegistry中提供的基础服务介绍"
date: 2013-01-07 22:08
comments: true
external-url: 
categories: [Hibernate]
---

## ClassloaderService

众所周知, classloader在Java里面是最基础和最容易出问题的地方了, 而这个service在Hibernate里面也出于最底层的位置, 位于BootstrapServiceRegistry当中.

Hibernate在内部有很多需要依赖到动态加载一个类的情况, 例如根据配置的Dialect来创建一个dialect实例, 这个dialect class即可以是hibernate自带的, 也可以是由用户自己提供的, 这时候, Hibernate就需要能够加载用户项目中提供的类, 在大多数Java SE的环境中, 这没有什么问题, 加载Hibernate自身的classloader通常和项目的classloader是同一个. 可是代码是跑在容器(包括Java EE 或者 OSGi)中的, 那情况就大大不同了.


以JBoss AS7 为例, 它在内部通过使用 [JBoss Modules](https://github.com/jbossas/jboss-modules) 来实现的模块化, 各个模块之间是相互独立的, 每个模块自己有一个classloader, 这一点在概念上是和OSGi的Bundle等同的.

并且各个模块之间只有声明了显示的依赖关系之后, 才能看到它所依赖模块中的类.

当一个JPA / Hibernate的项目(也是一个模块, 从JBoss Modules的角度来说)被部属到AS7当中的时候, AS7 的JPA 子系统会对项目扫描, 如果确认它使用了JPA (看有没有persistence.xml), 那么它会把内置的Hibernate模块注入到这个项目当中, 也就是, 这个项目模块会依赖hibernate模块, 这样, 项目代码中如果使用了Hibernate的类的话, 才不会抛出NPE.

但是, 这样有个问题, 如果这个项目对hibernate进行了扩展, 例如它使用了自己提供的一个dialect, 而不是hibernate内置的, 那么hibernate是没有办法初始化这个dialect的. 就是因为, hibernate的类是由它所在的module classloader加载的, 而它没有依赖项目的模块 (项目模块依赖了hibernate 模块), 所以它看不到定义在项目模块中的类.

为了解决这个问题, 我们需要让hibernate知道该项目所使用的module classloader, 这样, 当hibernate需要加载一个项目中的类的时候, 它可以让该项目所拥有的classloader去加载.

而这也正是AS7的JPA子系统所做的, 通过本章所介绍的 *ClassloaderService*

### 如何使用ClassloaderService

前面已经说过了, ClassloaderService是位于BootstrapServiceRegistry当中的, 所以, 它可以在任何层级的service registry当中被调用:

 
    ClassloaderService classloaderService = serviceRegistry.getService(ClassloaderService.class);


再来看看 *org.hibernate.boot.registry.classloading.spi.ClassLoaderService* 提供了那些方法呢:

* classForName 
* locateResource
* locateResourceStream
* locateResources
* loadJavaServices

以上方法的具体信息可参考javadoc, 基本上就是做类加载, resource 加载 和 加载java service (参见 [ServiceLoader](http://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html)

可以看到, 这个service把底层的classloader的细节都给屏蔽掉了, 当其它地方需要加载一个类的时候, 它可以直接调用这个service进行加载, 而不需要考虑当前所在的classloader和目标类可能所在的classloader等细节.

### ClassloaderService的实现

Hibernate 所提供的标准实现类是 *org.hibernate.boot.registry.classloading.internal.ClassLoaderServiceImpl*

具体的实现思路就是, 它内部保存了一个classloader的有序列表, 排序规则为:

1. 用户提供的classloader
2. Hibernate自身的classloader
3. TCCL -- 当前线程所使用的classloader
4. System classloader

当需要加载一个类的时候, 按照上面的顺序依次调用各个classloader, 直到某个classloader加载成功.

### ClassloaderService的扩展

最常用的扩展方式就是像AS7 的JPA子系统那样, 提供你自己的classloader给这个服务了, 这样, hibernate会调用你提供的classloader中去尝试加载一个类.

这是通过 *org.hibernate.boot.registry.BootstrapServiceRegistryBuilder\#with(ClassLoader classLoader)* 来完成的, 也就是, 先创建一个 *BootstrapServiceRegistryBuilder* 然后配置你所需要的classloader, 再创建 *BootstrapServiceRegistry*.

另外一种扩展是替换掉Hibernate内部的标准实现, 这就需要你创建一个自己的 *org.hibernate.boot.registry.classloading.spi.ClassLoaderService* 实现类了, 然后还需要写一个自己的 *org.hibernate.boot.registry.BootstrapServiceRegistryBuilder*.


## IntegratorService

Integrator service, 顾名思义, 是用于整合扩展的, 在以前没有这个service的时候, 如果想要同时使用Hibernate Envers / Validator / Search (或其它第三方的hibernate扩展模块), 那么我们需要在配置文件(hibernate.properties / persistence.xml)中定义各种各样的属性, 而这个服务, 让这些第三方的模块可以自动的被hibernate orm发现和配置.

在讲解这个service之前, 我们需要先对Java 的 [ServiceLoader](http://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html) 标准机制有所了解.

这个机制定义了一个标准来完成服务(这个更像是扩展的意思, 不要和前面所讲的hibernate的service弄混了)的发现, 让我们来用JPA举个例子吧:

你可能知道, JPA是一个Java EE的标准, 也就是, JPA只是定义了一些列的API, 而具体的实现代码由各个JPA Provider来完成. 

在JPA的代码中, 标准的创建一个EntityManagerFactory的方式是调用 *javax.persistence.Persistence\#createEntityManagerFactory* 这样, *Persistence* 类会创建一个 *EntityManagerFactory* 的实例, 但是, 因为JPA只负责指定API, 具体的实现类是由JPA provider所提供的, 所以, *Persistence* 类需要知道它要初始化哪个实现类.

一种情况是, 你的项目使用了JPA 并且使用Hibernate EntityManager作为实现, 那么可以在persistence.xml中加上

`<provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>`

来告诉JPA你想使用的provider 类是哪个.

但是这样做有点不方便的是, 如果你不想使用hibernate了, 想换到openjpa, 那么把hibernate的jar从classpath中移出, 把openjpa的jar添加进来之后, 还不能忘记修改这个persistence.xml.

并且, 可能很多人从来没有在自己的persistence.xml当中看到过\<provider\>这个元素, 可是项目还是运行的好好的, JPA也没有抱怨过不知道该使用哪个类来作为provider的入口.

这就是 *ServiceLoader* 在起作用了, 如果你解压缩任何一个jpa provider的jar的话, 例如, hibernate-entitymanager, 就可以发现 META-INF/services/目录下都有一个文件 *javax.persistence.spi.PersistenceProvider* , 这个文件的内容很简单, 只有一个类名 *org.hibernate.ejb.HibernatePersistence*.


*ServiceLoader* 的秘密也正在于此, 你定义一个API (在JPA这个例子中, 就是 javax.persistence.spi.PersistenceProvider) , 然后想要扩展这个API的模块在它的 META-INF/services中创建一个以这个API为名称的文本文件, 把扩展类的类名写在这个文件当中.

然后, 被扩展的模块(在这个例子中, 就是JPA了), 调用 ServiceLoader.load(Class apiClass, Classloader classloader), 这个方法会在classloader的范围内查找每个jar的 META-INF/services目录, 看里面有没有和 apiClass同名的文件, 如果有的话, 则读取其中的内容.

IntegratorService 的原理也正是如此, 我们定义了一个供第三方模块扩展的API -- *org.hibernate.integrator.spi.Integrator*, 而IntegratorService中只有一个方法:

`public Iterable<Integrator> getIntegrators();`    

也就是, 返回所有的integrator.

提供自己的Integrator扩展有两种方法, 一种是如同上面所说的, 通过service loader的机制, 例如[这里](https://github.com/hibernate/hibernate-orm/blob/master/hibernate-envers/src/main/resources/META-INF/services/org.hibernate.integrator.spi.Integrator) 就是hibernate envers所做的扩展.

另外一种方式, 是通过*org.hibernate.boot.registry.BootstrapServiceRegistryBuilder\#with(Integrator integrator)*

### Integrator

介绍了半天Integrator的扩展机制, 那么什么是Integrator呢?

让我们先来看看这个API所定义的方法:




    /**
     * Perform integration.
     *
     * @param configuration The configuration used to create the session factory
     * @param sessionFactory The session factory being created
     * @param serviceRegistry The session factory's service registry
     */
    public void integrate(
            Configuration configuration,
            SessionFactoryImplementor sessionFactory,
            SessionFactoryServiceRegistry serviceRegistry);
    
    /**
     * Tongue-in-cheek name for a shutdown callback.
     *
     * @param sessionFactory The session factory being closed.
     * @param serviceRegistry That session factory's service registry
     */
    public void disintegrate(SessionFactoryImplementor sessionFactory, SessionFactoryServiceRegistry serviceRegistry);



这个Integrator是在构造SessionFactory的时候被调用的, 已 *org.hibernate.envers.event.EnversIntegrator* 为例, 可以看到, 如果在classpath中包含了hibernate-envers的Jar, 并且 org.hibernate.envers.event.EnversIntegrator\#AUTO\_REGISTER 属性没有被设置成*false*的话, 那么, 就会自动的注册envers的监听器, 不需要任何配置了.

(注意, 如果你的entity中没有定义任何envers的annotation的话, 那么即使有这些监听器被自动注册上, 也不会有任何性能上的损耗的)





    public void integrate(
            Configuration configuration,
            SessionFactoryImplementor sessionFactory,
            SessionFactoryServiceRegistry serviceRegistry) {
        final boolean autoRegister = ConfigurationHelper.getBoolean( AUTO*REGISTER, configuration.getProperties(), true );
        if ( !autoRegister ) {
            LOG.debug( "Skipping Envers listener auto registration" );
            return;
        }

        EventListenerRegistry listenerRegistry = serviceRegistry.getService( EventListenerRegistry.class );
        listenerRegistry.addDuplicationStrategy( EnversListenerDuplicationStrategy.INSTANCE );

        final AuditConfiguration enversConfiguration = AuditConfiguration.getFor( configuration, serviceRegistry.getService( ClassLoaderService.class ) );

        appendListeners( listenerRegistry, enversConfiguration );
    }



而著名的[Joda](http://joda.sourceforge.net/) 也使用了Hibernate Integrator的机制来自动的扩展hibernate内部的date time类型, 用户只需要把这个jar也放在classpath中, 就可以直接使用joda所提供的类型了, 避免了还需要先在hibernate当中进行typedef的步骤.

需要指出的是, 因为Integrator是在构造SessionFactory的时候被调用的, 所以可能不是所有的SessionFactory的属性都已经构造完成了在Integrator的实现类中.

事实上, 在Hibernate内部, 已经广泛的使用了service loader机制, 并不仅限于*Integrator* API.

### org.hibernate.service.spi.ServiceContributor

这个即使另外一种对 *StandardServiceRegistry* 进行扩展的方法, 在 *org.hibernate.boot.registry.StandardServiceRegistryBuilder* 中有如下的方法



    private void applyServiceContributors() {
        LinkedHashSet<ServiceContributor> serviceContributors =
                bootstrapServiceRegistry.getService( ClassLoaderService.class ).loadJavaServices( ServiceContributor.class );
        for ( ServiceContributor serviceContributor : serviceContributors ) {
            serviceContributor.contribute( this );
        }
    }



所以, 想要扩展StandardServiceRegistry的话, 除了直接调用 *StandardServiceRegistryBuilder* 中的:

* addService
* addInitiator

方法之外, 也可以使用service loader的方法, 在自己项目的META-INF/services中创建一个名为 *org.hibernate.service.spi.ServiceContributor* 的文件, 然后在里面添加上这个API的实现类, 这个接口只有一个方法, 即



    public void contribute(StandardServiceRegistryBuilder serviceRegistryBuilder);



这样, 你可以有一个独立的jar来对service registry进行扩展, 它只要在classpath上, 就可以被hibernate自动发现并且加载了.

### 其它基于service loader的扩展点

在hibernate orm的metamodel branch当中, 我们还提供了如下的基于service loader的扩展点, 当前, 由于metamodel branch还在紧张的开发当中, 就先不详细介绍了, 等metamodel branch开发完毕之后吧, 有兴趣的可以先checkout这个分支看看, 它将会成为Hibernate ORM 5.

* org.hibernate.metamodel.spi.MetadataSourcesContributor
* org.hibernate.metamodel.spi.TypeContributor
* org.hibernate.metamodel.spi.MetadataContributor
* org.hibernate.metamodel.spi.AdditionalJaxbRootProducer


## StrategySelector

BootstrapServiceRegistry 中的最后一个服务就是StrategySelector了, 需要指出的是, 这个服务还没有被集成进4.1的发布版本当中, 它只存在于master 和metamodel两个分支中, 在Hibernate 4.2 和 5 中将被发布.

大家知道, Hibernate支持很多配置信息, 例如使用哪个dialect(1), 使用什么二级缓存, 使用哪个JTA, 连接池等等.

注1 : dialect 属性不是必须的, 如果没有给出dialect属性的话, hibernate会调用 *org.hibernate.engine.jdbc.dialect.spi.DialectResolver* (这也是一个标准服务)来通过数据库的元数据信息进行解析, 来获取对应的dialect.

在配置的时候, 以 *hibernate.transaction.jta.platform* 为例, 我们希望即能够给出具体的实现类的类名, 也可以使用一个简短的名字, 甚至, 在直接调用Configuration的时候, 使用一个jta platform的实例来作为此属性的值.

所以, 在配置文件中, 既可以写

 

    hibernate.transaction.jta.platform = org.hibernate.service.jta.platform.internal.JBossAppServerJtaPlatform



也可以写成



    hibernate.transaction.jta.platform =  JBossAS



也可以写成



    Configuration cfg = new Configuration();
    cfg.setProperty("hibernate.transaction.jta.platform", new JBossAppServerJtaPlatform());
    


同样的, 有了这个service, 在指定dialect的时候, 就可以写成



    hibernate.dialect = Oracle9i
    


        
而 StrategySelector 的作用正是在这里, 当其它地方从配置信息中取出一个自己需要的属性的时候, 交给 StrategySelector 来转化出属性值对应的对象, 还是以上面jta platform为例, *org.hibernate.engine.transaction.jta.platform.internal.JtaPlatformInitiator* 负责从配置文件中解析出正确的JtaPlatform的实现类的实例:



    public JtaPlatform initiateService(Map configurationValues, ServiceRegistryImplementor registry) {
            final Object setting = configurationValues.get( AvailableSettings.JTA*PLATFORM );
            final JtaPlatform platform = registry.getService( StrategySelector.class ).resolveStrategy( JtaPlatform.class, setting );
            if ( platform == null ) {
                LOG.debugf( " No JtaPlatform was specified, using default [%s]", NoJtaPlatform.class.getName() );
                return new NoJtaPlatform();
            }
            return platform;
        }



可以看到, 上面的方法中, 它先从 configurationValues 中获取 "hibernate.transaction.jta.platform" 的值, 上面说过了, 这个属性值可以是:

* org.hibernate.service.jta.platform.internal.JBossAppServerJtaPlatform
* JBossAS
* new JBossAppServerJtaPlatform()

然后把这个值传给strategy selector 进行转化.

### StrategySelector是如何工作的

我们把上面的属性, 属性值, 属性的简短名字称为strategy, 由 *org.hibernate.boot.registry.selector.Availability* 表示, 主要包括:

* strategy role
* strategy implementation
* selector name

用Dialect为例:

* strategy role -- org.hibernate.dialect.Dialect
* strategy impl -- 各种数据库的dialect 实现
* selector name -- mysql / oracle 等等简单方便的名字

三者之间的关系是:

* role 对应多个 name (通常是实现类的类名和一个简单的名字)
* role 对应多个 impl
* impl 对应多个name


在StrategySelector中, 则保存了一个大的 *Map\<Class,Map\<String,Class\>\> namedStrategyImplementorByStrategyMap*

这个map用strategy role作为key, value 也是一个Map\<String,Class\>, 这个value map则用selector name作为key, strategy impl作为value.

需要指出的是, strategy selector在解析的时候, 如果找不到short name对应的class 值, 则会尝试把这个short name当成class name去加载, 所以, 即使一个strategy没有在这里被注册, 而配置的时候只用了全类名也是没有问题的.

### 添加自己的 strategy

因为StrategySelector是BootstrapServiceRegistry中的一个service, 并且它提供了一个 *registerStrategyImplementor* 方法, 所以, 可以直接获取这个service然后调用注册的方法:


    StrategySelector s = serviceRegistry.getService(StrategySelector.class);
    s.registerStrategyImplementor(JtaPlatform.class, "my custom platform", MyCustomJtaPlatform.class);



另外, 我们还提供了 *org.hibernate.boot.registry.selector.AvailabilityAnnouncer* 它只有一个


    public Iterable<Availability> getAvailabilities();
    


的方法, 用户也可以实现这个接口, 把要自定义的stragety都封装成 *Availability*.

有了 *AvailabilityAnnouncer* 的实现类之后, 又有两种方法可以把这个信息添加到StrategySelector service中:

AvailabilityAnnouncer 是我们提供的另外一个使用service loader机制的扩展, 所以只需要按照上面所介绍的步骤, 把实现类的名称写入 META-INF/services/org.hibernate.boot.registry.selector.AvailabilityAnnouncer 文件中, 它就会自动被加载

* 使用 service loader   
* 通过 org.hibernate.boot.registry.BootstrapServiceRegistryBuilder\#withStrategySelectors方法


## 总结

至此, Hibernate在BootstrapServiceRegistry中提供的三个最基本的服务已经介绍完了, 其中, ClassloaderService 让hibernate能够工作在任何的类加载环境中, 并且对上层代码屏蔽了具体的细节; IntegratorService能够自动的发现并集成第三方的扩展内容, 而StrategySelector, 进一步简化了hibernate的配置信息.

希望对大家在使用hibernate的过程中有所帮助.  
     
  
