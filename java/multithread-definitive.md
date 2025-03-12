---

layout: single
title: Java学习（多线程进阶）
permalink: /java/multithread-definitive.html

classes: wide

author: Bob Dong

---

# 前言

先说点别的，为什么要逐渐学会读英文书籍


解释一个名词：上下文切换、Context switch



多任务系统中，上下文切换是指CPU的控制权由运行任务转移到另外一个就绪任务时所发生的事件。

When one thread’s execution is suspended and swapped off the processor, and another thread is swapped onto the processor and its execution is resumed, this is called a context switch.


为什么读英文版的计算机书籍，能理解的透彻和深刻。我们使用的语言，比如Java，是用英语开发的，其JDK中的类的命名、方法的命名，都是英文的，所以，当用英文解释一个名词、场景时，基于对这门编程语言的了解，我们马上就理解了，是非常形象的，简直就是图解，而用中文解释，虽然是我们的母语，但是对于计算机语言而言，却是外语，却是抽象的，反而不容易理解。

推荐书籍：Java Thread Programming ，但是要注意，这本书挺古老的，JDK1.1、1.2时代的产物，所以书中的思想OK，有些代码例子，可能得不出想演示的结果。



前言


关于多线程的知识，有非常多的资料可以参考。这里稍微总结一下，以求加深记忆。

关于多线程在日常工作中的使用：对于大多数的日常应用系统，比如各种管理系统，可能根本不需要深入了解，仅仅知道Thread/Runnable就够了；如果是需要很多计算任务的系统，比如推荐系统中各种中间数据的计算，对多线程的使用就较为频繁，也需要进行一下稍微深入的研究。



几篇实战分析线程问题的好文章：

怎样分析 JAVA 的 Thread Dumps

各种 Java Thread State 第一分析法则

数据库死锁及解决死锁问题

全面解决五大数据库死锁问题

关于线程池的几篇文章：

http://blog.csdn.net/wangpeng047/article/details/7748457

http://www.oschina.net/question/12_11255

http://jamie-wang.iteye.com/blog/1554927



基本知识


JVM最多支持多少个线程：http://www.importnew.com/10780.html

在一次真实的案例中，8G内存（Java应用分配了4G堆内存），4核，虚拟机，JVM 1.7，在应用down掉之前，大约开了12000个线程：http://blog.csdn.net/puma_dong/article/details/46669499



线程安全


如果你的代码所在的进程中有多个线程在同时运行，而这些线程可能会同时运行这段代码。

如果每次运行结果和单线程运行的结果是一样的，而且其他的变量的值也和预期的是一样的，就是线程安全的。

或者这样说：一个类或者程序所提供的接口对于线程来说是原子操作或者多个线程之间的切换不会导致该接口的执行结果存在二义性，也就是说我们不用考虑同步的问题。



或者我们这样来简单理解，同一段程序块，从某一个时间点同时操作某个数据，对于这个数据来说，分叉了，则这就不是线程安全；如果对这段数据保护起来，保证顺序执行，则就是线程安全。



原子操作


原子操作（atomic operation）：是指不会被线程调度机制打断的操作；这种操作一旦开始，就一直运行到结束，中间不会有任何 context switch （切换到另一个线程）。

原子操作（atomic operation）：如果一个操作所处的层（layer）的更高层不能发现其内部实现与结构，则这个操作就是原子的。

原子操作可以是一个步骤，也可以是多个操作步骤，但是其顺序是不可以被打乱，或者切割掉只执行部分。

在多进程（线程）访问资源时，原子操作能够确保所有其他的进程（线程）都不在同一时间内访问相同的资源。

原子操作时不需要synchronized，这是Java多线程编程的老生常谈，但是，这是真的吗？我们通过测试发现（return i），当对象处于不稳定状态时，仍旧很有可能使用原子操作来访问他们，所以，对于java中的多线程，要遵循两个原则：

a、Brian Goetz的同步规则，如果你正在写一个变量，它可能接下来将被另一个线程读取，或者正在读取一个上一次已经被另一个线程写过的变量，那么你必须使用同步，并且，读写线程都必须用相同的监视器锁同步；

b、Brain Goetz测试：如果你可以编写用于现代微处理器的高性能JVM，那么就有资格去考虑是否可以避免使用同步

通常所说的原子操作包括对非long和double型的primitive进行赋值，以及返回这两者之外的primitive。之所以要把它们排除在外是因为它们都比较大，而JVM的设计规范又没有要求读操作和赋值操作必须是原子操作（JVM可以试着去这么作，但并不保证）。



错误理解

