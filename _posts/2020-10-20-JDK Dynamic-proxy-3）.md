---
layout:     post
title:      "浅谈JDK动态代理（下）"
date:       2020-10-20 00:00:00
author:     "looveh"
header-img: "img/post-bg-rwd.jpg"
tags:
    - Web
    - 闲谈
    - 设计模式
    - Java

---

介绍完JDK动态代理，今天和大家一起做个小案例：模拟Spring的事务管理。

主要内容：

- 熟悉的陌生人
- 山寨AOP事务需求分析
- AOP事务具体代码实现

------

## 熟悉的陌生人

面试官如果问“请你谈谈你对Spring的理解”，估计很多人会脱口而出：IOC和AOP。IOC大概是大家对Spring最直接的印象，就是个大容器，装了很多bean，还会帮你做依赖注入。

![img](https://pic4.zhimg.com/80/v2-3a80286900fb71285ce1d7e0a08a821b_720w.jpg)IOC

但是对于AOP，很多人其实没有太多概念，一时不知道Spring哪里用了AOP。好像事务用了切面，但具体又不了解。这样吧，我问你一个问题，我自己写了一个UserController，以及UserServiceImpl implements UserService，并且在UserController中注入Service层对象：

```java
@Autowired
private UserService userService;
```

那么，这个userService一定是我们写的UserServiceImpl的实例吗？

如果你听不懂我要问什么，说明你本身对Spring的了解还是太局限于IOC。

实际上，Spring依赖注入的对象并不一定是我们自己写的类的实例，也可能是userServiceImpl的代理对象。下面分别演示这两种情况：

- 注入userServiceImpl对象

![img](https://pic1.zhimg.com/80/v2-b8adf7e9002d647076bb660c4b499ac4_720w.jpg)

注入的是UserServiceImpl类型

- 注入userServiceImpl的代理对象（CGLib动态代理）

![img](https://pic2.zhimg.com/80/v2-dd932daec51739a7ea584e4a2baf6e9d_720w.jpg)

注入的是CGLib动态代理生成的userServiceImpl的代理对象



为什么两次注入的对象不同？

因为第二次我给UserServiceImpl加了@Transactional 注解。

![img](https://pic1.zhimg.com/80/v2-0fc9c01248f4e4db1a832629020bd0ac_720w.jpg)

此时Spring读取到这个注解，便知道我们要使用事务。而我们编写的UserService类中并没有包含任何事务相关的代码。如果给你，你会怎么做？动态代理嘛！

但是要用动态代理完成事务管理，还需要自己编写一个通知类，并把通知对象传入代理对象，通知负责事务的开启和提交，并在代理对象内部调用目标对象同名方法完成业务功能。

![img](https://pic3.zhimg.com/80/v2-32cec15654c51f7ce1277b026aed8676_720w.jpg)

我们能想到的方案，Spring肯定也知道。同样地，Spring为了实现事务，也编写了一个通知类，TransactionManager。利用动态代理创建代理对象时，Spring会把transactionManager织入代理对象，然后将代理对象注入到UserController。

所以我们在UserController中使用的userService其实是代理对象，而代理对象才支持事务。

------

## 山寨AOP事务需求分析

了解了Spring事务的大致流程后，我们再来分析一下自己如何编写一个山寨的AOP事务。

AOP事务，有两个概念：AOP和事务。

事务，大家已经很熟悉，这里主要讲讲什么是AOP。AOP，它是**Aspect-Oriented Programming（面向切面编程）**的英文缩写。什么是面向切面编程？有时直接介绍一个东西是什么，可能比较难。但是一说到它是干嘛的，大家就立即心领神会了。

我们的系统中，常常存在交叉业务，比如事务、日志等。UserService的method1要用到它，BrandService的method2也要用到它。一个交叉业务就是要切入系统的一个方面。具体用代码展示就是：

![img](https://pic2.zhimg.com/80/v2-3f05c3673c38046fd80c84d32c33aa91_720w.jpg)这个切面，可以是日志，也可以是事务

交叉业务的编程问题即为面向切面编程。AOP的目标就是使交叉业务模块化。可以将切面代码移动到原始方法的周围：

![img](https://pic4.zhimg.com/80/v2-3d94c58c9161409d893b243b5efa185f_720w.jpg)

原先不用AOP时，交叉业务直接写在**方法内部的前后**，用了AOP交叉业务写在**方法调用前后**。这与AOP的底层实现方式有关：动态代理其实就是代理对象调用目标对象的同名方法，并在调用前后加增强代码。不过这两种最终运行效果是一样的。

而所谓的模块化，我个人的理解是将切面代码做成一个可管理的状态。比如日志打印，不再是直接硬编码在方法中的零散语句，而是做成一个通知类，通过通知去执行切面代码。

所以，现在需求已经很明确，我们需要一个通知类（TransactionManager）执行事务，一个代理工厂帮助生成代理对象，然后利用动态代理将事务代码织入代理对象的各个方法中。

就好比下面三个Service，原先是没有开启事务的：

![img](https://pic1.zhimg.com/80/v2-4588c1cd36e7471ac6513048e653c070_720w.jpg)

我们希望最终达到的效果是，我加了个@MyTransactional后，代理工厂给我返回一个代理对象：

![img](https://pic1.zhimg.com/80/v2-aa5de8985d9c4928daf524fdb70c3164_720w.jpg)代理工厂使用动态代理，为每一个目标对象创建一个代理对象

细节分析：

![img](https://pic2.zhimg.com/80/v2-033cb1891a8ca9d6c1ba6bc7827c24e5_720w.jpg)txManager其实是在目标对象test()方法的前后执行事务，而不是方法内部的前后

也就是说，代理对象方法 = 事务 + 目标对象方法。



另外，还有个棘手的问题：事务操作，必须使用同一个Connection对象。如何保证？第一次从数据源获取Connection对象并开启事务后，将它存入当前线程的ThreadLocal中，等到了DAO层，还是从ThreadLocal中取，这样就能保证开启事务和操作数据库使用的Connection对象是同一个。

![img](https://pic4.zhimg.com/80/v2-dede0e353501c76ab863fc64704146ef_720w.jpg)

![img](https://pic4.zhimg.com/80/v2-802a9d1e100f41639a19b61878fb4c07_720w.jpg)

![img](https://pic1.zhimg.com/80/v2-7f9f4d1e23c752e57c27b5d00ed1385c_720w.jpg)开启事务后，Controller并不是直接调用我们自己写的Service，而是Spring提供的代理对象

这就是事务的实现原理。

------

## AOP事务具体代码实现

ConnectionUtils工具类

```java
package com.demo.myaopframework.utils;

import org.apache.commons.dbcp.BasicDataSource;

import java.sql.Connection;

/**
 * 连接的工具类，它用于从数据源中获取一个连接，并且实现和线程的绑定
 */
public class ConnectionUtils {

    private ThreadLocal<Connection> tl = new ThreadLocal<Connection>();

    private static BasicDataSource dataSource = new BasicDataSource();

    //静态代码块,设置连接数据库的参数
    static{
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        dataSource.setUrl("jdbc:mysql://localhost:3306/test");
        dataSource.setUsername("root");
        dataSource.setPassword("123456");
    }


    /**
     * 获取当前线程上的连接
     * @return
     */
    public Connection getThreadConnection() {
        try{
            //1.先从ThreadLocal上获取
            Connection conn = tl.get();
            //2.判断当前线程上是否有连接
            if (conn == null) {
                //3.从数据源中获取一个连接，并且存入ThreadLocal中
                conn = dataSource.getConnection();
                tl.set(conn);
            }
            //4.返回当前线程上的连接
            return conn;
        }catch (Exception e){
            throw new RuntimeException(e);
        }
    }

    /**
     * 把连接和线程解绑
     */
    public void removeConnection(){
        tl.remove();
    }
}
```

AOP通知（事务管理器）

```java
package com.demo.myaopframework.utils;

/**
 * 和事务管理相关的工具类，它包含了，开启事务，提交事务，回滚事务和释放连接
 */
public class TransactionManager {

    private ConnectionUtils connectionUtils;

    public void setConnectionUtils(ConnectionUtils connectionUtils) {
        this.connectionUtils = connectionUtils;
    }

    /**
     * 开启事务
     */
    public  void beginTransaction(){
        try {
            connectionUtils.getThreadConnection().setAutoCommit(false);
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    /**
     * 提交事务
     */
    public  void commit(){
        try {
            connectionUtils.getThreadConnection().commit();
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    /**
     * 回滚事务
     */
    public  void rollback(){
        try {
            connectionUtils.getThreadConnection().rollback();
        }catch (Exception e){
            e.printStackTrace();
        }
    }


    /**
     * 释放连接
     */
    public  void release(){
        try {
            connectionUtils.getThreadConnection().close();//还回连接池中
            connectionUtils.removeConnection();
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

自定义注解

```text
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface MyTransactional {
}
```

Service

```java
public interface UserService {
	void getUser();
}

 
public class UserServiceImpl implements UserService {
	@Override
	public void getUser() {
		System.out.println("service执行...");
	}
}
```

实例工厂

```java
public class BeanFactory {

	public Object getBean(String name) throws Exception {
		//得到目标类的Class对象
		Class<?> clazz = Class.forName(name);
		//得到目标对象
		Object bean = clazz.newInstance();
		//得到目标类上的@MyTransactional注解
		MyTransactional myTransactional = clazz.getAnnotation(MyTransactional.class);
		//如果打了@MyTransactional注解，返回代理对象，否则返回目标对象
		if (null != myTransactional) {
			ProxyFactoryBean proxyFactoryBean = new ProxyFactoryBean();
			TransactionManager txManager = new TransactionManager();
			txManager.setConnectionUtils(new ConnectionUtils());
			//装配通知和目标对象
			proxyFactoryBean.setTxManager(txManager);
			proxyFactoryBean.setTarget(bean);
			Object proxyBean = proxyFactoryBean.getProxy();
			//返回代理对象
			return proxyBean;
		}
		//返回目标对象
		return bean;
	}
}
```

代理工厂

```java
public class ProxyFactoryBean {
	//通知
	private TransactionManager txManager;
	//目标对象
	private Object target;

	public void setTxManager(TransactionManager txManager) {
		this.txManager = txManager;
	}

	public void setTarget(Object target) {
		this.target = target;
	}

	//传入目标对象target，为它装配好通知，返回代理对象
	public Object getProxy() {
		Object proxy = Proxy.newProxyInstance(
				target.getClass().getClassLoader(),/*1.类加载器*/
				target.getClass().getInterfaces(), /*2.目标对象实现的接口*/
				new InvocationHandler() {/*3.InvocationHandler*/
					@Override
					public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
						try {
							//1.开启事务
							txManager.beginTransaction();
							//2.执行操作
							Object retVal = method.invoke(target, args);
							//3.提交事务
							txManager.commit();
							//4.返回结果
							return retVal;
						} catch (Exception e) {
							//5.回滚事务
							txManager.rollback();
							throw new RuntimeException(e);
						} finally {
							//6.释放连接
							txManager.release();
						}

					}
				}
		);
		return proxy;
	}

}
```

代码结构

![img](https://pic4.zhimg.com/80/v2-c373ef0577a0dd70c46ee42f418c4c87_720w.jpg)

得到普通UserService：

![img](https://pic4.zhimg.com/80/v2-6f7b8a900a24c268acfca2ec9b4854f7_720w.jpg)

给UserServiceImpl添加@MyTransactional注解，得到代理对象：

![img](https://pic4.zhimg.com/80/v2-1d5cb07752e6b0b1b956cbe07594e1ab_720w.jpg)

![img](https://pic1.zhimg.com/80/v2-56f29cdcbe1ed734ab17b1e55b0d7a18_720w.jpg)