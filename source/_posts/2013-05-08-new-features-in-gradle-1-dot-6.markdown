---
layout: post
title: "new features in gradle 1.6"
date: 2013-05-08 15:40
comments: true
external-url: 
categories: gradle
---

##  值得关注的Gradle 1.6 新特性

[Gradle](http://www.gradle.org) 1.6刚刚发布了, 其中有几个值得关注的新特性 (注: 详细的内容还是请参考[Release Note](http://www.gradle.org/docs/current/release-notes)).

### 指定一个task必须在另外一个task之后运行

Task之间的被调用的顺序是会影响整个build的结果的, 例如_compileTest_ 需要在 _compileJava_ 之后运行才有意义, _clean_ 只能在 _build_之前运行, 不然就会把build出来的东西给clean掉了.

所以, Gradle很早就给出了一个 _dependsOn_ 的方法, 也就是当两个task需要按顺序被调用的时候, 可以调用这个方法来显式的声明, 例如

	taskA.dependsOn(taskB)

这样, 在调用_taskA_的时候, 就会先调用_taskB_了

但是, 这个_dependsOn_ 实际上会引起一些问题, 因为它不仅仅声明的是task之间的调用顺序, 而是声明了一种**依赖关系**.

也就是说, 这里声明的是 _taskA_ 依赖于 _taskB_, 显然是和 "_taskA_ 需要在 _taskB_ 之前运行"是有区别的.

还是用 _clean_ 和 _build_ 两个task来举例.

很显然, 在同时调用这两个task的时候, 我们希望 _clean_ 永远在 _build_ 之前被执行, 但是, 我们并不希望任何时候我们单独调用_build_ task的时候, _clean_ 都被执行.

这时候, 我们就不能够使用 _dependsOn_ 了, 而是需要1.6里面的新特性 _mustRunAfter_

可以这样声明这个关系

	build.mustRunAfter clean

### Build Setup Plugin

这个插件一直是我想要的, 简而言之, 就是类似 maven archetype plugin (但是现在还没有那么强大, 起码现在还不支持maven archetype 的模版功能).

这个插件的主要功能包括:

1. 创建build.gradle文件
2. 创建settings.gradle文件 (用于多模块化项目)	
3. 创建gradle wrapper
4. 转化maven项目为gradle项目

先来看看升成的文件吧

	.
	├── build.gradle
	├── gradle
	│   └── wrapper
	│       ├── gradle-wrapper.jar
	│       └── gradle-wrapper.properties
	├── gradlew
	├── gradlew.bat
	└── settings.gradle

以上就是在一个空的目录执行 `gradle setupBuild` 得到的内容, 注意, 虽然这个build setup plugin是一个插件, 但是, 我们不能(也不需要)像别的插件那样在_build.gradle_ 文件中apply它, 而是直接在命令行进行调用的.

这样就可以得到一个可用的gradle项目了.

这个插件还有个功能就是可以直接转化一个maven项目为gradle项目, 具体用法就是在一个包含有pom.xml的目录下执行 _setupBuild_, 但是我试了一下不行, 总是报NPE, 具体可以参见这个[问题](http://issues.gradle.org/browse/GRADLE-2666)