```
import java.util.Hashtable;
class Test
{
	public static void main(String[] args) throws Exception {
		final Hashtable<String,Integer> h = new Hashtable<String,Integer>();
		long l1 = System.currentTimeMillis();
		for(int i=0;i<10000;i++) {
			new Thread(new Runnable(){
				@Override
				public void run() {
					h.put("test",1);
					Integer i1 = h.get("test");
					h.put("test",2);
					Integer i2 = h.get("test");
					if(i1 == i2) {
						System.out.println(i1 + ":" + i2);
					}
				}
			}).start();
		}
		long l2 = System.currentTimeMillis();
 
		System.out.println((l2-l1)/1000);
	}
}
```

有人觉得：既然Hashtable是线程安全的，那么以上代码的run()方法中的代码应该是线程安全的，这是错误理解。
线程安全的对象，指的是其内部操作（内部方法）是线程安全的，外部对其进行的操作，如果是一个序列，还是需要自己来保证其同步的，针对以上代码，在run方法的内部使用synchronized就可以了。



互斥锁和自旋锁


自旋锁：不睡觉，循环等待获取锁的方式成为自旋锁，在ConcurrentHashMap的实现中使用了自旋锁，一般自旋锁实现会有一个参数限定最多持续尝试次数，超出后，自旋锁放弃当前time slice，等下一次机会，自旋锁比较适用于锁使用者保持锁时间比较短的情况。

正是由于自旋锁使用者一般保持锁时间非常短，因此选择自旋而不是睡眠是非常必要的，自旋锁的效率远高于互斥锁。

CAS乐观锁适用的场景：http://www.tuicool.com/articles/zuui6z



ThreadLocal与synchronized


区别ThreadLocal 与 synchronized

ThreadLocal是一个线程隔离(或者说是线程安全)的变量存储的管理实体（注意：不是存储用的），它以Java类方式表现； 

synchronized是Java的一个保留字，只是一个代码标识符，它依靠JVM的锁机制来实现临界区的函数、变量在CPU运行访问中的原子性。 

两者的性质、表现及设计初衷不同，因此没有可比较性。

synchronized对块使用，用的是Object对象锁，对于方法使用，用的是this锁，对于静态方法使用，用的是Class对象的锁，只有使用同一个锁的代码，才是同步的。



理解ThreadLocal中提到的变量副本


事实上，我们向ThreadLocal中set的变量不是由ThreadLocal来存储的，而是Thread线程对象自身保存。

当用户调用ThreadLocal对象的set(Object o)时，该方法则通过Thread.currentThread()获取当前线程，将变量存入Thread中的一个Map内，而Map的Key就是当前的ThreadLocal实例。



Runnable与Thread


实现多线程，Runnable接口和Thread类是最常用的了，实现Runnable接口比继承Thread类会更有优势：

适合多个相同的程序代码的线程去处理同一个资源
可以避免java中的单继承的限制
增加程序的健壮性，代码可以被多个线程共享，代码和数据独立。


Java线程互斥和协作


阻塞指的是暂停一个线程的执行以等待某个条件发生（如某资源就绪）。Java 提供了大量方法来支持阻塞，下面让对它们逐一分析。


1、sleep()方法：sleep()允许指定以毫秒为单位的一段时间作为参数，它使得线程在指定的时间内进入阻塞状态，不能得到CPU 时间，指定的时间一过，线程重新进入可执行状态。

典型地，sleep() 被用在等待某个资源就绪的情形：测试发现条件不满足后，让线程阻塞一段时间后重新测试，直到条件满足为止。



2、（Java 5已经不推荐使用，易造成死锁！！） suspend()和resume()方法：两个方法配套使用，suspend()使得线程进入阻塞状态，并且不会自动恢复，必须其对应的 resume() 被调用，才能使得线程重新进入可执行状态。典型地，suspend() 和 resume() 被用在等待另一个线程产生的结果的情形：测试发现结果还没有产生后，让线程阻塞，另一个线程产生了结果后，调用resume()使其恢复。

stop()方法，原用于停止线程，也已经不推荐使用，因为stop时会解锁，可能造成不可预料的后果；推荐设置一个flag标记变量，结合interrupt()方法来让线程终止。



3.、yield() 方法：yield() 使得线程放弃当前分得的 CPU 时间，但是不使线程阻塞，即线程仍处于可执行状态，随时可能再次分得 CPU 时间。调用 yield() 的效果等价于调度程序认为该线程已执行了足够的时间从而转到另一个线程。


4.、wait() 和 notify() 方法：两个方法配套使用，wait() 使得线程进入阻塞状态，它有两种形式，一种允许指定以毫秒为单位的一段时间作为参数，另一种没有参数，前者当对应的 notify() 被调用或者超出指定时间时线程重新进入可执行状态，后者则必须对应的 notify() 被调用。



2和4区别的核心在于，前面叙述的所有方法，阻塞时都不会释放占用的锁（如果占用了的话），而这一对方法则相反。上述的核心区别导致了一系列的细节上的区别。


