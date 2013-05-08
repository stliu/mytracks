---
layout: post
title: "Jandex--快速高效的解析annotation"
date: 2013-01-23 21:29
comments: true
external-url: 
categories: [jandex]
---

## Jandex 介绍

[Jandex](https://github.com/jbossas/jandex) -- Java Annotation Indexer -- 正如其名字所示, 是用来索引annotation的. 在Java项目当中, Annotation是用的越来越多了, 大有取代XML之势, 甚至, 在Java EE6中, 绝大多数规范都已经把annotation作为首要配置, 而XML辅助了.

那么, 这时候一个问题就出现了, 如何快速高效的解析出annotation (要知道, annotation本质上还是定义一些元数据信息而已, 我们的目的终究还是读出这些元数据所表示的信息)

首先, JDK 提供的标准的读取annotation是通过 _java.lang.reflect.AnnotatedElement_ 提供的, 主要方法如下

* `boolean getAnnotation(Class<A> annotationClass)`
* `Annotation[] getAnnotations()`
* `boolean isAnnotationPresent(Class<? extends Annotation> annotationClass)`
* `Annotation[] getDeclaredAnnotations()`

JDK的方式不是本文的重点, 但是需要指出的是, JDK读取annotation是通过反射的方法在运行时读取的, 所以不可避免的会比较缓慢.

这种方式还有一个缺点, 就是, 如果通过Class对象来获取这个Class内部定义的Annotations之后, 这些annotations是会被缓存在Class对象内部的, 但是我们知道, 对于同一个Class, 如果它们被不同的classloader分别加载(这种情况对于一般的项目来说比较少见, 多发生于应用服务器环境或者OSGi的环境), 那么, 每个classloader加载的Class对象都个不相同, 也就是都要解析annotations的话, 那么需要反射多次, 而没法利用首次加载好的缓存信息.

而这通常是重复的操作, 因为annotation从本质上来讲, 它只是元数据信息, 和classloader是无关的.

而Jandex提出了另外的解决思路, 那就是, 既然通过运行时反射的方法比较慢, 那么我们就在运行时之前, 也就是编译期(或者是运行期)把需要的信息全都读取并且保存起来, 下次再用的时候, 就不需要再次解析了, 可以直接读取已经解析好的数据了就, 从而达到提升性能的效果.

正是这个思路, JBoss AS 7在编译的时候, 就为它所依赖的每个Jar都通过Jandex创建了相应的索引文件, 例如, 在 *jboss-as-7.1.1.Final/modules/org/hibernate/main*, 我们可以看到里面每个jar文件都对应着一个index文件.

* hibernate-commons-annotations-4.0.1.Final.jar        
* hibernate-commons-annotations-4.0.1.Final.jar.index  
* hibernate-core-4.0.1.Final.jar                      
* hibernate-core-4.0.1.Final.jar.index                 
* hibernate-entitymanager-4.0.1.Final.jar
* hibernate-entitymanager-4.0.1.Final.jar.index
* hibernate-infinispan-4.0.1.Final.jar
* hibernate-infinispan-4.0.1.Final.jar.index
* hibernate-infinispan-4.0.1.Final.jar.index
* module.xml

这样, 在AS 7启动的时候, 就不需要再通过反射的方式去读取annotation的信息了.

上面是AS 7自身所依赖的Jar, 事实上, AS 7还会对第一次部署到它内部的应用也创建索引文件, 这样, 重启的时候, 就会节省很多时间.

最后, 索引文件格式的设计使其能够被非常高效的读取和尽可能少的占用内存.


### 使用Jandex

使用Jandex非常的简单, 入口类是 _org.jboss.jandex.Indexer_, 而这个类只有两个方法:

* `ClassInfo index(InputStream stream)`
* `Index complete()`

也就是, 把所有你关心的定义了annotation(或者没有定义annotation, 详情见下面)的class, 都依次读入到一个 _InputStream_ 当中, 然后交给 _Indexer#index_ 方法, 都index完成之后, 调用 _Indexer#complete_ 的方法通知 _Indexer_ 完成, 它就会返回一个 _IndexView_ 对象.

_IndexView_ 对象可以被看成所有被index的class的一个总的注册表, 我们可以从中获取我们想要的annotation的信息, 以及其它的一些信息.

Jandex还同时提供了 _IndexWriter_ 和 _IndexReader_ 两个类, 可以方便我们把索引完成的 _IndexView_ 保存到文件系统, 不同于java所提供的默认序列化和反序列化支持, _IndexWriter_ 被设计的更加高效和紧凑.

最后, Jandex还提供了 _org.jboss.jandex.JarIndexer_ 和 _org.jboss.jandex.JandexAntTask_ , 分别是能够直接索引一个jar和一个ANT task, 方便大家使用.

从上面可以看到, 实际上Jandex既可以被用来在编译期对class/jar进行索引, 也可以直接在运行期进行索引.


![jandex-arch](/images/blog/jandex.png)

首先, 上图中, 除了 _Indexer_ 之外的所有的类都被设计成了不可变(Immutable)类, 所以它们自然也都是线程安全的, 可以自然的被多个线程同时访问.

注意, 这个设计是很重要的, 尤其对于大型项目

