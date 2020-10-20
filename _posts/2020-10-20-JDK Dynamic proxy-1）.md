---
layout:     post
title:      "浅谈JDK动态代理（上）"
date:       2020-10-20 00:00:00
author:     "looveh"
header-img: "img/post-bg-rwd.jpg"
tags:
    - Web
    - 闲谈
    - 设计模式
    - Java

---

动态代理。这四个字一出来，估计很多初学者已经开始冒冷汗。动态代理之所以给人感觉很难，有三点原因：

- 代码形式很诡异，让人搞不清调用逻辑
- 用到了反射，而很多初学者不了解反射
- 包含代理设计模式的思想，本身比较抽象

尽管动态代理看起来似乎有一定难度，但却必须拿下。因为Spring的事务控制依赖于AOP，AOP底层实现便是动态代理，环环相扣。到最后，还是看基本功。

主要内容：

- 一个小需求：给原有方法添加日志打印
- 静态代理实现日志打印
- 静态代理的问题

------

## 一个小需求：给原有方法添加日志打印

假设现在我们有一个类Calculator，代表一个计算器，它可以进行加减乘除操作

```java
public class Calculator {

	//加
	public int add(int a, int b) {
		int result = a + b;
		return result;
	}

	//减
	public int subtract(int a, int b) {
		int result = a - b;
		return result;
	}

	//乘法、除法...
}
```

现有一个需求：在每个方法执行前后打印日志。你有什么好的方案？





**直接修改**

很多人最直观的想法是直接修改Calculator类：

```java
public class Calculator {

	//加
	public int add(int a, int b) {
		System.out.println("add方法开始...");
		int result = a + b;
		System.out.println("add方法结束...");
		return result;
	}

	//减
	public int subtract(int a, int b) {
		System.out.println("subtract方法开始...");
		int result = a - b;
		System.out.println("subtract方法结束...");
		return result;
	}

	//乘法、除法...
}
```

上面的方案是有问题的：

1. 直接修改源程序，不符合开闭原则。应该对扩展开放，对修改关闭
2. 如果Calculator有几十个、上百个方法，修改量太大
3. 存在重复代码（都是在核心代码前后打印日志）
4. 日志打印硬编码在代理类中，不利于后期维护：比如你花了一上午终于写完了，组长告诉你这个功能取消，于是你又要打开Calculator花十分钟删除日志打印的代码！

所以，此种方案PASS！



## **静态代理实现日志打印**

“静态代理”四个字包含了两个概念：静态、代理。我们先来了解什么叫“代理”，至于何为“静态”，需要和“动态”对比着讲。

> 代理是一种模式，提供了对目标对象的间接访问方式，即通过代理访问目标对象。如此便于在目标实现的基础上增加额外的功能操作，前拦截，后拦截等，以满足自身的业务需求。
> 引用博客：[Java静态代理和动态代理 - 纪煜楷 - 博客园](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/jiyukai/p/6958744.html)

![img](https://pic3.zhimg.com/80/v2-6e2fa8c8c02e0f04a601cdd951045f82_720w.jpg)引用自：Java静态代理和动态代理 - 纪煜楷 - 博客园

常用的代理方式可以粗分为：静态代理和动态代理。

静态代理的实现比较简单：编写一个代理类，实现与目标对象相同的接口，并在内部维护一个目标对象的引用。通过构造器塞入目标对象，在代理对象中调用目标对象的同名方法，并添加前拦截，后拦截等所需的业务功能。

按上面的描述，代理类和目标类需要实现同一个接口，所以我打算这样做：

- 将Calculator抽取为接口
- 创建目标类CalculatorImpl实现Calculator
- 创建代理类CalculatorProxy实现Calculator



接口

```java
/**
 * Calculator接口
 */
public interface Calculator {
	int add(int a, int b);
	int subtract(int a, int b);
}
```

目标对象实现类

```java
/**
 * 目标对象实现类，实现Calculator接口
 */
public class CalculatorImpl implements Calculator {

	//加
	public int add(int a, int b) {
		int result = a + b;
		return result;
	}

	//减
	public int subtract(int a, int b) {
		int result = a - b;
		return result;
	}

	//乘法、除法...
}
```

代理对象实现类

```java
/**
 * 代理对象实现类，实现Calculator接口
 */
public class CalculatorProxy implements Calculator {
        //代理对象内部维护一个目标对象引用
	private Calculator target;
        
        //构造方法，传入目标对象
	public CalculatorProxy(Calculator target) {
		this.target = target;
	}

        //调用目标对象的add，并在前后打印日志
	@Override
	public int add(int a, int b) {
		System.out.println("add方法开始...");
		int result = target.add(a, b);
		System.out.println("add方法结束...");
		return result;
	}

        //调用目标对象的subtract，并在前后打印日志
	@Override
	public int subtract(int a, int b) {
		System.out.println("subtract方法开始...");
		int result = target.subtract(a, b);
		System.out.println("subtract方法结束...");
		return result;
	}

	//乘法、除法...
}
```

使用代理对象完成加减乘除，并且打印日志

```java
public class Test {
	public static void main(String[] args) {
		//把目标对象通过构造器塞入代理对象
		Calculator calculator = new CalculatorProxy(new CalculatorImpl());
		//代理对象调用目标对象方法完成计算，并在前后打印日志
		calculator.add(1, 2);
		calculator.subtract(2, 1);
	}
}  
```

![静态代理示意图](https://pic3.zhimg.com/80/v2-1332d491e2600b21c3ec28454eab3d9e_720w.jpg)

静态代理的优点：可以在不修改目标对象的前提下，对目标对象进行功能的扩展和拦截。但是它也仅仅解决了上一种方案4大缺点中的第1点：

1. 直接修改源程序，不符合开闭原则。应该对扩展开放，对修改关闭 <font color=green>**√**</font>
2. 如果Calculator有几十个、上百个方法，修改量太大 <font color=red>**×**</font>
3. 存在重复代码（都是在核心代码前后打印日志）<font color=red>**×**</font>
4. 日志打印硬编码在代理类中，不利于后期维护：比如你花了一上午终于写完了，组长告诉你这个功能取消，于是你又要打开Calculator花十分钟删除全部新增代码！<font color=red>**×**</font>



## 静态代理的问题

上面案例中，代理类是我们事先编写的，而且要和目标对象类实现相同接口。由于CalculatorImpl（目标对象）需要日志功能，我们即编写了CalculatorProxy（代理对象），并通过构造器传入CalculatorImpl（目标对象），调用目标对象同名方法的同时添加增强代码。

但是这里有个问题！代理对象构造器的参数类型是Calculator，这意味着它只能接受Calculator的实现类对象，亦即我们写的代理类CalculatorProxy只能给Calculator做代理，它们绑定死了！

如果现在我们系统需要全面改造，给其他类也添加日志打印功能，就得为其他几百个接口都各自写一份代理类...

![img](https://pic3.zhimg.com/80/v2-5f71691211f838b5e17dd016277be0e6_720w.png)



自己手动写一个类并实现接口实在太麻烦了。仔细一想，**我们其实想要的并不是代理类，而是代理对象！**那么，能否让JVM根据接口自动生成代理对象呢？

比如，有没有一个方法，我传入接口，它就给我自动返回代理对象呢？

![img](https://pic4.zhimg.com/80/v2-04157e707582d34233d6ae0eb5bf6867_720w.jpg)

答案是肯定的。

下一篇我们来看看JDK如何自动生成代理对象！