首先，前面叙述的所有方法都隶属于Thread 类，但是这一对却直接隶属于 Object 类，也就是说，所有对象都拥有这一对方法。因为这一对方法阻塞时要释放占用的锁，而锁是任何对象都具有的，调用任意对象的 wait() 方法导致线程阻塞，并且该对象上的锁被释放。而调用任意对象的notify()方法则导致因调用该对象的 wait() 方法而阻塞的线程中随机选择的一个解除阻塞（但要等到获得锁后才真正可执行）。



其次，前面叙述的所有方法都可在任何位置调用，但是这一对方法却必须在 synchronized 方法或块中调用，理由也很简单，只有在synchronized 方法或块中当前线程才占有锁，才有锁可以释放。同样的道理，调用这一对方法的对象上的锁必须为当前线程所拥有，这样才有锁可以释放。因此，这一对方法调用必须放置在这样的 synchronized 方法或块中，该方法或块的上锁对象就是调用这一对方法的对象。若不满足这一条件，则程序虽然仍能编译，但在运行时会出现 IllegalMonitorStateException 异常。



wait() 和 notify() 方法的上述特性决定了它们经常和synchronized 方法或块一起使用，将它们和操作系统的进程间通信机制作一个比较就会发现它们的相似性：synchronized方法或块提供了类似于操作系统原语的功能，它们的结合用于解决各种复杂的线程间通信问题。



关于 wait() 和 notify() 方法最后再说明三点：


　　第一：调用 notify() 方法导致解除阻塞的线程是从因调用该对象的 wait() 方法而阻塞的线程中随机选取的，我们无法预料哪一个线程将会被选择，所以编程时要特别小心，避免因这种不确定性而产生问题。


　　第二：除了 notify()，还有一个方法 notifyAll() 也可起到类似作用，唯一的区别在于，调用 notifyAll() 方法将把因调用该对象的wait()方法而阻塞的所有线程一次性全部解除阻塞。当然，只有获得锁的那一个线程才能进入可执行状态。


　　第三：wait/notify，是在作为监视器锁的对象上执行的，如果锁是a，执行b的wait，则会报java.lang.IllegalMonitorStateException。



关于interrupted，很容易理解错误，看两篇文章，如下：

http://www.blogjava.net/fhtdy2004/archive/2009/06/08/280728.html

http://www.blogjava.net/fhtdy2004/archive/2009/08/22/292181.html



演示线程间协作机制，wait/notify/condition


代码示例1（wait/notify）：

必须说这个代码是有缺陷的，会错失信号，想一想问题出在哪里，应该怎么完善？

```
/*
 * 线程之间协作问题：两个线程，一个打印奇数，一个打印偶数
 * 在调用wait方法时，都是用while判断条件的，而不是if，
 * 在wait方法说明中，也推荐使用while，因为在某些特定的情况下，线程有可能被假唤醒，使用while会循环检测更稳妥
 * */
public class OddAndEven {
 
	static int[] num = new int[]{1,2,3,4,5,6,7,8,9,10};
	static int index = 0;
 
	public static void main(String[] args) throws Exception {
		
		OddAndEven oae = new OddAndEven();
		//这里如果起超过2个线程，则可能出现所有的线程都处于wait状态的情况（网上很多代码都有这个Bug，要注意，这其实是一个错失信号产生的死锁问题，如果用notifyAll就不会有这个问题）		
		new Thread(new ThreadOdd(oae)).start();
		new Thread(new ThreadEven(oae)).start();
 
	}	
	static class ThreadOdd implements Runnable {
		private OddAndEven oae;
		public ThreadOdd(OddAndEven oae) {
			this.oae = oae;
		}
		@Override
		public void run() {
			while(index < 10) {
				oae.odd();
			}
		}	
	}
	static class ThreadEven implements Runnable {
		private OddAndEven oae;
		public ThreadEven(OddAndEven oae) {
			this.oae = oae;
		}
		@Override
		public void run() {
			while(index < 10) {
				oae.even();
			}
		}	
	}	
	//奇数
	public synchronized void odd() {
		while(index<10 && num[index] % 2 == 0) {
			try {
				wait();		//阻塞偶数
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
		if(index >= 10) return;
		System.out.println(Thread.currentThread().getName() + " 打印奇数 : " + num[index]);
		index++;
		notify();	//唤醒偶数线程
	}
	//偶数
	public synchronized void even() {
		while(index<10 && num[index] % 2 == 1) {
			try {
				wait();		//阻塞奇数
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
		if(index >= 10) return;
		System.out.println(Thread.currentThread().getName() + " 打印偶数 : " + num[index]);
		index++;
		notify();	//唤醒奇数线程
	}
}
```

代码示例2：

