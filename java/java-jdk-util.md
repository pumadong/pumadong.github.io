---

layout: single
title: Java学习（JDK-java.util包中的工具类）
permalink: /java/java-jdk-util.html

classes: wide

author: Bob Dong

---

# 前言

JDK，Java Development Kit。

我们必须先认识到，JDK只是，仅仅是一套Java基础类库而已，是Sun公司开发的基础类库，仅此而已，JDK本身和我们自行书写总结的类库，从技术含量来说，还是在一个层级上，它们都是需要被编译成字节码，在JRE中运行的，JDK编译后的结果就是jre/lib下的rt.jar，我们学习使用它的目的是加深对Java的理解，提高我们的Java编码水平。

本系列所有文章基于的JDK版本都是1.7.16。

源码下载地址：https://jdk7.java.net/source.html



本节内容


在本节中，简析java.util包所包含的工具类库，主要是集合相关的类库，其次还有正则、压缩解压、并发、日期时间等工具类。

本篇内容大致、简单的对于java.util包进行了一个描述，以后会逐渐进行内容补充，本篇文章相当于一个占位符，所谓先有了骨架，才能逐渐丰满



集合类


基本情况


主要接口及其继承关系如下：

SortedSet  -->  Set --> Collection -->  Iterable

List  -->  Collection  -->  Iterable

SortedMap  -->  Map

常用类及其继承关系如下：

HashSet/LinkedHashSet  --> Set

TreeSet  -->  SortedSet  --> Set

ArrayList/LinkedList  -->  List

HashMap  -->  Map

TreeMap  -->  SortedMap  -->  Map



统一称谓：Collection分支的，我们称之为“聚集”；Map分支的，我们称之为“映射”。

Collection继承自Iterable，所以其下的类都可以用迭代器Iterator访问，也可以用for(E e:es)形式访问；Map可以用实现了其内部接口Entry的对象，作为一个元素。

Hashtable和HashMap，他们都实现了Map接口；Hashtable继承自古老的抽象类Dictionary，是线程安全的；HashMap继承自较新的抽象类AbstractMap，不是线程安全的。

HashMap允许null的键和值，而Hashtable不允许null的键和值，这是因为：

Hashtable有方法contains方法（判断是否存在值），如果允许的话，则不论key或者value为null，都会返回null，这容易误解，所以Hashtable就强制限制了，对于null 键和值，直接抛出NullPointerException；

HashMap没有contains方法，分别是containsKey()和containsValues()。

另外JDK5开始，对于线程安全的Map，有一种ConcurrentHashMap，高效，其实现线程安全的过程中，没有使用synchronized，是一种分段的结构，并用CAS这种无锁算法实现了线程安全。



Hash


Object类有两种方法来推断对象的标识：equals()和hashCode()。

一般来说，如果您忽略了其中一种，您必须同时忽略这两种，因为两者之间有必须维持的至关重要的关系。

特殊情况是根据equals() 方法，如果两个对象是相等的，它们必须有相同的hashCode()值，Object源码中对此有要求，尽管这通常不是真的。

http://blog.sina.com.cn/s/blog_5dc351100101l57b.html

http://fhuan123.iteye.com/blog/1452275

关于HashMap的源码分析，可以参考：http://github.thinkingbar.com/hashmap-analysis/

LinkedHashMap，重写了HashMap的迭代器、AddEntry、Entry等几个方法和类，用一个双向链表存储元素加入的顺序；这可以按照访问顺序排序，最近访问的元素（get/put），会被放在链表的末尾，这是LRU算法（Least Recenty Used），最近最少使用算法。



ArrayList和LinkedList

关于ArrayList和LinkedList，ArrayList是基于数组的，这种方式将对象放在连续的位置中，读取快，但是容量不足时需要进行数组扩容，性能降低，插入和删除也慢；LinkedList是基于链表的，插入和删除都快，但是查找麻烦，不能按照索引查找。所以说，对于构造一个队列是用ArrayList或者LinkedList，是根据性能和方便来考虑的，比如LinkedList有removeLast()，ArrayList只能remove(index)，用LinkedList构造一个Queue的代码演示如下：

```

class Queue {
	private LinkedList<String> llt;
	public Queue() {
		 llt = new LinkedList<String>();
	}
	public void add(String s) {
		llt.add(s);
	}
	public String get() {
		return llt.removeLast();	//队列
		//return llt.removeFirst();	//堆栈
	}
	public boolean isNull() {
		return llt.isEmpty();
	}
}
```

### ConcurrentModificationException

