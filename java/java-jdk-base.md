---

layout: single
title: Java学习（JDK-java.lang包中的基础类）
permalink: /java/java-jdk-base.html

classes: wide

author: Bob Dong

---

# 前言

JDK，Java Development Kit。

我们必须先认识到，JDK只是，仅仅是一套Java基础类库而已，是Sun公司开发的基础类库，仅此而已，JDK本身和我们自行书写总结的类库，从技术含量来说，还是在一个层级上，它们都是需要被编译成字节码，在JRE中运行的，JDK编译后的结果就是jre/lib下的rt.jar，我们学习使用它的目的是加深对Java的理解，提高我们的Java编码水平。

本系列所有文章基于的JDK版本都是1.7.16。

这篇文章写于2014年，原文链接：[link](https://blog.csdn.net/puma_dong/article/details/39670411)。


在本节中，简析java.lang包所包含的基础类库，当我们新写一个class时，这个package里面的class都是被默认导入的，所以我们不用写import java.lang.Integer这样的代码，我们依然使用Integer这个类，当然，如果你显示写了import java.lang.Integer也没有问题，不过，何必多此一举呢。

# Object


默认所有的类都继承自Object，Object没有Property，只有Method，其方法大都是native方法（也就是用其他更高效语言，一般是c实现好了的）。

Object没有实现clone()，实现了hashCode()，哈希就是对象实例化后在堆内存的地址，用“==”比较两个对象，实际就是比较的内存地址是否是一个，也就是hashCode()是否相等。

默认情况下，对象的equals()方法和==符号是一样的，都是比较内存地址，但是有些对象重写了equals()方法，比如String，使其达到比较内容是否相同的效果。

另外两个方法wait()和notify()是和多线程编程相关的，多线程里面synchronized实际就是加锁，默认是用this当锁，当然也可以用任何对象当锁，wait()上锁，线程阻塞，notify()开锁，收到这个通知的线程运行。以下代码示例：

```
class Test implements Runnable
{
 
	private String name;
	private Object prev;
	private Object self;
 
	public Test(String name,Object prev,Object self)
	{
		this.name=name;
		this.prev = prev;
		this.self = self;
	}
 
	@Override
	public void run()
	{
		int count = 2;
		while(count>0)
		{
			synchronized(prev)
			{
				synchronized(self)
				{
					System.out.println(name+":"+count);
					count--;
					self.notify();	//通知self锁，开锁
				}
				try
				{
					prev.wait();	//prev锁，等待通知
				} catch(InterruptedException e)
				{
					e.printStackTrace();
				}
			}
		}
	}
 
	public static void main(String[] args) throws Exception
	{
		Object a = new Object();
		Object b = new Object();
		Object c = new Object();
		Test ta = new Test("A",c,a);
		Test tb = new Test("B",a,b);
		Test tc = new Test("C",b,c);
 
		new Thread(ta).start();	// 输出A:2，c锁wait，对a锁notify
		Thread.sleep(10);
		new Thread(tb).start();	// 输出B:2，a锁wait，对b锁notify
		Thread.sleep(10);
		new Thread(tc).start();	// 输出C:2，b锁wait，对c锁notify
		
		// c锁收到notify，向下执行，输出A:1，并对a锁notify
		// a锁收到notify，向下执行，输出B:1，并对b锁notify
		// b锁收到notify，向下执行，输出C:1，并对c锁notify
		// c锁继续wait，可以看到程序在等待notify中，并没有退出
	}
}
```

以上代码将顺序输出：A:2、B:2、C:2、A:1、B:1、C:1 。

Object类占用内存大小计算：<http://m.blog.csdn.net/blog/aaa1117a8w5s6d/8254922>

Java如何实现Swap功：<http://segmentfault.com/q/1010000000332606>

# 构造函数和内部类

构造函数不能继承，是默认调用的，如果不显示用super指明的话，默然是调用的父类中没有参数的构造函数。

```
class P {
	public P() {System.out.println("P");}
	public P(String name) {System.out.println("P" + name);}
}
class S extends P {
	public S(String name) {
		super("pname");	//	这里如果不指定的话，默认是调用的父类的没有参数的构造函数
		System.out.println("name");
	}
}
```

关于内部类，在应用编程中较少用到，但是JDK源码中大量使用，比如Map.Entry，ConcurrentHashMap，ReentrantLock等，enum本身也会被编译成static final修饰的内部类。

关于内部类的更多内容，可以参阅这篇文章：<http://android.blog.51cto.com/268543/384844/>

# Class和反射类


Java程序在运行时每个类都会对应一个Class对象，可以从Class对象中得到与类相关的信息，Class对象存储在方法区（又名Non-Heap，永久代），当我们运行Java程序时，如果加载的jar包非常多，大于指定的永久代内存大小时，则会报出PermGen错误，就是Class对象的总计大小，超过永久代内存的缘故。

Class类非常有用，在我们做类型转换时经常用到，比如以前用Thrift框架时，经常需要在Model类型的对象：Thrift对象和Java对象之间进行转换，需要手工书写大量模式化代码，于是，就写了个对象转换的工具，在Thrift对象和Java对象的Property名字一致的情况下，可以使用这个工具直接转换，其中大量使用了Class里面的方法和java.lang.reflect包的内容。

关于Calss类，方法众多，不详述。下面附上这个Thrift工具的代码。

```
import java.lang.reflect.Field;
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
 
/**
 * Thrift前缀的对象和不含Thrift前缀的对象相互转换.
 * 参考：
 * http://blog.csdn.net/it___ladeng/article/details/7026524
 * http://www.cnblogs.com/jqyp/archive/2012/03/29/2423112.html
 * http://www.cnblogs.com/bingoidea/archive/2009/06/21/1507889.html
 * http://java.ccidnet.com/art/3539/20070924/1222147_1.html
 * http://blog.csdn.net/justinavril/article/details/2873664
 */
public class ThriftUtil {
	public static final Integer THRIFT_PORT = 9177;
 
	/**
	 * Thrift生成的类的实例和项目原来的类的实例相关转换并赋值
	 * 1.类的属性的名字必须完全相同
	 * 2.当前支持的类型仅包括：byte,short,int,long,double,String,Date,List
	 * 3.如果有Specified列，则此列为true才赋值，否则，不为NULL就赋值
	 * @param sourceObject
	 * @param targetClass
	 * @param toThrift:true代表把JavaObject转换成ThriftObject，false代表把ThriftObject转换成JavaObject，ThriftObject中含有Specified列
	 * @return
	 */
	public static Object convert(Object sourceObject,Class<?> targetClass,Boolean toThrift)
	{
		if(sourceObject==null)
		{
			return null;
		}		
		//对于简单类型，不进行转换，直接返回
		if(sourceObject.getClass().getName().startsWith("java.lang"))
		{
			return sourceObject;
		}
		Class<?> sourceClass = sourceObject.getClass();
		Field[] sourceFields = sourceClass.getDeclaredFields();
		Object targetObject = null;
		try {
			targetObject = targetClass.newInstance();
		} catch (InstantiationException e1) {
			e1.printStackTrace();
		} catch (IllegalAccessException e1) {
			e1.printStackTrace();
		};
		if(targetObject==null)
		{
			return null;
		}
		for(Field sourceField:sourceFields)
		{
			try {
				//转换时过滤掉Thrift框架自动生成的对象
				if(sourceField.getType().getName().startsWith("org.apache.thrift")
						||sourceField.getName().substring(0,2).equals("__")
						||("schemes,metaDataMap,serialVersionUID".indexOf(sourceField.getName())!=-1)
						||(sourceField.getName().indexOf("_Fields")!=-1)
						||(sourceField.getName().indexOf("Specified")!=-1)
						)
				{
					continue;
				}
				
				//处理以DotNet敏感字符命名的属性，比如operator
				String sourceFieldName = sourceField.getName();
				if(sourceFieldName.equals("operator"))
				{
					sourceFieldName = "_operator";
				} else {
					if(sourceFieldName.equals("_operator"))
					{
						sourceFieldName = "operator";
					}
				}
				//找出目标对象中同名的属性
				Field targetField = targetClass.getDeclaredField(sourceFieldName);
				sourceField.setAccessible(true);
				targetField.setAccessible(true);
				String sourceFieldSimpleName = sourceField.getType().getSimpleName().toLowerCase().replace("integer", "int");
				String targetFieldSimpleName = targetField.getType().getSimpleName().toLowerCase().replace("integer", "int");
				//如果两个对象同名的属性的类型完全一致:Boolean,String,以及5种数字类型：byte,short,int,long,double，以及List
				if(targetFieldSimpleName.equals(sourceFieldSimpleName))
				{
					//对于简单类型，直接赋值
					if("boolean,string,byte,short,int,long,double".indexOf(sourceFieldSimpleName)!=-1)
					{
						Object o = sourceField.get(sourceObject);
						if(o != null)
						{
							targetField.set(targetObject, o);
							//处理Specified列，或者根据Specified列对数值对象赋NULL值
							try
							{
								if(toThrift)
								{
									Field targetSpecifiedField = targetClass.getDeclaredField(sourceFieldName+"Specified");
									if(targetSpecifiedField != null)
									{
										targetSpecifiedField.setAccessible(true);
										targetSpecifiedField.set(targetObject, true);
									}
								} else {
									Field sourceSpecifiedField = sourceClass.getDeclaredField(sourceFieldName+"Specified");
									if(sourceSpecifiedField != null 
											&& "B,S,B,I,L,D".indexOf(targetField.getType().getSimpleName().substring(0,1))!=-1
											)
									{
										sourceSpecifiedField.setAccessible(true);
										if(sourceSpecifiedField.getBoolean(sourceObject)==false)
										{
											targetField.set(targetObject, null);
										}
									}
								}
							} catch (NoSuchFieldException e) {
								//吃掉NoSuchFieldException，达到效果：如果Specified列不存在，则所有的列都赋值
							}
						}
						continue;
					}
					//对于List
					if(sourceFieldSimpleName.equals("list"))
					{
						@SuppressWarnings("unchecked")
						List<Object> sourceSubObjs = (ArrayList<Object>)sourceField.get(sourceObject);
						@SuppressWarnings("unchecked")
						List<Object> targetSubObjs = (ArrayList<Object>)targetField.get(targetObject);
						//关键的地方，如果是List类型，得到其Generic的类型 
						Type targetType = targetField.getGenericType();
						//如果是泛型参数的类型 
						if(targetType instanceof ParameterizedType) 
						{
							ParameterizedType pt = (ParameterizedType) targetType;
							//得到泛型里的class类型对象。  
							Class<?> c = (Class<?>)pt.getActualTypeArguments()[0]; 
							if(sourceSubObjs!=null)
							{
								if(targetSubObjs==null)
								{
									targetSubObjs = new ArrayList<Object>();
								}
								for(Object obj:sourceSubObjs)
								{
									targetSubObjs.add(convert(obj,c,toThrift));
								}
								targetField.set(targetObject, targetSubObjs);
							}							
						}
						continue;						
					}					
				}
				//转换成Thrift自动生成的类：Thrift没有日期类型，我们统一要求日期格式化成yyyy-MM-dd HH:mm:ss形式
				if(toThrift)
				{
					if(sourceFieldSimpleName.equals("date")&&targetFieldSimpleName.equals("string"))
					{
						Date d = (Date)sourceField.get(sourceObject);
						if(d!=null)
						{
							SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
							targetField.set(targetObject,sdf.format(d));							
						}
						continue;
					}
				} else {
					if(sourceFieldSimpleName.equals("string")&&targetFieldSimpleName.equals("date"))
					{
						String s = (String)sourceField.get(sourceObject);
						if(s!=null)
						{
							SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
							targetField.set(targetObject,sdf.parse(s));
						}
						continue;
					}
				}
				//对于其他自定义对象				
				targetField.set(targetObject, convert(sourceField.get(sourceObject),targetField.getType(),toThrift));
 
			} catch (SecurityException e) {
				e.printStackTrace();
			} catch (NoSuchFieldException e) {
				e.printStackTrace();
			} catch (IllegalArgumentException e) {
				e.printStackTrace();
			} catch (IllegalAccessException e) {
				e.printStackTrace();
			} catch (ParseException e) {
				e.printStackTrace();
			}
 
		}
		return targetObject;
	}
}
```

# ClassLoader


类装载器是用来把类(class)装载进JVM的，JVM规范定义了两种类型的类装载器：启动内装载器 (bootstrap) 和用户自定义装载器 (user-defined class loader) 。

JVM在运行时会产生三个ClassLoader：Bootstrap ClassLoader、Extension ClassLoader和App ClassLoader.。

Bootstrap是用C++编写的，我们在Java中看不到它，是JVM自带的类装载器，用来装载核心类库，如java.lang.*等。

AppClassLoader 的 Parent 是ExtClassLoader ，而 ExtClassLoader 的 Parent 为 Bootstrap ClassLoader 。 

Java 提供了抽象类 ClassLoader ，所有用户自定义类装载器都实例化自 ClassLoader 的子类。 

System Class Loader(或AppClassLoader) 是一个特殊的用户自定义类装载器，由 JVM 的实现者提供，在编程者不特别指定装载器的情况下默认装载用户类 ，System Class Loader 可以通过ClassLoader.getSystemClassLoader() 方法得到。

代码演示如下：

```
class ClassLoaderTest {
	public static void main(String[] args) throws Exception{
		Class c;
		ClassLoader cl;
		cl = ClassLoader.getSystemClassLoader();
		System.out.println(cl);
		while(cl != null) {
			cl = cl.getParent();
			System.out.println(cl);
		}
		c = Class.forName("java.lang.Object");
		cl = c.getClassLoader();
		System.out.println("java.lang.Object's loader is " + cl);
		c = Class.forName("ClassLoaderTest");
		cl = c.getClassLoader();
		System.out.println("ClassLoaderTest's loader is " + cl);
	}
}
```

关于类加载器的更多参考：<https://www.cnblogs.com/theRhyme/p/9469040.html>。

# 八种基本数据类型

| 类类型    | 原生类型(primitive) | 说明                                                     |
| --------- | ------------------- | -------------------------------------------------------- |
| Boolean   | boolean             | 布尔                                                     |
| Character | char                | 单个Unicode字符，占用两个字节，例如'a','中'，范围0-65535 |
| Byte      | byte                | 8位有符号整型                                            |
| Short     | short               | 16位有符号整型                                           |
| Integer   | int                 | 32位有符号整型                                           |
| Long      | long                | 64位有符号整型                                           |
| Float     | float               | 单精度浮点数                                             |
| Double    | double              | 双精度浮点数                                             |

了解到数据类型，就要了解一点编码（数字在计算机中的01编码）的知识，计算机是用01表示所有类型的，第一位是符号位，0+1-，java中的整型都是有符号的。

对于数字，定义了原码、反码和补码。

正数的原码、反码和补码都是一样的，负数的反码是除符号位之外，所有位取反，补码是反码+1，以Byte类型为例：

能表示的数字范围是-128~127，总计256个数，对于0~127，表示为：0000000~01111111，即0~2^7-1，对于负数，-128~-1，表示如下：

-127~-1，其原码分别为11111111~10000001，换算成补码是10000001~11111111，然后10000000代表-128，-128只有补码，没有原码和反码。

补码的设计目的：

使符号位能与有效值部分一起参加运算，从而简化运算规则。补码机器数中的符号位，并不是强加上去的，是数据本身的自然组成部分，可以正常地参与运算。

使减法运算转换为加法运算，进一步简化计算机中运算器的线路设计。

反码是原码与补码转换的一个中间过渡，使用较少。

所有这些转换都是在计算机的最底层进行的，而在我们使用的汇编、c等其他高级语言中使用的都是原码。

除此之外，JDK原码还经常使用`>>、>>>`运算符，`>>`是有符号移位，高位补符号位，右移；`>>>`是无符号移位，高位补0，右移。

# 关于数据类型的内存存储

int

原生类型，比如int i = 3，不论是变量，还是值，都是存储在栈中的；

类类型，比如Integer i = 300，则变量是存储在栈中，对象是存储在堆中的一个对象，既然Integer是这样，那么Integer a = 300，Integer b = 300，请问 a == b，是否成立？

答案是：不成立的，因为两个对象的内存地址不同了；

既然这样，那么Integer a = 1，Integer b = 1，请问a == b，是否成立？

答案是：成立的，这其实是Java的一个坑，我们可以看Integer对于int进行封箱操作的源码，如下：

```
public static Integer valueOf(int i) {
	if(i >= -128 && i<= IntegerCache.high)
		return IntegerCache.cache[i+128];
	else
		return new Integer(i);
}
```

由此可见，对于128之内的数字，是进行了Cache存储的 ，所以在堆中的内存地址是一样的，所以成立。



在编程中，可以对于数字前面加0x表示16进制数，加0表示8进制数，默认是10进制数；在数字的后面增加L表示long类型，默认是整型；对于浮点数，在后面增加F表示float类型，模式是double型。



对于类型转换，比如int i=(int)1.5，会直接把小数后面的部分去掉，对于数学函数Math.round()、Math.ceil()、Math.floor()，可以把数字想象成0为水平线，负数在水平线下，正数在水平线上，ceil是转换成天花板，floor是转换成地板，round()是四舍五入，等于+0.5之后求floor。



String


String a = "abc"，String b = "abc"，a == b是成立的，这是String的编译时优化，对于字符串常量，在内存中只存一份（JDK6是存储在“方法区”的“运行时常量区”，JDK7是存储在堆中）；

我们对于String进一步分析：

String a = "a" + "b" + "c"，String b = "abc"，a == b依然是成立的，原理同上；

String t = "a"，String a = t + "b" + "c"，String b = "abc"，则a == b是不成立的，因为对于变量，编译器无法进行编译时优化；但是a.intern() == b是成立的，因为当调用intern()方法时，是到运行时常量池中找值相对的对象，找到了b，所以a.intern() == b。

final String t = "a"，String a = t + "b" + "c"，String b = "abc"，则a == b是成立的，因为final是不可以重新赋值的，编译器可以进行编译时优化；

对于String s = a + b这样的操作，每次操作，都会在堆中开辟一块内存空间，所以对于频繁大量的字符操作，性能低，所以对于大量字符操作，推荐使用的StringBuffer和StringBuilder，其中StringBuffer是线程安全的，StringBuilder是1.5版本新加的，是非线程安全的。



实现36进制加法，及字符串替换代码演示

```
public class RadixAndReplace {
    public static void main(String[] args) {
        //1.36进制加法（不允许使用字符串整体转换为10进制再相加的办法，所以只能按位加）
        System.out.println(add("ZZ","ZZZ"));
        System.out.println(add("zz","zzz"));
        System.out.println(add("3A","2"));
        //2.替换字符串（考虑替换之后符合要求的情况，只用一次循环）
        System.out.println(replace("acbd"));
        System.out.println(replace("abcd"));
    }
    //36进制加法
    public static String add(String s1,String s2) {
        int l1 = s1.length(),l2 = s2.length();
        int maxLength = Math.max(l1, l2);
        int carrybit = 0;	//进位
        StringBuilder sb = new StringBuilder();
        for(int i = 0 ; i < maxLength ; i++) {
            int r1 = i < l1 ? convertToNumber(s1.charAt(s1.length()-1-i)) : 0;
            int r2 = i < l2 ? convertToNumber(s2.charAt(s2.length()-1-i)) : 0;
            String r = convertToString(r1 + r2 + carrybit);
            carrybit = r.length() == 2 ? 1 : 0;
            sb.append(r.charAt(r.length()-1));
        }
        if(carrybit == 1) {
            sb.append("1");
        }
        return sb.reverse().toString();
    }
    //把字符（A-Z代表10-36，a-z代表37-72）
    private static int convertToNumber(char ch) {
        int num = 0;
        if(ch >= 'A' && ch <= 'Z')
            num = ch - 'A' + 10;
        else if(ch >= 'a' && ch <= 'z')
            num = ch - 'a' + 10;
        else
            num = ch - '0';
        return num;
    }
    //转换数字为36进制的字符串表示
    //A的ASCII码是65，a的ASCII码是97
    private static String convertToString(int n) {
        if(n >= 0 && n <= 9) {
            return String.valueOf(n);
        }
        if(n >= 10 && n <= 35) {
            return String.valueOf((char)(n-10+65));
        }
        if(n >= 36 && n <= 71) {
            return "1" + convertToString(n-36);
        }
        return "0";
    }
    //替换字符串“ac”，“b”，考虑b在ac中间的情况
    public static String replace(String str) {
        StringBuilder sb = new StringBuilder();
        Boolean flag = false;	//连续判断标志位
        for(int i = 0; i < str.length(); i++) {
            char c = str.charAt(i);
            if(c == 'b') continue;
            if(c == 'a') {
                flag = true;
                continue;
            }
            if(c == 'c' && flag) {
                flag = false;
                continue;
            }
            if(flag) {
                sb.append('a').append(c);
            } else {
                sb.append(c);
            }
        }
        return sb.toString();
    }
}
```

# 异常和错误


Exception和Error都继承自Throwable对象，Exception分为两类，RuntimeException（运行时异常，unchecked）和一般异常（checked），Error是比较严重的错误，也是unchecked。

简单地讲，checked是必须用try/catch 或者throw处理的，可以在执行过程中恢复的，比如java.io.FileNotFoundException；而unchecked异常则是不需要try/catch处理的，比如java.lang.NullPointerException。

在比较流行的语言中，Java是唯一支持checked异常，要求我们必须进行处理的语言，这有利有弊。

关于异常类、错误类，在JDK中定义了很多，用来描述我们的代码可能遇到的各种类型的错误。

**UnsupportedOperationException**


举个例子，如下：

在其他语言（比如C#），如果一个类中，某些方法还没有完成，但是需要提供一个名字给别人调用，我们可以先对这个方法加行代码throw new Exception("xxx")；

刚用Java时，我们可能也想只是给一行代码throw new Exception，这是不合理的，因为：对于非RuntimeException，必须try/catch，或者在方法名后面增加throws Exception（这导致调用这个方法的地方都要try/catch，或者throws，是不合理的）；所以，对于这个功能，我们正确的做法是：throws new UnsupportedOperationException("xxx")，这个是运行时异常，unchecked。

# Runtime

java.lang包里有很多运行时环境相关的类，可以查看运行时环境的各种信息，比如内存、锁、安全、垃圾回收等等。

比如如下钩子代码，在JVM关闭时，执行一些不好在程序计算过程中进行的资源释放工作，如下：

```
public class MongoHook {
 
	static void addCloseHook(){
		Runtime.getRuntime().addShutdownHook( new Thread(){			
			@Override
			public void run() {
				MongoDBConn.destoryAllForHook() ;
			}			
		}) ;
	}
}
```

关于这个退出钩子，实际是java.lang包种的以下3个类配合完成的：Runtime、Shutdown、ApplicationShutdownHooks。

# 多线程


Thread、Runnable、ThreadLocal等，关于多线程的更多知识，可以参阅[多线程进阶](/java/multithread-definitive.html)。

# 接口


Clonable：声明式接口

Comparable：对象比较，泛型类

Appendable：

CharSequence：统一的字符串只读接口，共有4个方法，length()、charAt()、subSequence()、toString()。

Readable：

Iterable：迭代器

# 注解类


主要在java.lang.annotation包中，注解类用@interface来进行定义，注解类的作用范围可以是方法、属性、包等，作用时效可以是Source、Runtime、Class。

比如Override就是一个注解类，用来标记实现接口中定义的类，其源码如下；

```
package java.lang;
import java.lang.annotation.*;
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```

注解本身不做任何事情，只是像xml文件一样起到配置作用。注解代表的是某种业务意义。

spring中@Resource注解简单解析：首先解析类的所有属性，判断属性上面是否存在这个注解，如果存在这个注解，再根据搜索规则来取得这个bean，然后通过反射注入。

**注解有如下规则：**

1）所有的注解类都隐式继承于 java.lang.annotation.Annotation，注解不允许显式继承于其他的接口。

2）注解不能直接干扰程序代码的运行，无论增加或删除注解，代码都能够正常运行。Java语言解释器会忽略这些注解，而由第三方工具负责对注解进行处理。

3）注解的成员以无入参、无抛出异常的方式声明；可以通过default为成员指定一个默认值；成员类型是受限的，合法的类型包括primitive及其封装类、String、Class、enums、注解类型，以及上述类型的数组类型；注解类可以没有成员，没有成员的注解称为标识注解，解释程序以标识注解存在与否进行相应的处理。



代码示例1：

```
package com.cl.search.utils;
 
import java.lang.annotation.*;
import java.lang.reflect.*;
 
public class MyAnnotationTest {
	public static void main(String[] args) {
		Test test = Container.getBean();
		test.loginTest();
	}
}
 
@Documented
@Target({ElementType.METHOD, ElementType.TYPE, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@interface AnnotationTest {    
    public String nation() default "";
}
 
interface IUser {
    public void login();
}
 
class ChineseUserImpl implements IUser {
    @Override
    public void login() {
        System.err.println("用户登录！");
    }
}
 
class EnglishUserImpl implements IUser {
    @Override
    public void login() {
        System.err.println("User Login！");
    }
}
 
@AnnotationTest
class Test {
    private IUser userdao;
    public IUser getUserdao() {
        return userdao;
    }
    @AnnotationTest(nation = "EnglishUserImpl")
    public void setUserdao(IUser userdao) {
        this.userdao = userdao;
    }
    public void loginTest() {
        userdao.login();
    }
}
 
class Container {
    public static Test getBean() {
        Test test = new Test();
        if (Test.class.isAnnotationPresent(AnnotationTest.class)) {
            Method[] methods = Test.class.getDeclaredMethods();
            for (Method method : methods) {
                System.out.println(method);
                if (method.isAnnotationPresent(AnnotationTest.class)) {
                    AnnotationTest annotest = method.getAnnotation(AnnotationTest.class);
                    System.out.println("AnnotationTest(field=" + method.getName()
                            + ",nation=" + annotest.nation() + ")");
                    IUser userdao;
                    try {
                        userdao = (IUser) Class.forName("com.cl.search.utils." + annotest.nation()).newInstance();
                        test.setUserdao(userdao);
                    } catch (Exception ex) {
                        ex.printStackTrace();
                    }
                }
            }
        } else {
            System.out.println("没有注解标记！");
        }
        return test;
    }
}
```

# java.lang.ref包


java.lang.ref 是 Java 类库中比较特殊的一个包，它提供了与 Java 垃圾回收器密切相关的引用类。

# java.lang.AutoCloseable接口


在Java7中，引入了一个新特性try-with-resource，即在try中的代码，其资源会自动释放，不用手工执行资源释放操作。

要求跟在try后面的资源必须实现AutoCloseable接口，否则会报变异错误，代码示例如下：

```
class CustomResource implements AutoCloseable
{
	@Override
	public void close() throws Exception {
		System.out.println("进行资源释放操作！");
	}
	
	public static void main(String[] args) throws Exception {
		try(CustomResource r = new CustomResource()) {
			System.out.println("使用资源！");
		}
	}
}
```