```

public class Test implements Runnable {   
  
    private String name;   
    private Object prev;   
    private Object self;   
  
    private Test(String name, Object prev, Object self) {   
        this.name = name;   
        this.prev = prev;   
        this.self = self;   
    }   
  
    @Override  
    public void run() {   
        int count = 10;   
        while (count > 0) {   
            synchronized (prev) {   
                synchronized (self) {   
                    System.out.print(name);   
                    count--;  
                    try{
                    Thread.sleep(1);
                    }
                    catch (InterruptedException e){
                     e.printStackTrace();
                    }
                    
                    self.notify();   
                }   
                try {   
                    prev.wait();   
                } catch (InterruptedException e) {   
                    e.printStackTrace();   
                }   
            }   
  
        }   
    }   
  
    public static void main(String[] args) throws Exception {   
        Object a = new Object();   
        Object b = new Object();   
        Object c = new Object();   
        Test pa = new Test("A", c, a);   
        Test pb = new Test("B", a, b);   
        Test pc = new Test("C", b, c);   
           
           
        new Thread(pa).start();
        Thread.sleep(10);
        new Thread(pb).start();
        Thread.sleep(10);
        new Thread(pc).start();
        Thread.sleep(10);
    }   
}
```

代码示例3（Condition实现生产者消费者模式）：

```
import java.util.PriorityQueue;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
 
public class ConditionTest {
	
    private int queueSize = 10;
    private PriorityQueue<Integer> queue = new PriorityQueue<Integer>(queueSize);
    private Lock lock = new ReentrantLock();
    private Condition notFull = lock.newCondition();
    private Condition notEmpty = lock.newCondition();
     
    public static void main(String[] args)  {
    	ConditionTest test = new ConditionTest();
    	
        Producer producer = test.new Producer();
        Consumer consumer = test.new Consumer();
          
        producer.start();
        consumer.start();
    }
      
    class Consumer extends Thread {          
        @Override
        public void run() {
            consume();
        }          
        private void consume() {
            while(true){
                lock.lock();
                try {
                    while(queue.size() == 0){
                        try {
                            System.out.println("队列空，等待数据");
                            notEmpty.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    queue.poll();                //每次移走队首元素
                    notFull.signal();
                    System.out.println("从队列取走一个元素，队列剩余"+queue.size()+"个元素");
                } finally{
                    lock.unlock();
                }
            }
        }
    }
      
    class Producer extends Thread {          
        @Override
        public void run() {
            produce();
        }          
        private void produce() {
            while(true){
                lock.lock();
                try {
                    while(queue.size() == queueSize){
                        try {
                            System.out.println("队列满，等待有空余空间");
                            notFull.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    queue.offer(1);        //每次插入一个元素
                    notEmpty.signal();
                    System.out.println("向队列取中插入一个元素，队列剩余空间："+(queueSize-queue.size()));
                } finally{
                    lock.unlock();
                }
            }
        }
    }
}
```

代码示例4（两把锁的生产者消费者模式）：

```
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
 
/**
 * 程序是Catch住InterruptedException，还是走Thread.interrupted()，实际是不确定的
 */
public class Restaurant {
	public Meal meal;
	public ExecutorService exec = Executors.newCachedThreadPool();
	public WaitPerson waitPerson = new WaitPerson(this);
	public Chef chef = new Chef(this);
	public Restaurant() {
		exec.execute(waitPerson);
		exec.execute(chef);
	}
	public static void main(String[] args) {
		new Restaurant();
	}
}
 
class Meal {
	private final int orderNum;
	public Meal(int orderNum) {this.orderNum = orderNum;}
	public String toString() {return "Meal " + orderNum;}
}
 
class WaitPerson implements Runnable {
	private Restaurant restaurant;
	public WaitPerson(Restaurant restaurant) {this.restaurant = restaurant;}
	public void run() {
		try {
			while(!Thread.interrupted()) {
				synchronized(this) {
					while(restaurant.meal == null) {
						wait();
					}
				}
				System.out.println("WaitPerson got " + restaurant.meal);
				synchronized(restaurant.chef) {
					restaurant.meal = null;
					restaurant.chef.notifyAll();
				}
			}
		}
		catch(InterruptedException e) {
			System.out.println(Thread.currentThread().getName() + " " + e.toString());
		}
	}
}
 
class Chef implements Runnable {
	private Restaurant restaurant;
	private int count = 0;
	public Chef(Restaurant restaurant) {this.restaurant = restaurant;}
	public void run() {
		try {
			while(!Thread.interrupted()) {
				synchronized(this) {
					while(restaurant.meal != null) {
						wait();
					}
				}
				if(++count == 10) {
					restaurant.exec.shutdownNow();
				}
				System.out.println("Order up!");
				synchronized(restaurant.waitPerson) {
					restaurant.meal = new Meal(count);
					restaurant.waitPerson.notifyAll();
				}
			}
		}
		catch(InterruptedException e) {
			System.out.println(Thread.currentThread().getName() + " " + e.toString());
		}
	}
}
```

### 反面教材代码演示：