```
import java.util.*;
import java.util.Map.Entry;
class Test
{
	public static void main(String[] args) throws Exception {
		HashMap<String,Integer> mTemp = new HashMap<String,Integer>();
		mTemp.put("test1",1);		
		Iterator<Entry<String,Integer>> iTemp = mTemp.entrySet().iterator();
		//以下这行代码会引发java.util.ConcurrentModificationException，
		//因为对聚集创建迭代器之后，进行遍历或者修改操作时，如果遇到期望的修改计数器和实际的修改计数器不一样的情况（modCount != expectedModCount）
		//就会报这个Exception，乐观锁的思想
		//mTemp.put("test2",2);	
		while(iTemp.hasNext()) {
			System.out.println(iTemp.next().getKey());
		}
 
		//for循环,写法更简单一些，在编译后，还是会被转换为迭代器
		for(Entry<String,Integer> e : mTemp.entrySet()) {
			System.out.println(e.getKey());
		}
 
		ArrayList<string> al = new ArrayList<string>();
		al.add("test");
		for(String s : al) {
			Integer i = Integer.reverse((new java.util.Random().nextInt(100)));
			al.add(i.toString());	//这行代码也会报ConcurrentModificationException		
		}
	}
}
```

对于这种情况，可以使用java.util.concurrent包中的相关类，比如CopyOnWriteArrayList，就不会报这个异常了，因为CopyOnWriteArrayList类最大的特点就是，在对其实例进行修改操作（add/remove等）会新建一个数据并修改，修改完毕之后，再将原来的引用指向新的数组。这样，修改过程没有修改原来的数组，也就没有了ConcurrentModificationException错误。

我们可以参考CopyOnWriteArrayList的源码：

```
    /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```

ConcurrentModificationException，表明：我正读取的内容被修改掉了，你是否需要重新遍历？或是做其它处理？这就是fast-fail（快速失败机制）的含义。

虽然这只是在一个线程之内，并不是多线程的，我们同样也可以这样理解fast-fail：

Fail-fast是并发中乐观(optimistic)策略的具体应用，它允许线程自由竞争，但在出现冲突的情况下假设你能应对，即你能判断出问题何在，并且给出解决办法；

悲观(pessimistic)策略就正好相反，它总是预先设置足够的限制，通常是采用锁(lock)，来保证程序进行过程中的无错，付出的代价是其它线程的等待开销。

快速失败机制主要目的在于使iterator遍历数组的线程能及时发现其他线程对Map的修改（如put、remove、clear等），因 此，fast-fail并不能保证所有情况下的多线程并发错误，只能保护iterator遍历过程中的iterator.next()与写并发.



TreeSet和Collections.sort


TreeSet是基于TreeMap的实现，底层数据结构是“红黑树”，数据加入时已经排好顺序，存取及查找性能不如HashSet；Collections.sort是先把List转换成数组，再利用“归并排序”算法进行排序，归并排序是一种稳定排序。

关于TreeMap的文章：

http://zh.wikipedia.org/wiki/%E7%BA%A2%E9%BB%91%E6%A0%91

http://www.cnblogs.com/fornever/archive/2011/12/02/2270692.html

http://shmilyaw-hotmail-com.iteye.com/blog/1836431

http://www.ibm.com/developerworks/cn/java/j-lo-tree/index.html

http://blog.csdn.net/chenhuajie123/article/details/11951777

对这两种排序算法的性能比较如下（24核、64G内存，RHEL6.2）：

在数据已经基本排好顺序的情况下，排序元素数目，在某个段内（大约是2万-20万），TreeSet更高效；其他数目下Collections.sort更高效；

在数据随机性较强的情况下，排序元素数目，在1万之内，相差不大，Collections.sort性能略高；在1万之外，80万之内，TreeSet性能明显高于Collections.sort；80万之外，Collection.sort性能更高；java.util.concurrent.ConcurrentSkipListSet这种基于“跳表”的线程安全的可排序类，在30万之内，性能高于Collection.sort，30万之外，性能低于Collection.sort，ConcurrentSkipListSet的排序性能总是低于TreeSet。

ConcurrentSkipListSet有一个平衡的树形索引机构没有的好处，就是在并发环境下其表现很好。

这里可以想象，在没有了解SkipList这种数据结构之前，如果要在并发环境下构造基于排序的索引结构，那么也就红黑树是一种比较好的选择了，但是它的平衡操作要求对整个树形结构的锁定，因此在并发环境下性能和伸缩性并不好。

代码演示如下：

