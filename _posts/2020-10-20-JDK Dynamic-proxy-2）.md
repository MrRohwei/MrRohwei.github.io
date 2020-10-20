---
layout:     post
title:      "浅谈JDK动态代理（中）"
date:       2020-10-20 00:00:00
author:     "looveh"
header-img: "img/post-bg-rwd.jpg"
tags:
    - Web
    - 闲谈
    - 设计模式
    - Java

---

接上篇[浅谈JDK动态代理（上）](https://github.lvbok.com/2020/10/20/JDK-Dynamic-proxy-1/)

这篇文章咬咬牙能看完的话，再看其他动态代理的文章就轻松愉快多了。希望想搞懂动态代理的同学，能坚持下去。

主要内容：

- 前情提要
- 接口创建对象的可行性分析
- 动态代理
- Proxy.getProxyClass()的秘密
- 编写可生成代理和可插入通知的通用方法
- 类加载补充
- 小结
- 彩蛋

文章较长，有点啰嗦了，希望看简化版的戳下方链接：[Java 动态代理作用是什么？](https://www.zhihu.com/question/20794107/answer/658139129)

## **前情提要**

假设现在项目经理有一个需求：在项目现有所有类的方法前后打印日志。

你如何在**不修改已有代码的前提下**，完成这个需求？



## **静态代理**

具体做法如下：

1.为现有的每一个类都编写一个**对应的**代理类，并且让它实现和目标类相同的接口（假设都有）

![img](https://pic3.zhimg.com/80/v2-5f71691211f838b5e17dd016277be0e6_720w.png)

2.在创建代理对象时，通过构造器塞入一个目标对象，然后在代理对象的方法内部调用目标对象同名方法，并在调用前后打印日志。也就是说，**代理对象 = 增强代码 + 目标对象（原对象）**，有了代理对象后，就不用原对象了

![img](https://pic1.zhimg.com/80/v2-c9fe2378193d6727d291708e1c6b3740_720w.jpg)



**静态代理的缺陷**

程序员要手动为每一个目标类，编写对应的代理类。如果当前系统已经有成百上千个类，工作量太大了。所以，现在我们的努力方向是：如何少写或者不写代理类，却能完成代理功能？



## **接口创建对象的可行性分析**

**复习对象的创建过程**

首先，在很多初学者的印象中，类和对象的关系是这样的：

![img](https://pic4.zhimg.com/80/v2-61a28889025be57193e270f57372ebcf_720w.jpg)

虽然知道源代码经过javac命令编译后会在磁盘中得到字节码文件（.class文件），也知道java命令会启动JVM将字节码文件加载进内存，但也仅仅止步于此了。至于从字节码文件加载进内存到堆中产生对象，期间具体发生了什么，他们并不清楚。

所谓“万物皆对象”，字节码文件也难逃“被对象”的命运。它被加载进内存后，JVM为其创建了一个对象，以后所有该类的实例，皆以它为模板。这个对象叫Class对象，它是Class类的实例。

![img](https://pic3.zhimg.com/80/v2-cc19377f2d5c9dc4a722c74ee9afc7e2_720w.jpg)



大家想想，Class类是用来描述所有类的，比如Person类，Student类...那我如何通过Class类创建Person类的Class对象呢？这样吗：

```text
Class clazz = new Class();
```

好像不对吧，我说这是Student类的Class对象也行啊。有点晕了...

**其实，程序员是无法自己new一个Class对象的，它仅由JVM创建。**

![img](https://pic3.zhimg.com/80/v2-b92dfd598a20fe5c3db903465204cd02_720w.jpg)

- Class类的构造器是private的，杜绝了外界通过new创建Class对象的可能。当程序需要某个类时，JVM自己会调用这个构造器，并传入ClassLoader（类加载器），让它去加载字节码文件到内存，然后**JVM为其创建对应的Class对象**
- 为了方便区分，Class对象的表示法为：Class<String>，Class<Person>

所以借此机会，我们不妨换种方式看待类和对象：

![img](https://pic1.zhimg.com/80/v2-cd2d00047d06b3a7f9516a240f1fa0dc_720w.jpg)

也就是说，**要得到一个类的实例，关键是先得到该类的Class对象！**只不过new这个关键字实在太方便，为我们隐藏了底层很多细节，我在刚开始学习Java时甚至没意识到Class对象的存在。

**接口Class和类Class的区别**

来分析一下接口Class和类Class的区别。以Calculator接口的Class对象和CalculatorImpl实现类的Class对象为例：

```java
import java.lang.reflect.Constructor;
import java.lang.reflect.Executable;
import java.lang.reflect.Method;

public class ProxyTest {
	public static void main(String[] args) {
		/*Calculator接口的Class对象
                  得到Class对象的三种方式：1.Class.forName(xxx) 
                                           2.xxx.class 
                                           3.xxx.getClass()
                  注意，这并不是我们new了一个Class对象，而是让虚拟机加载并创建Class对象            
                */
		Class<Calculator> calculatorClazz = Calculator.class;
		//Calculator接口的构造器信息
		Constructor[] calculatorClazzConstructors = calculatorClazz.getConstructors();
		//Calculator接口的方法信息
		Method[] calculatorClazzMethods = calculatorClazz.getMethods();
		//打印
		System.out.println("------接口Class的构造器信息------");
		printClassInfo(calculatorClazzConstructors);
		System.out.println("------接口Class的方法信息------");
		printClassInfo(calculatorClazzMethods);

		//Calculator实现类的Class对象
		Class<CalculatorImpl> calculatorImplClazz = CalculatorImpl.class;
		//Calculator实现类的构造器信息
		Constructor<?>[] calculatorImplClazzConstructors = calculatorImplClazz.getConstructors();
		//Calculator实现类的方法信息
		Method[] calculatorImplClazzMethods = calculatorImplClazz.getMethods();
		//打印
		System.out.println("------实现类Class的构造器信息------");
		printClassInfo(calculatorImplClazzConstructors);
		System.out.println("------实现类Class的方法信息------");
		printClassInfo(calculatorImplClazzMethods);
	}

	public static void printClassInfo(Executable[] targets){
		for (Executable target : targets) {
			// 构造器/方法名称
			String name = target.getName();
			StringBuilder sBuilder = new StringBuilder(name);
			// 拼接左括号
			sBuilder.append('(');
			Class[] clazzParams = target.getParameterTypes();
			// 拼接参数
			for(Class clazzParam : clazzParams){
				sBuilder.append(clazzParam.getName()).append(',');
			}
			//删除最后一个参数的逗号
			if(clazzParams!=null && clazzParams.length != 0) {
				sBuilder.deleteCharAt(sBuilder.length()-1);
			}
			//拼接右括号
			sBuilder.append(')');
			//打印 构造器/方法
			System.out.println(sBuilder.toString());
		}
	}
}
```

运行结果：

![img](https://pic1.zhimg.com/80/v2-e5333f939edf9ebe6a3015f6176250b0_720w.jpg)

- 接口Class对象没有构造方法，所以Calculator接口不能直接new对象
- 实现类Class对象有构造方法，所以CalculatorImpl实现类可以new对象
- 接口Class对象有两个方法add()、subtract()
- 实现类Class对象除了add()、subtract()，还有从Object继承的方法

也就是说，接口和实现类的Class信息除了构造器，基本相似。

既然我们希望通过接口创建实例，就无法避开下面两个问题：

1.接口方法体缺失问题

> 首先，接口的Class对象已经得到，它描述了方法信息。
>
> 但它没方法体。
>
> 没关系，反正代理对象的方法是个空壳，只要调用目标对象的方法即可。
>
> JVM可以在创建代理对象时，随便糊弄一个空的方法体，反正后期我们会想办法把目标对象塞进去调用。
>
> 所以这个问题，勉强算是解决。

2.接口Class没有构造器，无法new

> 这个问题好像无解...毕竟这么多年了，的确没听哪位仁兄直接new接口的。
>
> 但是，仔细想想，接口之所以不能new，是因为它缺少构造器，它本身是具备完善的类结构信息的。就像一个武艺高强的大内太监（接口），他空有一身绝世神功（类结构信息），却后继无人。如果江湖上有一位妙手圣医，能克隆他的一身武艺，那么克隆人不就武艺高强的同时，还能生儿育女了吗？
> 所以我们就想，JDK有没有提供这么一个方法，比如getXxxClass()，我们传进一个接口Class对象，它帮我们克隆一个具有相同类结构信息，又具备构造器的新的Class对象呢？

至此，分析完毕，我们无法根据接口直接创建对象（废话）。

那动态代理是怎么创建实例的呢？它到底有没有类似getXxxClass()这样的方法呢？



## 动态代理

不错，动态代理确实存在getXxxClass()这样的方法。

我们需要java.lang.reflect.InvocationHandler接口和 java.lang.reflect.Proxy类的支持。Proxy后面会用到InvocationHandler，因此我打算以Proxy为切入点。首先，再次明确我们的思路：

![img](https://pic3.zhimg.com/80/v2-c761a16f8fac465237e235ffa6429caa_720w.jpg)

通过查看API，我们发现Proxy类有一个静态方法可以帮助我们。

![img](https://pic3.zhimg.com/80/v2-2b2e2c983b2214dfc525c2131cdf8cfa_720w.jpg)

Proxy.getProxyClass()：返回代理类的Class对象。终于找到妙手圣医。

也就说，只要传入目标类实现的接口的Class对象，getProxyClass()方法即可返回代理Class对象，而不用实际编写代理类。这相当于什么概念？

![img](https://pic4.zhimg.com/80/v2-68c4ded341f6d447758efa83dc560f77_720w.jpg)

废话不多说，开搞。

```java
import java.lang.reflect.Constructor;
import java.lang.reflect.Executable;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class ProxyTest {
	public static void main(String[] args) {
		/*
		 * 参数1：Calculator的类加载器（当初把Calculator加载进内存的类加载器）
		 * 参数2：代理对象需要和目标对象实现相同接口Calculator
		 * */
		Class calculatorProxyClazz = Proxy.getProxyClass(Calculator.class.getClassLoader(), Calculator.class);
		//以Calculator实现类的Class对象作对比，看看代理Class是什么类型
                System.out.println(CalculatorImpl.class.getName());
		System.out.println(calculatorProxyClazz.getName());
		//打印代理Class对象的构造器
		Constructor[] constructors = calculatorProxyClazz.getConstructors();
		System.out.println("----构造器----");
		printClassInfo(constructors);
		//打印代理Class对象的方法
		Method[] methods = calculatorProxyClazz.getMethods();
		System.out.println("----方法----");
		printClassInfo(methods);
	}

	public static void printClassInfo(Executable[] targets) {
		for (Executable target : targets) {
			// 构造器/方法名称
			String name = target.getName();
			StringBuilder sBuilder = new StringBuilder(name);
			// 拼接左括号
			sBuilder.append('(');
			Class[] clazzParams = target.getParameterTypes();
			// 拼接参数
			for (Class clazzParam : clazzParams) {
				sBuilder.append(clazzParam.getName()).append(',');
			}
			//删除最后一个参数的逗号
			if (clazzParams != null && clazzParams.length != 0) {
				sBuilder.deleteCharAt(sBuilder.length() - 1);
			}
			//拼接右括号
			sBuilder.append(')');
			//打印 构造器/方法
			System.out.println(sBuilder.toString());
		}
	}
}
```

运行结果：

![img](https://pic1.zhimg.com/80/v2-72dc0b06c56aad6924bff2946471e5a8_720w.jpg)

大家还记得接口Class的打印信息吗？

![img](https://pic1.zhimg.com/80/v2-246049bae5a3ca412102e0b1f9ec2ac8_720w.jpg)



也就是说，通过给Proxy.getProxyClass()传入类加载器和接口Class对象，我们得到了一个**加强版的Class**：即包含接口的方法信息add()、subtract()，又包含了构造器$Proxy0(InvocationHandler)，还有一些自己特有的方法以及从Object继承的方法。



梳理一下：

1.原先我们本打算直接根据接口Class得到代理对象，无奈接口Class只有方法信息，没有构造器

2.于是，我们想，有没有办法创建一个Class对象，既有接口Class的方法信息，同时又包含构造器方便创建代理实例呢？

3.利用Proxy类的静态方法getProxyClass()方法，给它传一个接口Class对象，它能返回一个加强版Class对象。也就是说getProxyClass()的本质是：**用Class，造Class。**

![img](https://pic1.zhimg.com/80/v2-2b77f72c33aa99d2b3ddf112ef269368_720w.jpg)



要谢谢Proxy类和JVM，让我们不写代理类却直接得到代理Class对象，进而得到代理对象。

![img](https://pic2.zhimg.com/80/v2-3979759089c5226bf8997b141109632d_720w.jpg)静态代理



![img](https://pic2.zhimg.com/80/v2-ccdc245db98539305abc7ffef9da46b5_720w.jpg)动态代理：用Class造Class



既然Class<$Proxy0>有方法信息，又有构造器，我们试着用它得到代理实例吧：

![img](https://pic2.zhimg.com/80/v2-f8faece26ded5e37b5c73ff623a8c9b5_720w.jpg)

我们发现，newInstance()创建对象失败。因为**Class的newInstance()方法底层会走无参构造器**。而之前打印$Proxy0的Class信息时，我们发现它没有无参构造，只有有参构造$Proxy0(InvocationHandler)。那就靠它了：

![img](https://pic1.zhimg.com/80/v2-3ee642093680a6ba5618c7065e3dc550_720w.jpg)constructor.newInstance()需要传入一个InvocationHandler对象，这里采用匿名对象的方式，invoke()方法不做具体实现，直接返回null

舒服~



## **Proxy.getProxyClass()的秘密**

**一个小问题**

好不容易通过Proxy.getProxyClass()得到代理Class，又通过反射最终得到代理对象，当然要玩一玩：

![img](https://pic1.zhimg.com/80/v2-6c7eb940665170e6ef1b62326884d220_720w.jpg)

尴尬，竟然发生了空指针异常。纵观整个代码，新写的add()和subtract()返回值是int，不会是空指针。而再往上的代码之前编译都是通过的，应该没问题啊。再三思量，我们发现匿名对象InvocationHandler的invoke()返回null。难道是它？做个实验：让invoke()返回1，然后观察结果。

![img](https://pic4.zhimg.com/80/v2-2c2d46da74320a71ad4946db9031912b_720w.jpg)结果代理对象的add和subtract都返回1

巧合吗？应该不是。我猜：**每次调用代理对象的方法都会调用invoke()，且invoke()的返回值就是代理方法的返回值。**如果真是如此，空指针异常就可以解释了：add()和suntract()期待的返回值类型是int，但是之前invoke()返回null，类型不匹配，于是空指针异常。

以防万一，再验证一下invoke()和代理对象方法的关系：

![img](https://pic1.zhimg.com/80/v2-3efdbb60fe5749035744be30540c6898_720w.jpg)

好了，什么都不用说了。就目前的实验来看，调用过程应该是这样：

![img](https://pic3.zhimg.com/80/v2-2fab867f05f180010504fa9dae862c42_720w.jpg)



**动态代理底层调用逻辑**

同样的，知道了结果后，我们再反推原理。

静态代理：往代理对象的构造器传入目标对象，然后代理对象调用目标对象的同名方法。

动态代理：constructor反射创建代理对象时，需要传入InvocationHandler，我猜，代理对象内部有一个成员变量InvocationHandler：

![img](https://pic2.zhimg.com/80/v2-13c4d59b7396a26921ef9b3f2a387701_720w.jpg)

果然不出所料。那么动态代理的大致设计思路就是：

![img](https://pic4.zhimg.com/80/v2-c152e0d68f387241dc4ff368ae8144af_720w.jpg)

为什么这么设计？

为了解耦，也为了通用性。

如果JVM生成代理对象的同时生成了特定逻辑的方法体，那这个代理对象后期就没有扩展的余地，只能有一种玩法。而引入InvocationHandler的好处是：

- JVM创建代理对象时不必考虑方法实现，只要造一个空壳的代理对象，舒服
- 后期代理对象想要什么样的方法实现，我写在invocationHandler对象的invoke()方法里送进来便是

所以，invocationHandler的作用，倒像是把“方法”和“方法体”分离。JVM只造一个空的代理对象给你，后面想怎么玩，由你自己组装。反正代理对象中有个成员变量invocationHandler，每一个方法里只有一句话：handler.invoke()。所以调任何一个代理方法，最终都会跑去调用invoke()方法。

invoke()方法是代理对象和目标对象的桥梁。

![img](https://pic1.zhimg.com/80/v2-9438d5f7907422f4ccb606d0af0689b8_720w.jpg)

但是我们真正想要的结果是：调用代理对象的方法时，去调用目标对象的方法。

所以，接下来努力的方向就是：**设法在invoke()方法得到目标对象，并调用目标对象的同名方法。**



**代理对象调用目标对象方法**

那么，如何在invoke()方法内部得到目标对象呢？我们来看看能不能从invoke()方法的形参上获取点线索：

- Object proxy：很遗憾，是代理对象本身，而不是目标对象（不要调用，会无限递归）
- Method method：本次被调用的代理对象的方法
- Obeject[] args：本次被调用的代理对象的方法参数

很可惜，proxy不是代理对象。其实想想也知道，创建代理对象的过程中自始至终没有目标对象参与，所以也就无法产生关联。而且一个接口可以同时被多个类实现，所以JVM也无法判断当前代理对象想要代理哪个目标对象。但好在我们已经知道本次调的方法名(Method)和参数(args)。我们接下来要做的就是得到目标对象并调用同名方法，然后把参数给它。

如何得到目标对象呢？没办法，为今之计只能new了...哈哈哈哈。我靠，饶了一大圈，又是动态代理，又是invoke()的，结果还是要手动new？别急，先玩玩。后面会改进的：

![img](https://pic1.zhimg.com/80/v2-1a37afa3de6ee63bbcd56aded88a3578_720w.jpg)



但是这样的写法显然是倒退30年，一夜回到解放前。我们需要改进一下，封装Proxy.getProxyClass()，使得目标对象可以作为参数传入：

```java
public class ProxyTest {
	public static void main(String[] args) throws Throwable {
		CalculatorImpl target = new CalculatorImpl();
                //传入目标对象
                //目的：1.根据它实现的接口生成代理对象 2.代理对象调用目标对象方法
		Calculator calculatorProxy = (Calculator) getProxy(target);
		calculatorProxy.add(1, 2);
		calculatorProxy.subtract(2, 1);
	}

	private static Object getProxy(final Object target) throws Exception {
		//参数1：随便找个类加载器给它， 参数2：目标对象实现的接口，让代理对象实现相同接口
		Class proxyClazz = Proxy.getProxyClass(target.getClass().getClassLoader(), target.getClass().getInterfaces());
		Constructor constructor = proxyClazz.getConstructor(InvocationHandler.class);
		Object proxy = constructor.newInstance(new InvocationHandler() {
			@Override
			public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
				System.out.println(method.getName() + "方法开始执行...");
				Object result = method.invoke(target, args);
				System.out.println(result);
				System.out.println(method.getName() + "方法执行结束...");
				return result;
			}
		});
		return proxy;
	}
}
```

![img](https://pic4.zhimg.com/80/v2-5e2bfa7494a2f7851df97816603fd0a7_720w.jpg)



厉害厉害...可惜，还是太麻烦了。有没有更简单的方式获取代理对象？有！

![img](https://pic1.zhimg.com/80/v2-beb7e4fd989ddea7f5eb6fe6fe882c84_720w.jpg)

直接返回代理对象，而不是代理对象Class

从一开始就存在，哈哈。但是我觉得getProxyClass()切入更好理解。

```java
public class ProxyTest {
	public static void main(String[] args) throws Throwable {
		CalculatorImpl target = new CalculatorImpl();
		Calculator calculatorProxy = (Calculator) getProxy(target);
		calculatorProxy.add(1, 2);
		calculatorProxy.subtract(2, 1);
	}

	private static Object getProxy(final Object target) throws Exception {
		Object proxy = Proxy.newProxyInstance(
				target.getClass().getClassLoader(),/*类加载器*/
				target.getClass().getInterfaces(),/*让代理对象和目标对象实现相同接口*/
				new InvocationHandler(){/*代理对象的方法最终都会被JVM导向它的invoke方法*/
					public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
						System.out.println(method.getName() + "方法开始执行...");
						Object result = method.invoke(target, args);
						System.out.println(result);
						System.out.println(method.getName() + "方法执行结束...");
						return result;
					}
				}
		);
		return proxy;
	}
}
```



## 编写可生成代理和可插入通知的通用方法

上面的代码，已经比上一篇开头直接修改目标类好多了。再来看一下当时的四大缺点：

1. 直接修改源程序，不符合开闭原则。应该对扩展开放，对修改关闭<font color=green>**√**</font>
2. 如果Calculator有几十个、上百个方法，修改量太大<font color=green>**√**</font>
3. 存在重复代码（都是在核心代码前后打印日志）<font color=red>**×**</font>
4. 日志打印硬编码在代理类中，不利于后期维护：比如你花了一上午终于写完了，组长告诉你这个功能取消，于是你又要打开Calculator花十分钟删除日志打印的代码！<font color=red>**×**</font>

使用动态代理，让我们避免手写代理类，只要给getProxy()方法传入target就可以生成对应的代理对象。但是日志打印仍是硬编码在invoke()方法中。虽然修改时只要改一处，但是别忘了“开闭原则”。所以最好是能把日志打印单独拆出来，像目标对象一样作为参数传入。

日志打印其实就是AOP里的通知概念。我打算定义一个Advice接口，并且写一个MyLogger实现该接口。

通知接口

```java
public interface Advice {
	void beforeMethod(Method method);
	void afterMethod(Method method);
}
```

日志打印

```java
public class MyLogger implements Advice {

	public void beforeMethod(Method method) {
		System.out.println(method.getName() + "方法执行开始...");
	}

	public void afterMethod(Method method) {
		System.out.println(method.getName() + "方法执行结束...");
	}
}
```

测试类

```java
public class ProxyTest {
	public static void main(String[] args) throws Throwable {
		CalculatorImpl target = new CalculatorImpl();
		Calculator calculatorProxy = (Calculator) getProxy(target, new MyLogger());
		calculatorProxy.add(1, 2);
		calculatorProxy.subtract(2, 1);
	}

	private static Object getProxy(final Object target, Advice logger) throws Exception {
		/*代理对象的方法最终都会被JVM导向它的invoke方法*/
		Object proxy = Proxy.newProxyInstance(
				target.getClass().getClassLoader(),/*类加载器*/
				target.getClass().getInterfaces(),/*让代理对象和目标对象实现相同接口*/
				(proxy1, method, args) -> {
					logger.beforeMethod(method);
					Object result = method.invoke(target, args);
					System.out.println(result);
					logger.afterMethod(method);
					return result;
				}
		);
		return proxy;
	}
}
```

差一点完美~下篇讲讲更完美的做法。



## 类加载器补充

初学者可能对诸如“字节码文件”、Class对象比较陌生。所以这里花一点点篇幅介绍一下类加载器的部分原理。如果我们要定义类加载器，需要继承ClassLoader类，并覆盖findClass()方法：

```java
@Override
public Class<?> findClass(String name) throws ClassNotFoundException {
	try {
		/*自己另外写一个getClassData()
                  通过IO流从指定位置读取xxx.class文件得到字节数组*/
		byte[] datas = getClassData(name);
		if(datas == null) {
			throw new ClassNotFoundException("类没有找到：" + name);
		}
		//调用类加载器本身的defineClass()方法，由字节码得到Class对象
		return this.defineClass(name, datas, 0, datas.length);
	} catch (IOException e) {
		e.printStackTrace();
		throw new ClassNotFoundException("类找不到：" + name);
	}
}
```

所以，这就是类加载之所以能把xxx.class文件加载进内存，并创建对应Class对象的深层原因。具体文章可以参考基友写的另一篇：[请叫我程序猿大人：好怕怕的类加载器](https://zhuanlan.zhihu.com/p/54693308)



------

## 小结

**静态代理**

代理类CalculatorProxy是我们事先写好的，编译后得到Proxy.class字节码文件。随后和目标类一起被ClassLoader（类加载器）加载进内存，生成Class对象，最后生成实例对象。代理对象中有目标对象的引用，调用同名方法并前后加上日志打印。

![img](https://pic2.zhimg.com/80/v2-f7f7a866c209a850f215689442fdd5a1_720w.jpg)

优点：不用修改目标类源码

缺点是：高度绑定，不通用。硬编码，不易于维护。



**动态代理**

我们本想通过接口Class直接创建代理实例，无奈的是，接口Class虽然有方法信息描述，却没有构造器，无法创建对象。所以我们希望JDK能提供一套API，我们传入接口Class，它自动复制里面的方法信息，造出一个有构造器、能创建实例的代理Class对象。

![img](https://pic4.zhimg.com/80/v2-7f7dbee80a373448e772bb372a3a037f_720w.jpg)

优点：

- 不用写代理类，根据目标对象直接生成代理对象
- 通知可以传入，不是硬编码

------

## 彩蛋

上面的讨论都在刻意回避代理对象的类型，放最后来聊一聊。

最后讨论一下代理对象是什么类型。

首先，请区分两个概念：代理Class对象和代理对象。

![img](https://pic2.zhimg.com/80/v2-bb82bd129d63f77265f51b2209159269_720w.jpg)

单从名字看，代理Class和Calculator的接口确实相去甚远，但是我们却能讲代理对象赋值给接口类型：

![img](https://pic3.zhimg.com/80/v2-e869e67fc4fbc708b793ff6ea6e2c012_720w.jpg)

但谁说能否复制给接口是看名字的？难道不是只要实现接口就行了吗？

> 代理对象的本质就是：和目标对象实现相同接口的实例。代理Class可以叫任何名字，whatever，只要它实现某个接口，就能成为该接口类型。

![img](https://pic4.zhimg.com/80/v2-91d716b1a95099ad364233de91fca7a3_720w.jpg)

我写了一个MyProxy类，那么它的Class名字必然叫MyProxy。**但这和能否赋值给接口没有任何关系。**由于它实现了Serializable和Collection，所以myProxy（代理实例）**同时**是这两个接口的类型。



我想了个很骚的比喻，希望能解释清楚：

接口Class对象是大内太监，里面的方法和字段比做他的一身武艺，但是他没有小DD（构造器），所以不能new实例。一身武艺后继无人。

那怎么办呢？



正常途径（implements）：

写一个类，实现该接口。这个就相当于大街上拉了一个人，认他做干爹。一身武艺传给他，只是比他干爹多了小DD，可以new实例。



非正常途径（动态代理）：

通过妙手圣医Proxy的克隆大法（Proxy.getProxyClass()），克隆一个Class，但是有小DD。所以这个克隆人Class可以创建实例，也就是代理对象。



代理Class其实就是附有构造器的接口Class，一样的类结构信息，却能创建实例。

![img](https://pic3.zhimg.com/80/v2-33094b28321ab388bb0db46608eae74a_720w.jpg)JDK动态代理生成的实例

![img](https://pic3.zhimg.com/80/v2-b99009ee292273a56ab483170b2e20aa_720w.jpg)CGLib动态代理生成的实例

如果说继承的父类是亲爹（只有一个），那么实现的接口是干爹（可以有多个）。

实现接口是一个类认干爹的过程。接口无法创建对象，但实现该接口的类可以。

比如

```text
class Student extends Person implements A, B
```

这个类new一个实例出来，你问它：你爸爸是谁啊？它会告诉你：我只有一个爸爸Person。

但是student instanceof A interface，或者student instanceof B interface，它会告诉你两个都是它干爹（true），都可以用来接收它。

![img](https://pic3.zhimg.com/80/v2-1c36d27a6a2a49a266a7fc2ed457e532_720w.jpg)

然而，凡是有利必有弊。

![img](https://pic3.zhimg.com/80/v2-991ea99b9038d52875ff6ba57e9032de_720w.jpg)

也就是说，动态代理生成的代理对象，最终都可以用接口接收，和目标对象一起形成了多态，可以随意切换展示不同的功能。但是切换的同时，只能使用该接口定义的方法。

下一篇是案例篇，写山寨Spring AOP事务：通过@MyTransactional随意切换普通Service对象和代理Service对象（含事务）。