```
class SuspendAndResume {
        private final static Object object = new Object();
 
        static class ThreadA extends Thread {
                public void run() {
                        synchronized(object) {
                                System.out.println("start...");
                                Thread.currentThread().suspend();
                                System.out.println("thread end");
                        }
                }
        }
 
        public static void main(String[] args) throws InterruptedException {
                ThreadA t1 = new ThreadA();
                ThreadA t2 = new ThreadA();
                t1.start();
                t2.start();
                Thread.sleep(100);
                System.out.println(t1.getState());
                System.out.println(t2.getState());
                t1.resume();
                t2.resume();
        }
}
```

程序输出结果如下：

```
localhost:test puma$ sudo java SuspendAndResume
start...
RUNNABLE
BLOCKED
thread end
start...
```

关于suspend()/resume()这两个方法，类似于wait()/notify()，但是它们不是等待和唤醒线程。suspend()后的线程处于RUNNING状态，而不是WAITING状态，但是线程本身在这里已经挂起了，线程本身饿状态就开始对不上号了。


以上的例子解释如下：



首先t1.start()/t2.start()，main睡sleep10秒，让两个子线程都进入运行的区域；

打印状态，t1运行，t2被synchronized阻塞；

t1.resume()，此时t1打印thread end，马上执行t2.resume()，此时由于t1的synchronized还没来得及释放锁，所以这段代码是在t2的synchronized外执行的，也就是在t2.suspend()之前执行的，所以是无效的；而当t2线程被挂起时，输出start，但是由于t2.suspend()已经被执行完了，所以t2就会一直处于挂起状态，一直持有锁不释放，这些信息的不一致就导致了各种资源无法释放的问题。

对于这个程序，如果在t1.resume()和t2.resume()之间增加一个Thread.sleep()，可以看到又正常执行了。

总得来说，问题应当出在线程状态对外看到的是RUNNING状态，外部程序并不知道这个对象挂起了需要去做resume()操作。另外，它并不是基于对象来完成这个动作的，因此suspend()和wait()相关的顺序性很难保证。所以suspend()和resume()不推荐使用了。

反过来想，这也更加说明了wait()和notify()为什么要基于对象（而不是线程本身）来做数据结构，因为要控制生产者和消费者之间的关系，它需要一个临界区来控制它们之间的平衡。它不是随意地在线程上做操作来控制资源的，而是由资源反过来控制线程状态的。当然wait()和notify()并非不会导致死锁，只是它们的死锁通常是程序设计不当导致的，并且在通常情况下是可以通过优化来解决的。



同步队列


wait和notify以一种非常低级的方式解决了任务互操作问题，即每次交互时都握手。在许多情况下，可以瞄向更高的抽象级别，使用同步队列来解决任务协作问题，同步队列在任何时刻都只允许一个操作插入或移除元素。

如果消费者任务试图从队列中获取对象，而该队列此时为空，那么这些队列还可以挂起消费者任务，并且当有更多的元素可用时恢复消费者任务。阻塞队列可以解决非常大量的问题，而其方式与wait和notify相比，则简单并可靠的多。



代码示例1（将多个LiftOff的执行串行化，消费者是LiftOffRunner，将每个LiftOff对象从BlockingQueue中推出并直接运行）：

```
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.SynchronousQueue;
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
 
public class TestBlockingQueues {
	public static void main(String[] args) {
		test("LinkedBlockingQueue",new LinkedBlockingQueue<LiftOff>());	//Unlimited size
		test("ArrayBlockingQueue",new ArrayBlockingQueue<LiftOff>(3));	//Fixed size
		test("SynchronousQueue",new SynchronousQueue<LiftOff>());	//Size of 1
	}
	private static void test(String msg,BlockingQueue<LiftOff> queue) {
		System.out.println(msg);
		LiftOffRunner runner = new LiftOffRunner(queue);
		Thread t = new Thread(runner);
		t.start();
		for(int i = 0; i < 5; i++) {
			runner.add(new LiftOff(i));
		}
		getKey("Press 'Enter' (" + msg + ")");
		t.interrupt();
		System.out.println("Finished " + msg + " test");
	}
	private static void getKey() {
		try {
			new BufferedReader(new InputStreamReader(System.in)).readLine();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
	private static void getKey(String message) {
		System.out.println(message);
		getKey();
	}
}
 
class LiftOffRunner implements Runnable {
	private BlockingQueue<LiftOff> rockets;
	public LiftOffRunner(BlockingQueue<LiftOff> queue) {rockets = queue;}
	public void add(LiftOff lo) {
		try {
			rockets.put(lo);
		} catch (InterruptedException e) {
			System.out.println("Interrupted during put()");
		}
	}
	@Override
	public void run() {
		try
		{
			while(!Thread.interrupted()) {
				LiftOff rocket = rockets.take();
				rocket.run();
			}
		} catch (InterruptedException e) {
			System.out.println("Waking from take()");
		}
		System.out.print("Exiting LiftOffRunner");
	}
}
 
class LiftOff {
	private int num;
	public LiftOff(int num) {this.num = num;}
	public void run() {
		System.out.println(Thread.currentThread().getName() + " " + num);
	}
}
```

