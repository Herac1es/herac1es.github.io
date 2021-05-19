---
title: "设计模式-单例模式"
date: 2021-05-19T20:14:40+08:00
tags: ["design pattern", "Go"]
categories: ["Go语言学习","设计模式学习笔记"]
draft: false
---

## 单例模型

### 定义

单例（Singleton）模式的定义：指一个类只有一个实例，且该类能自行创建这个实例的一种模式。例如，Windows 中只能打开一个任务管理器，这样可以避免因打开多个任务管理器窗口而造成内存资源的浪费，或出现各个窗口显示内容的不一致等错误。

在计算机系统中，还有 Windows 的回收站、操作系统中的文件系统、多线程中的线程池、显卡的驱动程序对象、打印机的后台处理服务、应用程序的日志对象、数据库的连接池、网站的计数器、Web 应用的配置对象、应用程序中的对话框、系统中的缓存等常常被设计成单例。

单例模式在现实生活中的应用也非常广泛，例如公司 CEO、部门经理等都属于单例模型。J2EE 标准中的 ServletContext 和 ServletContextConfig、Spring框架应用中的 ApplicationContext、数据库中的连接池等也都是单例模式。

单例模式有 3 个特点：

1. 单例类只有一个实例对象；
1. 该单例对象必须由单例类自行创建；
1. 单例类对外提供一个访问该单例的全局访问点。

### 优缺点

单例模式的优点：

单例模式可以保证内存里只有一个实例，减少了内存的开销。
可以避免对资源的多重占用。
单例模式设置全局访问点，可以优化和共享资源的访问。
单例模式的缺点：

单例模式一般没有接口，扩展困难。如果要扩展，则除了修改原来的代码，没有第二种途径，违背开闭原则。
在并发测试中，单例模式不利于代码调试。在调试过程中，如果单例中的代码没有执行完，也不能模拟生成一个新的对象。
单例模式的功能代码通常写在一个类中，如果功能设计不合理，则很容易违背单一职责原则。

### 应用场景

单例模式的应用场景主要有以下几个方面:

* 需要频繁创建的一些类，使用单例可以降低系统的内存压力，减少 GC。
* 某类只要求生成一个对象的时候，如一个班中的班长、每个人的身份证号等。
* 某些类创建实例时占用资源较多，或实例化耗时较长，且经常使用。
* 某类需要频繁实例化，而创建的对象又频繁被销毁的时候，如多线程的线程池、网络连接池等。
* 频繁访问数据库或文件的对象。
* 对于一些控制硬件级别的操作，或者从系统上来讲应当是单一控制逻辑的操作，如果有多个实例，则系统会完全乱套。
* 当对象需要被共享的场合。由于单例模式只允许创建一个对象，共享该对象可以节省内存，并加快对象访问速度。如 Web 中的配置对象、数据库的连接池等。

### Go语言实现

单例对象定义:
```go
type Singleton struct {
	name string
}
```

1. 饿汉式

不管三七二十一，在程序启动时就创建好，故称 "饿汉"，简单粗暴。

```go
var singletonHungry *Singleton

// 通过 go 的 init 函数侵入
func init() {
	singletonHungry = &Singleton{name: "hungry"}
}

func GetInstanceHungry() *Singleton {
	return singletonHungry
}
```

2. 懒汉式

与饿汉式不同，此时不在程序启动时创建，而是将初始化延迟到第一次使用的时候，即懒加载。

懒汉式经典实现方式：双重检测（需要加锁）

```go
var (
	singletonLazy *Singleton
	mutex sync.Mutex
)

func GetInstanceLazy() *Singleton {
	if singletonLazy == nil {
		mutex.Lock()
		// 再次判断的原因是并发情况下在外层判断为 nil 到获取锁这中间，可能已经被别的工作者初始化了
		if singletonLazy == nil {
			singletonLazy = &Singleton{name: "lazy"}
		}
		mutex.Unlock()
	}
	return singletonLazy
}
```

在这里给出一种使用 atomic 原子操作来实现的版本，类似于双重检测，但是不需要加 Mutex 重锁。

```go
var (
	singletonLazy *Singleton
	epoch         int64
)

func GetInstanceLazy() *Singleton {
	if singletonLazy == nil && atomic.AddInt64(&epoch, 1) == 1 {
		singletonLazy = &Singleton{name: "lazy"}
	}
	return singletonLazy
}
```

最后一种懒汉式实现就是借助与 sync.Once 来保证初始化只执行一次。

```go
var (
	singletonLazyII *Singleton
	once            sync.Once
)

func GetInstanceLazyII() *Singleton {
	if singletonLazyII == nil {
		once.Do(func() {
			singletonLazyII = &Singleton{name: "lazy II"}
		})
	}
	return singletonLazyII
}

```

### 总结

单例模式应该是设计模式里常规并且容易理解的一种了，一般应用场景就是节省资源、创建全局的枚举对象或者重要资源的初始化（如基础库的全局默认对象）。
谢谢观看～



