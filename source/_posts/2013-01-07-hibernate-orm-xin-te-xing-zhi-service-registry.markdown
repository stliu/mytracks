---
layout: post
title: "Hibernate ORM 新特性之 Service(Registry)"
date: 2013-01-07 22:07
comments: true
external-url: 
categories: [Hibernate]
---
### Service Registry
已经迁移到 *Hibernate Core 4.0* 的用户(非JPA)可能已经注意到，以前大家熟知的构造 *SessionFactory* 的方法已经不推荐使用了：  

   
    Configuration configuration * new Configuration();
    SessionFactory sf * configuration.buildSessionFactory();

在 *Hibernate ORM 4* 里面推荐的方式是 *org.hibernate.cfg.Configuration\#buildSessionFactory(ServiceRegistry serviceRegistry)*, 需要先构造一个 *ServiceRegistry* 对象。

那么, 神马是 *ServiceRegistry*?

顾名思义, *ServiceRegistry* 是 *Service* 的注册表, 它为*Service*提供了一个统一的加载 / 初始化 / 存放 / 获取机制.


首先来看看一个整体的类图:

![p1](https://raw.github.com/stliu/random-data/master/blog/h4-serviceregistry/p1.jpg)

整个 *ServiceRegistry* 结构和我们大家了解的 *Classloader* 的代理结构类似, 只不过和 *Classloader* 的首先代理给父节点，父节点找不到再从本级的 *Classloader* 中查找的策略不同，*ServiceRegistry* 是先从本级节点查找，找不到再去父亲中查找。

![p2](https://raw.github.com/stliu/random-data/master/blog/h4-serviceregistry/p3.jpg)

可以看到hibernate里面的 *ServiceRegistry*实际上是由三层(范围)组成的:

* BootstrapServiceRegistry

    主要提供 Classloading 和 Service loading 服务, 供全局使用.

* ServiceRegistry

    标准的*ServiceRegistry*, 同样服务于全局, 在初始化这个Registry中的服务的时候,可能需要使用到*BootstrapServiceRegistry*中的服务.    

* SessionFactoryServiceRegistry

    于*SessionFactory*相关联(一对一的关系), 这里面的Services在初始化的时候需要访问*SessionFactory*.


所以, *BootstrapServiceRegistry* 中主要包括全局都要使用的基础服务, 而 *ServiceRegistry* 主要包括在 *BootstrapServiceRegistry* 之上的, 和Hibernate 相关的服务, 例如配置文件解析, JDBC, JNDI 等.

而, *SessionFactoryServiceRegistry* 则是和sessionfactory相关的了, 包括2LC的region factory, event listener等.

这里面的层级关系, 对于 service registry的client来说是完全透明的, 你可以直接调用*getService* 去获取你需要的service, 而不用去关心你在哪个层级上, 如同 classloader.


### Service

那么，什么是 *Service* ? 简单的来说，*Service* 就是一个功能，它由一个接口(*Service Role*)和一个实现类组成.

*Service* 在Hibernate里面并不是一个新的概念，在以前的版本里面，我们已经有了这些东西，而这次，是通过把这些服务的提供者(类)，给抽取成一个个的*Service*,从而可以用统一的方式进行访问/加载/初始化等.

下图列出了所有的Hibernate中内置的*Services*, 详细介绍请[参考](http://docs.jboss.org/hibernate/core/4.1/devguide/en-US/html*single/#d0e3329):

![p3](https://raw.github.com/stliu/random-data/master/blog/h4-serviceregistry/p2.jpg)


### Building ServiceRegistry

在本文的开头即说过, 现在标准的构造一个*SessionFactory*的做法是传递一个*ServiceRegistry*实例给*Configuration\#buildSessionFactory*方法, 那么, 这里就需要我们首先创建一个 *ServiceRegistry*的实例. 

最简单的做法是:


    StandardServiceRegistryBuilder serviceRegistryBuilder = new StandardServiceRegistryBuilder();
    ServiceRegistry serviceRegistry = serviceRegistryBuilder.build();
    SessionFactory sf = configuration.buildSessionFactory(serviceRegistry);


让我们来继续分析这里面究竟发生了什么.

如上文所介绍的, *ServiceRegistry*是一个代理结构, 那么, 实际上是需要分别创建 *BootstrapServiceRegistry*, *ServiceRegistry* 和 *SessionFactoryServiceRegistry*.

#### BootstrapServiceRegistry

*org.hibernate.service.ServiceRegistryBuilder* 有两个构造函数, 一个是接受一个 *BootstrapServiceRegistry*实例作为参数, 另外一个是无参的, 在内部自己构造一个 *org.hibernate.boot.registry.internal.BootstrapServiceRegistryImpl*的实例.

如同上面所讲的, 事实上我们是先build了一个 BootstrapServiceRegistry 的.

而Hibernate也如你所愿的提供了 *org.hibernate.boot.registry.BootstrapServiceRegistryBuilder*.

通过这个builder类, 我们可以通过调用如下的方法来对一些基础服务进行扩展:

* with(Integrator integrator)
* with(ClassLoader classLoader)
* withStrategySelector 

(关于这里的扩展的详细解释敬请期待下一篇)

现在我们有了 *BootstrapServiceRegistryBuilder* 和 *ServiceRegistryBuilder* , 那么很自然的, 会想到是不是也有一个 *SessionFactoryServiceRegistryBuilder* 呢?

很遗憾, 没有!

是的, 不过我们提供了 *org.hibernate.service.spi.SessionFactoryServiceRegistryFactory*, 它是一个位于 *ServiceRegistry* 中的标准service. 

这主要是因为它的全部工作就是创建一个没啥意思的 *org.hibernate.service.internal.SessionFactoryServiceRegistryImpl* 实例, 也没啥可扩展的地方, 所以自然也就不需要一个builder了.

那么, 如果你非得要替换到这个默认的实现, 或者, 更进一步, 例如你想要替换掉某一个Service的标准实现, 异或, 你想把自己写的service也添加到service registry当中呢?


### 更进一步来看看Service

首先, BootstrapServiceRegistry中所提供的service是固定的, 无法增减, 除非你提供一个自己的 *org.hibernate.boot.registry.internal.BootstrapServiceRegistryImpl* 实现类, 但是并不推荐.

从*org.hibernate.boot.registry.internal.BootstrapServiceRegistryImpl\#locateServiceBinding* 方法中我们可以看到BootStrapServiceRegistry所提供的标准服务.


    public <R extends Service> ServiceBinding<R> locateServiceBinding(Class<R> serviceRole) {
        if ( ClassLoaderService.class.equals( serviceRole ) ) {
            return (ServiceBinding<R>) classLoaderServiceBinding;
        }
        else if ( StrategySelector.class.equals( serviceRole) ) {
            return (ServiceBinding<R>) strategySelectorBinding;
        }
        else if ( IntegratorService.class.equals( serviceRole ) ) {
            return (ServiceBinding<R>) integratorServiceBinding;
        }
        return null;
    }

推荐的扩展位置是 *ServiceRegistry*, 在这里, 你可以增加你自定义的service, 替换标准的service实现等等. 

*ServiceRegistry* 中所提供的标准服务是定义在 *org.hibernate.service.StandardServiceInitiators* 当中:


    public class StandardServiceInitiators {
        public static List<StandardServiceInitiator> LIST = buildStandardServiceInitiatorList();
    
        private static List<StandardServiceInitiator> buildStandardServiceInitiatorList() {
            final List<StandardServiceInitiator> serviceInitiators = new ArrayList<StandardServiceInitiator>();
    
            serviceInitiators.add( ConfigurationServiceInitiator.INSTANCE );
            serviceInitiators.add( ImportSqlCommandExtractorInitiator.INSTANCE );
    
            serviceInitiators.add( JndiServiceInitiator.INSTANCE );
            serviceInitiators.add( JmxServiceInitiator.INSTANCE );
    
            serviceInitiators.add( PersisterClassResolverInitiator.INSTANCE );
            serviceInitiators.add( PersisterFactoryInitiator.INSTANCE );
    
            serviceInitiators.add( ConnectionProviderInitiator.INSTANCE );
            serviceInitiators.add( MultiTenantConnectionProviderInitiator.INSTANCE );
            serviceInitiators.add( DialectResolverInitiator.INSTANCE );
            serviceInitiators.add( DialectFactoryInitiator.INSTANCE );
            serviceInitiators.add( BatchBuilderInitiator.INSTANCE );
            serviceInitiators.add( JdbcEnvironmentInitiator.INSTANCE );
            serviceInitiators.add( JdbcServicesInitiator.INSTANCE );
            serviceInitiators.add( RefCursorSupportInitiator.INSTANCE );
    
            serviceInitiators.add( SchemaManagementToolInitiator.INSTANCE );
    
            serviceInitiators.add( MutableIdentifierGeneratorFactoryInitiator.INSTANCE);
    
            serviceInitiators.add( JtaPlatformInitiator.INSTANCE );
            serviceInitiators.add( TransactionFactoryInitiator.INSTANCE );
    
            serviceInitiators.add( SessionFactoryServiceRegistryFactoryInitiator.INSTANCE );
    
    
            return Collections.unmodifiableList( serviceInitiators );
        }
    }


可以看到, 每一个标准服务都被封装成了 *StandardServiceInitiator*, 那么什么是 *ServiceInitiator* 呢?

#### ServiceInitiator

`org.hibernate.service.spi.ServiceInitiator` 定义了一个 *Service* 的加载器, 这个接口里面只有一个方法


    /**
     * Obtains the service role initiated by this initiator.  Should be unique within a registry
     *
     * @return The service role.
     */
    public Class<R> getServiceInitiated();


而*org.hibernate.service.StandardServiceInitiator* 继承了 *org.hibernate.service.spi.ServiceInitiator* 并提供了另外一个方法:


    /**
     * Initiates the managed service.
     *
     * @param configurationValues The configuration values in effect
     * @param registry The service registry.  Can be used to locate services needed to fulfill initiation.
     *
     * @return The initiated service.
     */
    public R initiateService(Map configurationValues, ServiceRegistryImplementor registry);
    
    
认同下图所示, ServiceInitiator定义了一个Service的接口 以及如何初始化这个service.

![p4](https://raw.github.com/stliu/random-data/master/blog/h4-serviceregistry/p4.jpg)

所以, 当有了ServiceInitiator之后, 可以通过调用*org.hibernate.boot.registry.StandardServiceRegistryBuilder* 的方法添加到注册表中:

* addInitiator(StandardServiceInitiator initiator)
* addService(final Class serviceRole, final Service service)


hibernate在初始化这些service的时候, 会先初始化内置的, 所以, 如果你想要替换以后的标准service的话, 只需要原样调用上面两个方法之一就行了.


Hibernate 还提供了一些接口来供*Service*的创建者来使用(具体如何使用请参考javadoc，在此就不重复了)：

* org.hibernate.service.spi.Configurable
* org.hibernate.service.spi.Manageable
* org.hibernate.service.spi.ServiceRegistryAwareService
* org.hibernate.service.spi.InjectService
* org.hibernate.service.spi.Startable
* org.hibernate.service.spi.Stoppable
* org.hibernate.service.spi.Wrapped


至此, 已经把ServiceRegistry中的大部分内容介绍完了, 希望大家对此有个感性的认识. 在以后的blog中, 我会按照从bootstrap, standard, sessionfactory的顺序把各个service registry中提供的标准服务一一解释, 我会尽量的从代码级别来解释为什么这样设计, 和如何使用.