代码示例2：

线程间通过BlockingQueue协作，3个任务，一个做面包，一个对面包抹黄油，一个对抹过黄油的面包抹果酱。整个代码没有显示的使用同步，任务之间完成了很好地协作；因为同步由队列（其内部是同步的）和系统的设计隐式的管理了。每片Toast在任何时刻都只有一个任务在操作。因为队列的阻塞，使得处理过程将被自动的挂起和恢复。

我们可以看到通过使用BlockingQueue带来的简化十分明显，在使用显式的wait/notify时存在的类和类之间的耦合被消除了，因为每个类都只和他得BlockingQueue通信。

```
import java.util.Random;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;
 
public class ToastOMatic {
	public static void main(String[] args) throws Exception {
		ToastQueue dryQueue = new ToastQueue(),
				butteredQueue = new ToastQueue(),
				finishedQueue = new ToastQueue();
		ExecutorService exec = Executors.newCachedThreadPool();
		exec.execute(new Toaster(dryQueue));
		exec.execute(new Butterer(dryQueue,butteredQueue));
		exec.execute(new Jammer(butteredQueue,finishedQueue));
		exec.execute(new Eater(finishedQueue));
		TimeUnit.SECONDS.sleep(5);
		exec.shutdownNow();
	}
}
 
class Toast {
	public enum Status {DRY,BUTTERED,JAMMED};
	private Status status = Status.DRY;
	private final int id;
	public Toast(int idn) {id = idn;}
	public void butter() {status = Status.BUTTERED;}
	public void jam() {status = Status.JAMMED;}
	public Status getStatus() {return status;}
	public int getId() {return id;}
	public String toString() {return "Toast " + id + " : " + status;}
}
 
@SuppressWarnings("serial")
class ToastQueue extends LinkedBlockingQueue<Toast> {}
 
class Toaster implements Runnable {
	private ToastQueue toastQueue;
	private int count = 0;
	private Random rand = new Random(47);
	public Toaster(ToastQueue tq) {toastQueue = tq;}
	public void run() {
		try {
			while(!Thread.interrupted()) {
				TimeUnit.MICROSECONDS.sleep(100 + rand.nextInt(500));
				//Make toast
				Toast t = new Toast(count++);
				System.out.println(t);
				//Insert into queue
				toastQueue.put(t);
			}
		} catch(InterruptedException e) {
			System.out.println("Toaster interrupted");
		}
		System.out.println("Toaster off");
	}
}
 
//Apply butter to toast
class Butterer implements Runnable {
	private ToastQueue dryQueue,butteredQueue;
	public Butterer(ToastQueue dry, ToastQueue buttered) {
		dryQueue = dry;
		butteredQueue = buttered;
	}
	public void run() {
		try {
			while(!Thread.interrupted()) {
				//Blocks until next piece of toast is available
				Toast t = dryQueue.take();
				t.butter();
				System.out.println(t);
				butteredQueue.put(t);
			}
		} catch(InterruptedException e) {
			System.out.println("Butterer interrupted");
		}
		System.out.println("Butterer off");
	}
}
 
//Apply jam to buttered toast
class Jammer implements Runnable {
	private ToastQueue butteredQueue,finishedQueue;
	public Jammer(ToastQueue buttered,ToastQueue finished) {
		butteredQueue = buttered;
		finishedQueue = finished;
	}
	public void run() {
		try {
			while(!Thread.interrupted()) {
				//Blocks until next piece of toast is available
				Toast t = butteredQueue.take();
				t.jam();
				System.out.println(t);
				finishedQueue.put(t);;
			}
		} catch(InterruptedException e) {
			System.out.println("Jammer interrupted");
		}
		System.out.println("Jammer off");
	}
}
 
//Consume the toast
class Eater implements Runnable {
	private ToastQueue finishedQueue;
	private int counter = 0;
	public Eater(ToastQueue finished) {
		finishedQueue = finished;
	}
	public void run() {
		try {
			while(!Thread.interrupted()) {
				//Blocks until next piece of toast is available
				Toast t = finishedQueue.take();
				//Verify that the toast is coming in order, and that all pieces are getting jammed
				if(t.getId() != counter++ || t.getStatus() != Toast.Status.JAMMED) {
					System.out.println(">>>> Error : " + t);
					System.exit(1);
				} else {
					System.out.println("Chomp! " + t);
				}
			}
		} catch(InterruptedException e) {
			System.out.println("Eater interrupted");
		}
		System.out.println("Eater off");
	}
}
```

任务间使用管道进行输入输出


Java输入输出类库中的PipedWriter（允许任务向管道写），PipedReader（允许不同任务从同一个管道中读取），这个模型可以看成“生产者-消费者”问题的变体，这里的管道就是一个封装好的解决方案。