```
import java.util.TreeSet;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.Collections;
import java.util.Arrays;
import java.util.ListIterator;
import java.util.Random;
import java.util.Iterator;
 
class Test {
        public static void main(String[] args) {
 
                final int LEN = 300000;
 
                final int SEED = 100000;
                Random r = new Random();
 
                System.out.println("---------------------------");
 
                long b = System.currentTimeMillis();
                TreeSet<Temp> ts = new TreeSet<Temp>(new Comparator<Temp>(){
                        public int compare(Temp t1,Temp t2) {return t1.id-t2.id;}
                });
                for(int i=0;i<LEN;i++) {
                        ts.add(new Temp(r.nextInt(SEED)));
                }
 
                System.out.println(System.currentTimeMillis() - b);
 
                ArrayList<Temp> aTemp = new ArrayList<Temp>();
                Iterator<Temp> it = ts.iterator();
                while(it.hasNext()) {
                        aTemp.add(it.next());
                }
 
                System.out.println(System.currentTimeMillis() - b);
 
                System.out.println("---------------------------");
 
                b = System.currentTimeMillis();
                ArrayList<Temp> al = new ArrayList<Temp>();
                for(int i=0;i<LEN;i++) {
                        al.add(new Temp(r.nextInt(SEED)));
                }
                //split to the real excution unit
                /*
                Collections.sort(al,new Comparator<Temp>() {
                        public int compare(Temp t1,Temp t2) {return t1.id-t2.id;}
                });*/
                Temp[] a = new Temp[al.size()];
                al.toArray(a);
                System.out.println(System.currentTimeMillis() - b);
                Arrays.sort(a,new Comparator<Temp>() {
                        public int compare(Temp t1,Temp t2) {return t1.id-t2.id;}
                });
                System.out.println(System.currentTimeMillis() - b);
                ListIterator<Temp> li = al.listIterator();
                for(int i=0;i<a.length;i++) {
                        li.next();
                        li.set(a[i]);
                }
                System.out.println(System.currentTimeMillis() - b);
        }
}
 
class Temp {
        public Temp(int id) {this.id = id;}
        public int id;
}
```

一个错误验证：

增减进行过一个错误验证，发现对一个对象使用TreeSet排序，和使用同样数据Entry<String,Double>进行排序比较，性能很差。开始以为JDK对Entry做过优化，static/final之类，后来把对象也改成final，里面元素也改成final，发现性能依旧很差，完全不能解释，感觉无法理解。

后来，发现是两段代码不一致，使用Entry进行排序的代码有bug，导致排序的数据很少，所以显得性能好。。。。

所以，无端的臆测还是不要的，建立在JDK深入理解的基础上就好。



另外一个排序思路

比如，取出Top 20，也不一定要全部排序，可以只取前20个，经验证，小数据量时，性能也是非常高，大数据未验证。代码大致如下：

```
int n = 0;
double minScore = 100;	//Top20中最小的积分
String minKey = "";		//最小值所在的Key
Map<String,Double> skuTop = new HashMap<String,Double>();
Set<String> styles = new HashSet<String>();	//过滤同款
 
for(String sku :tempSkuViewSkus.get(goodsUser.getKey())) {
	boolean filter = false;
	filter = filterSameStyle(sku,styles);
	if(filter) continue;
	
	//过滤不成功，直接continue
	Set<String> userSet = goodsUserView.get(sku);
	if(userSet == null || userSet.size() == 0) continue;
	//这一步，积分的计算，是最耗时的操作（性能瓶颈所在）
	double score = mathTools.getJaccardSimilar(goodsUser.getValue(), userSet);
	//前20个直接进入Map
	if(n++ < ConstMongo.maxRecomNum) {
		skuTop.put(sku, score);
		if(score < minScore) {
			minScore = score;
			minKey = sku;
		}
		continue;
	}
	if(score <= minScore) continue;
	//替换最小值
	skuTop.remove(minKey);
	skuTop.put(sku, score);
	minScore = score;
	minKey = sku;
	for(Entry<String,Double> e : skuTop.entrySet()) {
		if(e.getValue() < minScore) {
			minScore = e.getValue();
			minKey = e.getKey();
		}
	}
}
```

### 正则表达式

```
import java.util.regex.Matcher;
import java.util.regex.Pattern;
 
public class Hello {
 
	public static void main(String[] args)
	{
		Pattern pattern = Pattern.compile("正则表达式");
		//Pattern pattern = Pattern.compile("Hello,正则表达式\\s[\\S]+");
		Matcher matcher = pattern.matcher("正则表达式 Hello,正则表达式 World");
		//替换第一个符合正则的数据
		System.out.println(matcher.replaceFirst("Java"));
	}
}
```

常用的开发语言都支持正则表达式，但是其对于正则支持的程度是不一样的。
Js正则：http://msdn.microsoft.com/zh-cn/library/ae5bf541(VS.80).aspx

Python正则：http://www.cnblogs.com/huxi/archive/2010/07/04/1771073.html

Java正则：

http://www.blogjava.net/xzclog/archive/2006/09/19/70603.html

http://www.cnblogs.com/android-html5/archive/2012/06/02/2533924.html


并发相关类


如下章节的内容有简单使用演示：http://blog.csdn.net/puma_dong/article/details/37597261#t5



压缩解压类


如下章节的内容有简单使用演示：http://blog.csdn.net/puma_dong/article/details/23018555#t20



其他工具类


定时器、日期、时间、货币等


# 后记