管道基本上是一个阻塞队列，存在于多个引入BlockingQueue之前的Java版本中。



代码示例1：

当Receiver调用read时，如果没有更多地数据，管道将自动阻塞。

Sender和Receiver是在main中启动的，即对象构造彻底完成之后。如果启动了一个没有彻底构造完毕的对象，在不同的平台上，管道可能产生不一致的行为。相比较而言，BlockingQueue使用起来更加健壮而容易。

shutdownNow被调用时，可以看到PipedReader和普通IO之间最重要的差异，PipedReader是可以中断的。

```

import java.io.IOException;
import java.io.PipedReader;
import java.io.PipedWriter;
import java.util.Random;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
 
public class PipedIO {
	public static void main(String[] args) throws Exception {
		Sender sender = new Sender();
		Receiver receiver = new Receiver(sender);
		ExecutorService exec = Executors.newCachedThreadPool();
		exec.execute(sender);
		exec.execute(receiver);
		TimeUnit.SECONDS.sleep(5);
		exec.shutdownNow();
	}
}
 
class Sender implements Runnable {
	private Random rand = new Random(47);
	private PipedWriter out = new PipedWriter();
	public PipedWriter getPipedWriter() { return out; }
	public void run() {
		try {
			while(true) {
				for(char c = 'A'; c <= 'Z'; c++) {
					out.write(c);
					TimeUnit.MILLISECONDS.sleep(rand.nextInt(50));
				}
			}
		} catch(IOException e) {
			System.out.println(e + " Sender write exception");
		} catch(InterruptedException e) {
			System.out.println(e + " Sender sleep interrupted");
		}
	}
}
 
class Receiver implements Runnable {
	private PipedReader in;
	public Receiver(Sender sender) throws IOException {
		in = new PipedReader(sender.getPipedWriter());
	}
	public void run() {
		try {
			while(true) {
				//Blocks until characters are there
				System.out.println("Read: " + (char)in.read() + ", ");
			}
		} catch(IOException e) {
			System.out.println(e + " Receiver read exception");
		}
	}
}
```

线程是JVM级别的

我们知道静态变量是ClassLoader级别的，如果Web应用程序停止，这些静态变量也会从JVM中清除。

但是线程则是JVM级别的，如果用户在Web应用中启动一个线程，这个线程的生命周期并不会和Web应用程序保持同步。

也就是说，即使停止了Web应用，这个线程依旧是活跃的。

正是因为这个很隐晦的问题，所以很多有经验的开发者不太赞成在Web应用中私自启动线程。



获取异步线程的返回结果

通过java.util.concurrent包种的相关类，实现异步线程返回结果的获取。代码演示例子如下：

```
import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
 
public class FutureTest {
 
        public static class TaskRunnable implements Runnable {
                @Override
                public void run() {
                        System.out.println("runnable");
                }
        }
 
        public static class TaskCallable implements Callable<String> {
                private String s;
 
                public TaskCallable(String s) {
                        this.s = s;
                }
 
                @Override
                public String call() throws Exception {
                        System.out.println("callable");
                        return s;
                }
        }
 
        public static void main(String[] args) {
                ExecutorService es = Executors.newCachedThreadPool();
                for (int i = 0; i < 100; i++) {
                        es.submit(new TaskRunnable());
                        System.out.println(i);
                }
 
                List<Future<String>> futList = new LinkedList<Future<String>>();
                for (int i = 0; i < 100; i++) {
                        futList.add(es.submit(new TaskCallable(String.valueOf(i))));
                }
 
                for (Future<String> fut : futList) {
                        try {
                                System.out.println(fut.get());
                        } catch (InterruptedException e) {
                                // TODO Auto-generated catch block
                                e.printStackTrace();
                        } catch (ExecutionException e) {
                                // TODO Auto-generated catch block
                                e.printStackTrace();
                        }
                }
        }
}
```

## ReentrantLock和synchronized两种锁定机制的对比

```
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;
import java.net.URI;
import java.net.URISyntaxException;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map.Entry;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
 
public class Test {
	public static int I;
	public static Object oLock = new Object();
	public static Lock lock = new ReentrantLock();
	public static void main(String[] args) throws Exception {
		long b = System.currentTimeMillis();
		List<Thread> t1= new ArrayList<Thread>();
		for(int j=0;j<30;j++)
		{
			t1.add(new Thread(new R1()));
		}
		for(Thread t: t1) t.start();
		
		for(Thread t:t1) {
			t.join();
		}
		long e = System.currentTimeMillis();		
		System.out.println(Test.I + "  |  " + (e-b));
 
 
		Test.I = 0;
		b = System.currentTimeMillis();
		List<Thread> t2= new ArrayList<Thread>();
		for(int j=0;j<30;j++)
		{
			t2.add(new Thread(new R2()));
		}
		for(Thread t: t2) t.start();
		
		for(Thread t:t2) {
			t.join();
		}
		e = System.currentTimeMillis();		
		System.out.println(Test.I + "  |  " + (e-b));
	}
}
class R1 implements Runnable {
	@Override
	public void run() {		
		for(int i=0;i<1000000;i++)
		{
			Test.lock.lock();
			Test.I++;
			Test.lock.unlock();
		}
		
	}
}
class R2 implements Runnable {
	@Override
	public void run() {		
		for(int i=0;i<1000000;i++)
		{
			synchronized("") {
				Test.I++;
			}
		}
		
	}
}
```

经过测试，输出结果分别为：

Windows7(2核，2G内存），结果：3000000  |  2890，和3000000  |  8198，性能差距还是比较明显；

Mac10.7（4核，4G内存），结果：3亿次计算，ReentrantLock用8秒，synchronized反而只用4秒，结果反过来了；

RHEL6.1（24核，64G内存），结果：3亿次计算，二者相差不多，都是20-50之间，但是ReentrantLock表现更好一些。

ReentrantLock利用的是“非阻塞同步算法与CAS(Compare and Swap)无锁算法”，是CPU级别的，参考网址：

http://www.cnblogs.com/Mainz/p/3556430.html

关于两种锁机制的更多比较，请参阅：http://www.ibm.com/developerworks/cn/java/j-jtp10264/index.html



关于线程安全的N种实现场景


（1）synchronized

（2）immutable对象是自动线程安全的

（3）volatile，被volatile修饰的属性，对于读操作是线程安全的

（4）ConcurrentHashMap之类，java.util.concurrent包中的一些并发操作类，是线程安全的，但是没有使用synchronized关键字，实现巧妙，利用的基本特性是：volatile、Compare And Swap、分段，对于读不加锁（volatile保证线程安全），对于写，对于相关的segment通过ReentrantLock加锁



Java线程死锁

代码示例：

```
public class Test {
	public static void main(String[] args) {		
		Runnable t1 = new DeadLock(true);
		Runnable t2 = new DeadLock(false);
		new Thread(t1).start();
		new Thread(t2).start();
	}
}
 
class DeadLock implements Runnable {
	private static Object lock1 = new Object();
	private static Object lock2 = new Object();
	private boolean flag;
	public DeadLock(boolean flag) {
		this.flag = flag;
	}
	@Override
	public void run() {
		if(flag) {
			synchronized(lock1) {
				try {
					Thread.sleep(1000);	//保证晚于另一个线程锁lock2，目的是产生死锁
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				synchronized(lock2) {
					System.out.println("flag=true:死锁了，还能print的出来吗？");
				}
			}
		} else {
			synchronized(lock2) {
				try {
					Thread.sleep(1000);	//保证晚于另一个线程锁lock1，目的是产生死锁
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				synchronized(lock1) {
					System.out.println("flag=false:死锁了，还能print的出来吗？");
				}
			}
		}
	}	
}
```

死锁可以这样比喻：两个人吃饭，需要刀子和叉子，其中一人拿了刀子，等待叉子；另一个人拿了叉子，等待刀子；就死锁了。
Java线程死锁是一个经典的多线程问题，因为不同的线程都在等待那些根本不可能被释放的锁，导致所有的工作都无法完成；

导致死锁的根源在于不适当地运用“synchronized”关键词来管理线程对特定对象的访问。

如何避免死锁的设计规则：

（1）让所有的线程按照同样地顺序获得一组锁。这种方法消除了X和Y的拥有者分别等待对方的资源的问题。这也是避免死锁的一个通用的经验法则是:当几个线程都要访问共享资源A、B、C时，保证使每个线程都按照同样的顺序去访问它们，比如都先访问A，在访问B和C。

（2）将多个锁组成一组并放到同一个锁下。比如，把刀子和叉子，都放在一个新创建的（银器对象）下面，要获得子锁，先获得父锁。



更多


微博设计框架：http://mars914.iteye.com/blog/1218492  http://timyang.net/

避免多线程时，开销分布在调度上，可以采取的策略：减少线程到合适的程度、避免线程内IO、采用合适的优先级。

关于IO和多线程，有个案例，Redis是单线程的，支持10万的QPS；MemberCache是多线程的，性能反而不如Redis；也可以佐证，对于IO非常多的操作，多线程未必能提高更好的性能，即使是内存IO。

另外，听百分点公司的讲座时，他们分享了一个案例，当计算的性能瓶颈在硬盘时，把硬盘换成SSD，可以性能翻倍，所以，应该把SSD当做是便宜的内存来使用，而不应该是当做昂贵的硬盘来使用。

在java.util.concurrent中，还提供了很多有用的线程写作类，比如：

CountDownLatch：倒计时锁、CyclicBarrier：循环栅栏、DelayQueue：延迟队列、PriorityBlockingQueue：优先级队列、ScheduledThreadPoolExecutor：定时任务、Semaphore：信号量、Exchanger：交互栅栏。

# 后记

