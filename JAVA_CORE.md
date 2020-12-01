# **JAVA**


## 并发

### java内存模型

Java内存模型规定了所有的变量都存储在主内存中，每条线程还有自己的工作内存，线程的工作内存中保存了该线程中是用到的变量的主内存副本拷贝，线程对变量的所有操作都必须在工作内存中进行，而不能直接读写主内存。不同的线程之间也无法直接访问对方工作内存中的变量，线程间变量的传递均需要自己的工作内存和主存之间进行数据同步进行

- 原子性

	一次操作不能再拆分

- 可见性

	当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值

	- volatile

		volatile禁止指令重排，强迫线程每次读取变量都去主存
		
		可见行：被该关键字修饰的变量，在每次写操作之后，都会加入一条store内存屏障命令，将最新值刷新到主内存，在每个读操作之前，都会加入一条load内存屏障命令，强制线程从主内存加载该变量到工作内存  
		
		禁止指令重排：插入内存屏障，告诉CPU和编译器，无论什么命令都不能与该指令重排

- 有序性

	程序执行的顺序按照代码的先后顺序执行

	- <-against->

		- 指令重排序

			处理器为了提高程序运行效率，可能会对输入代码进行优化，它不保证程序中各个语句的执行先后顺序同代码中的顺序一致，但是它会保证程序最终执行结果和代码顺序执行的结果是一致的

			- Java内存模型中，允许编译器和处理器对指令进行重排序，重排序会影响到多线程并发执行的正确性

### [各种锁](https://tech.meituan.com/2018/11/15/java-lock.html)

- 乐观锁

	如其名，适用于读多写少的场景，主要实现方式CAS

- 悲观锁

- synchronized

	1.修饰方法：作用于当前调用该方法的实例对象  
	2.修饰静态方法：作用于类的所有对象实例  
	3.修饰代码块：  
	  a）synchronized(this)：锁定当前对象  
	  b）synchronized(Demo.class)：锁定所有Demo对象  
	
	
	4.总结：  
	偏向锁通过对比Mark Word解决加锁问题，避免执行CAS操作。而轻量级锁时通过CAS操作和自旋来解决加锁问题，避免线程阻塞和唤醒影响性能。重量级锁是将除了拥有锁的线程以外的线程都阻塞

	- Java对象头

		- Mark Word（标记字段）

		- Klass Pointer（类型指针）

			对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例

	- 无锁

		不锁定资源，所有的线程都能访问并修改同一个资源，但同时只有一个线程能修改成功（基于CAS）

	- 偏向锁

		一个对象刚开始实例化的时候，没有任何线程来访问它的时候。它是可偏向的，意味着，它现在认为只可能有一个线程来访问它，所以当第一个线程来访问它的时候，它会偏向这个线程，此时，对象持有偏向锁。偏向第一个线程，这个线程在修改对象头成为偏向锁的时候使用CAS操作，并将对象头中的ThreadID改成自己的ID，之后再次访问这个对象时，只需要对比ID，不需要再使用CAS在进行操作。一旦有第二个线程访问这个对象，因为偏向锁不会主动释放，所以第二个线程可以看到对象时偏向状态，这时表明在这个对象上已经存在竞争了，检查原来持有该对象锁的线程是否依然存活，如果挂了，则可以将对象变为无锁状态，然后重新偏向新的线程，如果原来的线程依然存活，则马上执行那个线程的操作栈，检查该对象的使用情况，如果仍然需要持有偏向锁，则偏向锁升级为轻量级锁，如果不存在使用了，则可以将对象回复成无锁状态，然后重新偏向

	- 轻量级锁

		若当前只有一个等待线程，则该线程通过自旋进行等待。但是当自旋超过一定的次数，或者一个线程在持有锁，一个在自旋，又有第三个来访时，轻量级锁升级为重量级锁  
		
		获取过程：使用CAS操作尝试将对象的Mark Word更新为指向线程的Lock Record的指针，并将Lock record里的owner指针指向对象的mark word

	- 重量级锁

		升级为重量级锁时，锁标志的状态值变为“10”，此时Mark Word中存储的是指向堆中monitor对象的指针，此时等待锁的线程都会进入阻塞状态
		
		依赖于操作系统Mutex Lock实现的锁  
		
		当一个线程尝试获得锁时，如果该锁已经被占用，则会将该线程封装成一个ObjectWaiter对象插入到ContentionList的队列尾部，然后暂停当前线程。当持有锁的线程释放锁前，会将ContentionList中的所有元素移动到EntryList中去，并唤醒EntryList的队首线程。  
		如果一个线程在同步块中调用了Object#wait方法，会将该线程对应的ObjectWaiter从EntryList移除并加入到WaitSet中，然后释放锁。当wait的线程被notify之后，会将对应的ObjectWaiter从WaitSet移动到EntryList中

		- Monitor

			每一个java对象就有一把看不见的锁，被称为内部锁或者Monitor锁
			
			当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，则获得锁成功，如果重入锁将再次加1
			
			通过Monitor来实现线程同步，Monitor是依赖于底层的操作系统的Mutex Lock（互斥锁）来实现的线程同步

			- 对方法加锁时

				当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先持有monitor， 然后再执行方法，方法执行完毕释放monitor

			- 对代码块加锁时

				VM底层使用指令entermonitor用来获取monitor的所有权，进行加锁。离开时，使用指令monitorexit来释放对monitor占用，即解锁。只有解锁之后，其他线程才可以竞争monitor

	- 总结

		1.线程遇到monitorenter或者ACC_SYNCHRONIZED就被要求获取monitor对象才可以执行同步代码（不管是什么级别的锁）
		
		2.如果是偏向锁，对象头Mark Word存储持有锁的线程ID  
		
		3.如果是轻量级锁，对象头的Mark Word存储线程Lock Record的指针  
		
		4.如果是重量级锁，对象头Mark Word中存储的是指向堆中monitor对象的指针

- 死锁

	两个任务以不合理的顺序互相争夺资源

	- 简单例子

		public static void main(String[] args) {  
		    final Object a = new Object();  
		    final Object b = new Object();  
		    Thread threadA = new Thread(new Runnable() {  
		        public void run() {  
		            synchronized (a) {  
		                try {  
		                    System.out.println("now i in threadA-locka");  
		                    Thread.sleep(1000l);  
		                    synchronized (b) {  
		                        System.out.println("now i in threadA-lockb");  
		                    }  
		                } catch (Exception e) {  
		                    // ignore  
		                }  
		            }  
		        }  
		    });  
		
		    Thread threadB = new Thread(new Runnable() {  
		        public void run() {  
		            synchronized (b) {  
		                try {  
		                    System.out.println("now i in threadB-lockb");  
		                    Thread.sleep(1000l);  
		                    synchronized (a) {  
		                        System.out.println("now i in threadB-locka");  
		                    }  
		                } catch (Exception e) {  
		                    // ignore  
		                }  
		            }  
		        }  
		    });  
		
		    threadA.start();  
		    threadB.start();  
		}

	- 防止死锁

		- 确定顺序获取锁

			- 银行家算法

		- 超时放弃

			使用Lock API取代synchronized

	- 线程池死锁

		final ExecutorService executorService = Executors.newSingleThreadExecutor();  
		Future<Long> f1 = executorService.submit(new Callable<Long>() {  
		
		    public Long call() throws Exception {  
		        System.out.println("start f1");  
		        Thread.sleep(1000);//延时
		        Future<Long> f2 =   
		           executorService.submit(new Callable<Long>() {  
		
		            public Long call() throws Exception {  
		                System.out.println("start f2");  
		                return -1L;  
		            }  
		        });  
		        System.out.println("result" + f2.get());  
		        System.out.println("end f1");  
		        return -1L;  
		    }  
		});  
		
		线程池的任务1依赖任务2的执行结果，但是线程池是单线程的，也就是说任务1不执行完，任务2永远得不到执行

	- 网络连接池死锁

		// 连接1
		final MultiThreadedHttpConnectionManager connectionManager1 = new MultiThreadedHttpConnectionManager();  
		final HttpClient httpClient1 = new HttpClient(connectionManager1);  
		httpClient1.getHttpConnectionManager().getParams().setMaxTotalConnections(1);  //设置整个连接池最大连接数
		
		// 连接2
		final MultiThreadedHttpConnectionManager connectionManager2 = new MultiThreadedHttpConnectionManager();  
		final HttpClient httpClient2 = new HttpClient(connectionManager2);  
		httpClient2.getHttpConnectionManager().getParams().setMaxTotalConnections(1);  //设置整个连接池最大连接数
		
		ExecutorService executorService = Executors.newFixedThreadPool(2);  
		executorService.submit(new Runnable() {  
		    public void run() {  
		        try {  
		            PostMethod httpost = new PostMethod("http://www.baidu.com");  
		            System.out.println(">>>> Thread A execute 1 >>>>");  
		            httpClient1.executeMethod(httpost);  
		            Thread.sleep(5000l);  
		
		            System.out.println(">>>> Thread A execute 2 >>>>");  
		            httpClient2.executeMethod(httpost);  
		            System.out.println(">>>> End Thread A>>>>");  
		        } catch (Exception e) {  
		            // ignore  
		        }  
		    }  
		});  
		
		executorService.submit(new Runnable() {  
		    public void run() {  
		        try {  
		            PostMethod httpost = new PostMethod("http://www.baidu.com");  
		            System.out.println(">>>> Thread B execute 2 >>>>");  
		            httpClient2.executeMethod(httpost);  
		            Thread.sleep(5000l);  
		
		            System.out.println(">>>> Thread B execute 1 >>>>");  
		            httpClient1.executeMethod(httpost);  
		            System.out.println(">>>> End Thread B>>>>");  
		
		        } catch (Exception e) {  
		            // ignore  
		        }  
		    }  
		});  
		
		此时有两个线程A和B，两个数据库连接池N1和N2，连接池大小都只有1，如果线程A按照先N1后N2的顺序获得网络连接，而线程B按照先N2后N1的顺序获得网络连接，并且两个线程在完成执行之前都不释放自己已经持有的链接，因此也造成了死锁

- 可重入锁&非可重入锁

	当线程尝试获取锁时，可重入锁先尝试获取并更新state值，如果state == 0表示没有其他线程在执行同步代码，则把state置为1，当前线程开始执行。如果state != 0，则判断当前线程是否是获取到这个锁的线程，如果是的话执行state +1，且当前线程可以再次获取锁。而非可重入锁是直接去获取并尝试更新当前state的值，如果state != 0的话会导致其获取锁失败，当前线程阻塞

- 公平&非公平锁

	公平锁是指多个线程按照申请锁的顺序来获取锁，线程直接进入队列中排队，队列中的第一个线程才能获得锁。公平锁的优点是等待锁的线程不会饿死。缺点是整体吞吐效率相对非公平锁要低，等待队列中除第一个线程以外的所有线程都会阻塞，CPU唤醒阻塞线程的开销比非公平锁大。
	
	非公平锁是多个线程加锁时直接尝试获取锁，获取不到才会到等待队列的队尾等待。但如果此时锁刚好可用，那么这个线程可以无需阻塞直接获取到锁，所以非公平锁有可能出现后申请锁的线程先获取锁的场景。非公平锁的优点是可以减少唤起线程的开销，整体的吞吐效率高，因为线程有几率不阻塞直接获得锁，CPU不必唤醒所有线程。缺点是处于等待队列中的线程可能会饿死，或者等很久才会获得锁

### JUC：AQS

1.核心变量state(1已加锁，0未加锁) (基于CAS修改)  

2.关键变量，用来记录当前加锁的是哪个线程(用于重入锁判断)  

3.维护着一个FIFO的双向同步队列，当线程获取同步状态失败后，则会加入到这个CLH同步队列的队尾(添加队尾的过程CAS)自旋，自旋过程中若立即获取到同步状态(前置结点为head并且尝试获取同步状态成功)则可以直接执行，若无法立即获取到同步状态则会将前置结点置为SIGNAL状态同时自身通过LockSupport.park()方法进入阻塞状态，等待LockSupport.unpark()方法唤醒  

4.双向链表中，第一个节点为虚节点，其实并不存储任何信息，只是占位。真正的第一个有数据的节点，是在第二个节点开始的  

5.独占式释放锁唤醒头节点下个节点，共享式遍历队列唤醒所有节点

- 独占式

	######获取
	调用入口方法acquire(arg)
	调用模版方法tryAcquire(arg)尝试获取锁，若成功则返回，若失败则走下一步
	将当前线程构造成一个Node节点，并利用CAS将其加入到同步队列到尾部，然后该节点对应到线程进入自旋状态
	自旋时，首先判断其前驱节点是否为头节点&是否成功获取同步状态，两个条件都成立，则将当前线程的节点设置为头节点的后继节点，如果不是，则利用LockSupport.park(this)将当前线程挂起 ,等待被前驱节点唤醒
	
	######释放
	调用入口方法release(arg)
	调用模版方法tryRelease(arg)释放同步状态
	获取当前节点的下一个节点  
	利用LockSupport.unpark(currentNode.next.thread)唤醒后继节点（接获取的第四步） 
	
	######问题  
	1.为什么获取锁失败还自旋？  
	因为设计者认为其他获取锁的线程处理同步代码很快，所以在获取锁失败的时候，稍等一会自旋说不定能成功获得锁

	- lock

		lock()  等待获取锁
		
		lockInterruptibly()  可中断等待获取锁
		
		tryLock() 尝试获取锁，立即返回true或false
		
		tryLock(long time, TimeUnit unit)    指定时间内等待获取锁
		
		unlock()      释放锁
		
		newCondition()   返回一个绑定到此 
		Lock 实例上的 Condition 实例

		- StampedLock

			1.不可重入
			
			2.提供乐观读锁，基于CAS思想

		- ReadWriteLock

			- ReentrantReadWriteLock

				获取读锁条件：没有线程持有写锁和没有线程请求获取写锁  
				释放读锁：读锁线程数减一并唤醒其它线程  
				
				获取写锁条件：没有线程持有写锁或写锁  
				写锁线程数减一并唤醒其它线程

		- lock()基于tryAcquire()

		- unlock()基于tryRelease()

		- ReentrantLock

			- Condition条件变量

				- await()

					调用该方法的线程成功获取了锁的线程，也就是同步队列中的首节点，该方法会将当前线程构造成节点并加入等待队列中，然后释放同步状态，唤醒同步队列中的后继节点，然后当前线程会进入等待状态

				- lockInterruptibly()

					- 获取锁，如果锁正在被其他线程使用中，则等待，但在等待期间可以中断等待:thread.interrupt()

				- signal()

					等待队列中的节点被唤醒，开始尝试获取同步状态。如果不是通过其他线程调用Condition.signal()方法唤醒，而是对等待线程进行中断，则会抛出InterruptedException

					- 利用锁机制，每个condition绑定着一个lock，可以实现线程精确唤醒

				- signalAll()

					- For循环唤醒所有

				- 底层原理

					- await()-等待队列

						等待队列是一个FIFO的队列，在队列中的每个节点都包含了一个线程引用，该线程就是在Condition对象上等待的线程，如果一个线程调用了Condition.await()方法，那么该线程将会释放锁、构造成节点加入等待队列并进入等待状态（相当于同步队列的首节点（获取了锁的节点）移动到等待队列中）

					- signal()-通知

						调用Condition的signal()方法，将会唤醒在等待队列中等待时间最长的节点（首节点），在唤醒节点之前，会将节点移到同步队列中

					- 节点移动

						1.调用await(), signal(), signalAll()方法将在同步队列和等待队列之间移动节点

				- demo

					public class ConditionDemo {  
					    public static void main(String[] args) {  
					        final ReentrantLock reentrantLock = new ReentrantLock();  
					        final Condition condition = reentrantLock.newCondition();  
					
					        new Thread(new Runnable() {  
					            @Override  
					            public void run() {  
					                try {  
					                    reentrantLock.lock();  
					                    System.out.println(Thread.currentThread().getName() + "在等待被唤醒");
					                    condition.await();  
					                    System.out.println(Thread.currentThread().getName() + "恢复执行了");
					                } catch (InterruptedException e) {  
					                    e.printStackTrace();  
					                } finally {  
					                    reentrantLock.unlock();  
					                }  
					            }  
					        }, "thread1").start();  
					
					        new Thread(new Runnable() {  
					            @Override  
					            public void run() {  
					                try {  
					                    reentrantLock.lock();  
					                    System.out.println(Thread.currentThread().getName() + "抢到了锁");
					                    condition.signal();  
					                    System.out.println(Thread.currentThread().getName() + "唤醒其它等待的线程");
					                } catch (Exception e) {  
					                    e.printStackTrace();  
					                } finally {  
					                    reentrantLock.unlock();  
					                }  
					            }  
					        }, "thread2").start();  
					    }  
					}  
					
					thread1在等待被唤醒
					thread2抢到了锁
					thread2唤醒其它等待的线程
					thread1恢复执行了5
					=

- 共享式

	######获取锁
	调用acquireShared(arg)入口方法
	进入tryAcquireShared(arg)模版方法获取同步状态，如果返回值>=0，则说明同步状态state=0，获取锁成功直接返回
	如果tryAcquireShared(arg)返回值<0，说明获取同步状态失败，向队列尾部添加一个共享类型的Node节点，随即该节点进入自旋状态
	自旋时，首先检查前驱节点释放为头节点&tryAcquireShared()是否>=0(即成功获取同步状态)
	如果是，则说明当前节点可执行，同时把当前节点设置为头节点，并且唤醒所有后继节点  
	如果否，则利用LockSupport.unpark(this)挂起当前线程，等待被前驱节点唤醒
	
	######释放锁
	调用releaseShared(arg)模版方法释放同步状态
	如果释放成功，则遍历整个队列，利用LockSupport.unpark(nextNode.thread)唤醒所有后继节点 

	- CountDownLatch计数器

		内部维护一个计数器，发生一个事件后，调用countDown方法，计数器减1，await用于等待计数器为0后继续执行当前线程

		- await()

			- 线程会被挂起，它会等待直到count值为0才继续执行

				- 基于tryAcquireShared()

		- countDown()

			- 将count值减1

				- 基于tryReleaseShared()

		- 底层原理

			- 通过AQS里面的共享锁来实现

		- 应用场景

			1.实现多个线程开始执行任务的最大并行性。注意是并行性，不是并发，强调的是多个线程在某一时刻同时开始执行。类似于赛跑，将多个线程放到起点，等待发令枪响，然后同时开跑。做法是初始化一个共享的 CountDownLatch 对象，将其计数器初始化为 1 ：new CountDownLatch(1) ，多个线程在开始执行任务前首先 coundownlatch.await()，当主线程调用 countDown() 时，计数器变为0，多个线程同时被唤醒。
			
			2.全员各就位才能进行下一步的场景
			
			3.启动一个服务时，主线程需要等待多个组件加载完毕，之后再继续执行

		- demo

			public class Test {  
			     public static void main(String[] args) {     
			         final CountDownLatch latch = new CountDownLatch(2);  
			            
			         new Thread(){  
			             public void run() {  
			                 try {  
			                     System.out.println("子线程"+Thread.currentThread().getName()+"正在执行");
			                    Thread.sleep(3000);  
			                    System.out.println("子线程"+Thread.currentThread().getName()+"执行完毕");
			                    latch.countDown();  
			                } catch (InterruptedException e) {  
			                    e.printStackTrace();  
			                }  
			             };  
			         }.start();  
			            
			         new Thread(){  
			             public void run() {  
			                 try {  
			                     System.out.println("子线程"+Thread.currentThread().getName()+"正在执行");
			                     Thread.sleep(3000);  
			                     System.out.println("子线程"+Thread.currentThread().getName()+"执行完毕");
			                     latch.countDown();  
			                } catch (InterruptedException e) {  
			                    e.printStackTrace();  
			                }  
			             };  
			         }.start();  
			            
			         try {  
			             System.out.println("等待2个子线程执行完毕...");
			            latch.await();  
			            System.out.println("2个子线程已经执行完毕");
			            System.out.println("继续执行主线程");
			        } catch (InterruptedException e) {  
			            e.printStackTrace();  
			        }  
			     }  
			}  
			
			线程Thread-0正在执行
			线程Thread-1正在执行
			等待2个子线程执行完毕...
			线程Thread-0执行完毕
			线程Thread-1执行完毕
			2个子线程已经执行完毕
			继续执行主线程

	- CyclicBarrier同步屏障

		public class CyclicBarrier {  
		    // 栅栏的年代/年龄
		    private static class Generation {  
		        boolean broken = false;  
		    }  
		    // CyclicBarrier基于ReentrantLock实现
		    private final ReentrantLock lock = new ReentrantLock();  
		    // lock上的Condition条件,当线程未全部就位时,到达栅栏的线程将被添加到该条件队列
		    private final Condition trip = lock.newCondition();  
		    // 代表线程的数量,在构造时传入,当前就位线程数(count)==parties时,栅栏打开
		    private final int parties;  
		    // 当前就位的线程数.当count==parties时,栅栏打开
		    private int count  
		    // 在所有线程就位之后且未被唤醒期间执行
		    private final Runnable barrierCommand;  
		    // 当线程全部就位,栅栏每次打开 或者 栅栏重置reset后,年代改变
		    private Generation generation = new Generation();  
		    ...  
		}

		- await()

			- 让出CPU资源，同时唤醒其他线程，基于condition

		- 底层原理

			基于ReentrantLockh和Condition实现，每次调用await()执行--count==0(相当于state)，满足条件就调用Condition.singal()唤醒其他线程，否则Condition.await()继续睡觉。
			跟CountDownLatch的不同在于，它可以重用

		- demo

			public class Test {  
			    public static void main(String[] args) {  
			        int N = 4;  
			        CyclicBarrier barrier  = new CyclicBarrier(N);  
			        for(int i=0;i<N;i++)  
			            new Writer(barrier).start();  
			    }  
			    static class Writer extends Thread{  
			        private CyclicBarrier cyclicBarrier;  
			        public Writer(CyclicBarrier cyclicBarrier) {  
			            this.cyclicBarrier = cyclicBarrier;  
			        }  
			   
			        @Override  
			        public void run() {  
			            System.out.println("线程"+Thread.currentThread().getName()+"正在写入数据...");
			            try {  
			                Thread.sleep(5000);      //以睡眠来模拟写入数据操作
			                System.out.println("线程"+Thread.currentThread().getName()+"写入数据完毕，等待其他线程写入完毕");
			                cyclicBarrier.await();  
			            } catch (InterruptedException e) {  
			                e.printStackTrace();  
			            }catch(BrokenBarrierException e){  
			                e.printStackTrace();  
			            }  
			            System.out.println("所有线程写入完毕，继续处理其他任务...");
			        }  
			    }  
			}  
			
			线程Thread-0正在写入数据...
			线程Thread-3正在写入数据...
			线程Thread-2正在写入数据...
			线程Thread-1正在写入数据...
			线程Thread-2写入数据完毕，等待其他线程写入完毕
			线程Thread-0写入数据完毕，等待其他线程写入完毕
			线程Thread-3写入数据完毕，等待其他线程写入完毕
			线程Thread-1写入数据完毕，等待其他线程写入完毕
			所有线程写入完毕，继续处理其他任务...
			所有线程写入完毕，继续处理其他任务...
			所有线程写入完毕，继续处理其他任务...
			所有线程写入完毕，继续处理其他任务...

	- Semaphore信号量

		内部维护了一个计数器，其值为可以访问的共享资源的个数。一个线程要访问共享资源，先获得信号量，如果信号量的计数器值大于1，意味着有共享资源可以访问，则使其计数器值减去1，再访问共享资源。如果计数器值为0，线程进入休眠。当某个线程使用完共享资源后，释放信号量，并将信号量内部的计数器加1，之前进入休眠的线程将被唤醒并再次试图获得信号量

		- acquire()

			自旋获取锁资格，在state > 0时获取成功， state < 0加入阻塞队列

			- 获取一个许可，基于CAS，state-1

		- release()

			- 释放一个许可，基于CAS，state+1

		- 底层原理

			- CAS操作加无限循环的算法，用来保证共享变量的正确性

		- 应用场景

			- 长隆欢乐世界排队坐过山车

		- demo

			public class TestSemaphore {  
			    public static void main(String[] args) {  
					ExecutorService exec = Executors.newCachedThreadPool();  
					final Semaphore semp = new Semaphore(5);        // 只能5个线程同时访问
					for (int index = 0; index < 10; index++) {      // 模拟10个客户端访问
					    final int NO = index;  
					    Runnable run = new Runnable() {  
					        public void run() {  
					            try {  
					                semp.acquire();                 // 获取许可
					                System.out.println("Accessing: " + NO);  
					                Thread.sleep((long) (Math.random() * 10000));  
					                semp.release();                 // 访问完后，释放
					                System.out.println("-----------------"+semp.availablePermits());  
					        	} catch (InterruptedException e) {  
					                e.printStackTrace();  
					        	}  
					   		}  
						};  
						exec.execute(run);  
			        }  
			     exec.shutdown();                                   // 退出线程池
			    }  
			}   
			
			执行结果如下：  
			Accessing: 0  
			Accessing: 1  
			Accessing: 3  
			Accessing: 4  
			Accessing: 2  
			-----------------0  
			Accessing: 6  
			-----------------1  
			Accessing: 7  
			-----------------1  
			Accessing: 8  
			-----------------1  
			Accessing: 10  
			-----------------1  
			Accessing: 9  
			-----------------1  
			Accessing: 5

- CAS

	CAS算法：总共由三个操作数，一个内存值v，一个线程本地内存旧值a（期望操作前的值）和一个新值b，在操作期间先拿旧值a和内存值v比较有没有发生变化，如果没有发生变化，才能内存值v更新成新值b，发生了变化则不交换。循环CAS算法则是不停的执行CAS操作。一般情况下，“更新”是一个不断重试的操作

	- ABA问题

		小明在提款机，提取了50元，因为提款机问题，有两个线程，同时把余额从100变为50线程1（提款机）：获取当前值100，期望更新为50，线程2（提款机）：获取当前值100，期望更新为50，线程1成功执行，线程2某种原因block了，这时，某人给小明汇款50线程3（默认）：获取当前值50，期望更新为100，这时候线程3成功执行，余额变为100，线程2从Block中恢复，获取到的也是100，compare之后，继续更新余额为50！！！此时可以看到，实际余额应该为100（100-50+50），但是实际上变为了50（100-50+50-50）这就是ABA问题带来的成功提交

		- 在变量前面加上版本号，每次变量更新的时候变量的版本号都+1，即A->B->A就变成了1A->2B->3A

	- 循环时间长开销大

	- 只能保证一个共享变量的原子操作

		对一个共享变量执行操作时，CAS能够保证原子操作，但是对多个共享变量操作时，CAS是无法保证操作的原子性的。
		
		Java从1.5开始JDK提供了AtomicReference类来保证引用对象之间的原子性，可以把多个变量放在一个对象里来进行CAS操作

	- AtomicReference支持对象级别的原子操作

	- CAS线程空转问题

		JVM有个类longAdder，它是一个数组，存储的是字节0/1。每个线程到不同的下标修改元素值，改0或者1，改完就溜。计算元素值的时候就把数组所有元素相加，这样就避免了多个线程同时对一个元素修改，只有一个线程能成功，其他线程空转做无用功的问题。将被修改的元素拆分成多个元素。

### 阻塞队列

- 非阻塞队列常用方法

	- add(E e)

		将元素e插入到队列末尾，如果插入成功，则返回true；如果插入失败（即队列已满），则会抛出异常

	- remove()

		移除队首元素，若移除成功，则返回true；如果移除失败（队列为空），则会抛出异常

	- offer(E e)

		将元素e插入到队列末尾，如果插入成功，则返回true；如果插入失败（即队列已满），则返回false

	- peek()

		获取队首元素，若成功，则返回队首元素；否则返回null

	- poll()

		移除并获取队首元素，若成功，则返回队首元素；否则返回null

- 阻塞队列常用方法

	- put(E e)

		向队尾存入元素，如果队列满，则等待

	- take()

		从队首取元素，如果队列为空，则等待

	- offer(E e, long timeout, TimeUnit unit)

		向队尾存入元素，如果队列满，则等待一定的时间，当时间期限达到时，如果还没有插入成功，则返回false；否则返回true

	- poll(long timeout, TimeUnit unit)

		从队首取元素，如果队列空，则等待一定的时间，当时间期限达到时，如果取到，则返回null；否则返回取得的元素

- 常用队列

	- ArrayBlockingQueue

		- 基于数组，有界阻塞队列，默认非公平锁

	- LinkedBlockingQueue

		- 基于链表，有界阻塞队列，容量Integer.MAX_VALUE

	- PriorityBlockingQueue

		- 基于链表，无界阻塞队列，默认 使用字典对元素进行优先级排序

	- DelayQueue

		DelayQueue是一个支持延时获取元素的无界阻塞队列。队列使用PriorityQueue来实现。队列中的元素必须实现Delayed接口，在创建元素时可以指定多久才能从队列中获取当前元素。只有在延迟期满时才能从队列中提取元素。我们可以将DelayQueue运用在以下应用场景：  
		
		缓存系统的设计：可以用DelayQueue保存缓存元素的有效期，使用一个线程循环查询DelayQueue，一旦能从DelayQueue中获取元素时，表示缓存有效期到了。  
		定时任务调度。使用DelayQueue保存当天将会执行的任务和执行时间，一旦从DelayQueue中获取到任务就开始执行，从比如TimerQueue就是使用DelayQueue实现的

		- 基于PriorityBlockingQueue、无界阻塞队列。元素需要实现compareTo和getDelay方法，基于最小堆实现排序

	- SynchronousQueue

		- 一个不存储元素的阻塞队列，每一个put操作必须等待一个take操作

	- LinkedTransferQueue

		一个由链表结构组成的无界阻塞队列

		- tryTransfer()

			则是用来试探下生产者传入的元素是否能直接传给消费者。如果没有消费者等待接收元素，则返回false。和transfer方法的区别是tryTransfer方法无论消费者是否接收，方法立即返回。而transfer方法是必须等到消费者消费了才返回

		- transfer()

			如果当前有消费者正在等待接收元素（消费者使用take()方法或带时间限制的poll()方法时），transfer方法可以把生产者传入的元素立刻transfer（传输）给消费者。如果没有消费者在等待接收元素，transfer方法会将元素存放在队列的tail节点，并等到该元素被消费者消费了才返回

	- LinkedBlockingDeque

		- 一个由链表结构组成的双向阻塞队列。初始化LinkedBlockingDeque时可以初始化队列的容量，用来防止其再扩容时过渡膨胀。另外双向阻塞队列可以运用在“工作窃取”模式中

### happens-before规则

1.程序次序规则：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作
2.锁定规则：一个unLock操作先行发生于后面对同一个锁额lock操作  
3.volatile变量规则：对一个变量的写操作先行发生于后面对这个变量的读操作  
4.传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C  
5.线程启动规则：Thread对象的start()方法先行发生于此线程的每个一个动作  
6.线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生  
7.线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行  
8.对象终结规则：一个对象的初始化完成先行发生于他的finalize()方法的开始

### 多线程

- 实现多线程

	- 实现Runnable接口

		public class MyThread extends OtherClass implements Runnable {    
		　　public void run() {  
		　　 System.out.println("MyThread.run()");  
		　　}  
		}    
		MyThread myThread = new MyThread();    
		Thread thread = new Thread(myThread);    
		thread.start(); 

	- 实现Callable接口

		public class SomeCallable<V> extends OtherClass implements Callable<V> {  
		
		    @Override  
		    public V call() throws Exception {  
		        // TODO Auto-generated method stub  
		        return null;  
		    }  
		
		}  
		Callable<V> oneCallable = new SomeCallable<V>();     
		//由Callable<Integer>创建一个FutureTask<Integer>对象：   
		FutureTask<V> oneTask = new FutureTask<V>(oneCallable);     
		//注释：FutureTask<Integer>是一个包装器，它通过接受Callable<Integer>来创建，它同时实现了Future和Runnable接口。 
		//由FutureTask<Integer>创建一个Thread对象：   
		Thread oneThread = new Thread(oneTask);     
		oneThread.start();     
		//至此，一个线程就创建完成了。

	- 继承Thread类

		public class MyThread extends Thread {    
		　　public void run() {  
		　　 System.out.println("MyThread.run()");  
		　　}  
		}    
		   
		MyThread myThread1 = new MyThread();    
		MyThread myThread2 = new MyThread();    
		myThread1.start();    
		myThread2.start();

	- 拓展：使用ExecutorService

- 线程通信

	- 消息传递的并发模型

		- wait() + notify()/notifyAll()

		- await() + signal()/signalAll()

	- 共享内存并发的模型

		- 同步锁

- Executor框架

	- ExecutorService

		- execute()/submie()区别

			1.从接受类型上：execute只能接受Runnable类型的任务；submit不管是Runnable还是Callable类型的任务都可以接受，但是Runnable返回值均为void，所以使用Future的get()获得的还是null  
			
			2.从返回值上：execute没有返回值；submit有返回值，所以需要返回值的时候必须使用submit  
			
			3.从异常处理上：execute中的是Runnable接口的实现，所以只能使用try、catch来捕获CheckedException；submit不管提交的是Runnable还是Callable类型的任务，如果不对返回值Future调用get()方法，都会吃掉异常

		- shutdown()&shutdownNow()

			关闭&立即关闭

		- ThreadPoolExecutor

			- 静态子类

				- newFixedThreadPool

					public static ExecutorService newFixedThreadPool(int nThreads) {  
					    return new ThreadPoolExecutor(nThreads, nThreads,  
					                                  0L, TimeUnit.MILLISECONDS,  
					                                  new LinkedBlockingQueue<Runnable>());  
					}

					- 创建的线程池corePoolSize和maximumPoolSize值是相等的，使用LinkedBlockingQueue，也叫长度固定线程池

				- newCachedThreadPool

					public static ExecutorService newCachedThreadPool() {  
					    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,  
					                                  60L, TimeUnit.SECONDS,  
					                                  new SynchronousQueue<Runnable>());  
					}

					- 将corePoolSize设置为0，将maximumPoolSize设置为Integer.MAX_VALUE，使用的SynchronousQueue，来任务就创建线程运行，当线程空闲超过60秒，就销毁线程

				- newScheduledThreadPool

					- 创建一个定长线程池，支持定时及周期性任务执行

				- newSingleThreadExecutor

					public static ExecutorService newSingleThreadExecutor() {  
					    return new FinalizableDelegatedExecutorService  
					        (new ThreadPoolExecutor(1, 1,  
					                                0L, TimeUnit.MILLISECONDS,  
					                                new LinkedBlockingQueue<Runnable>()));  
					}

					- 创建的线程池corePoolSize和maximumPoolSize值都为1，使用LinkedBlockingQueue

			- 核心参数

				- corePoolSize：核心池的大小

					默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中。或者调用了prestartAllCoreThreads()或者prestartCoreThread()预先创建线程：

					- poolSize：线程池中当前线程的数量

				- maximumPoolSize：线程池最大线程数

					线程池最大线程数，表示在线程池中最多能创建多少个线程

					- largestPoolSize：曾经出现过最大线程数

				- 四者区别

					提交一个任务时：  
					1.poolSize<corePoolSize，新增加一个线程处理新的任务。
					2.poolSize=corePoolSize，新任务会被放入阻塞队列等待。
					3.阻塞队列的容量达到上限，且这时poolSize<maximumPoolSize，新增线程来处理任务。
					4.阻塞队列满了，且poolSize=maximumPoolSize，那么线程池已经达到极限，会根据饱和策略RejectedExecutionHandler拒绝新的任务

					- 

				- threadFactory：线程工厂

				- keepAliveTime：存活时间

					表示线程没有任务执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用，直到线程池中的线程数不大于corePoolSize，即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。但是如果调用了allowCoreThreadTimeOut(boolean)方法，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的线程数为0

				- unit：keepAliveTime的时间单位

					TimeUnit.DAYS;   TimeUnit.HOURS;        
					TimeUnit.MINUTES;    
					TimeUnit.SECONDS;   
					TimeUnit.MILLISECONDS; TimeUnit.MICROSECONDS;  TimeUnit.NANOSECONDS; 

				- workQueue：阻塞队列

				- RejectedExecutionHandler：拒绝处理任务策略

					ThreadPoolExecutor.AbortPolicy：丢弃任务并抛出RejectedExecutionException异常。 
					ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。 
					ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
					ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务 

- ForkJoinPool

	- 工作窃取+双端队列

	- 架构

		- ForkJoinTask

			代表运行在ForkJoinPool中的任务

			- RecursiveTask:用于有返回结果的任务

			- RecursiveAction：用于没有返回结果的任务

		- ForkJoinPool

			执行ForkJoinTask任务

			- ForkJoinTask数组：存放任务

			- ForkJoinWorkerThread：执行任务

	- api

		- fork()

			在当前线程运行的线程池中安排一个异步执行。简单的理解就是再创建一个子任务

		- join()

			当任务完成的时候返回计算结果

		- invoke()

			开始执行任务，如果必要，等待计算完成

## 类

### 类加载

- 类加载器

	- Bootstrap ClassLoader

		- 随着JVM启动，首先依托启动类加载器去加载Java安装目录下的“lib”目录中的核心类库

	- Extension ClassLoader

		- 随着JVM启动，从Java安装目录下，加载这个“lib\ext”目录中的类

	- Application ClassLoader

		- 负责加载“ClassPath”环境变量所指定的路径中的类，相当于自己写的类

	- Customize ClassLoader

		- 自定义类加载器

	- 双亲委派机制

		如果一个类加载器收到了类加载的请求，它首先不会自己去加载这个类，而是把这个请求委派给父类加载器去完成，每一层的类加载器都是如此，这样所有的加载请求都会被传送到顶层的启动类加载器中，只有当父加载无法完成加载请求（它的搜索范围中没找到所需的类）时，子加载器才会尝试去加载类

		- 避免多层级的加载器结构重复加载某些类

		- 继承类加载顺序

			执行父类静态代码 
			执行子类静态代码  
			初始化父类成员变量  
			初始化父类构造函数  
			初始化子类成员变量  
			初始化子类构造函数

- 类加载过程

	- 加载

		- 类加载时机

			1.预加载：在虚拟机启动的时候加载，加载的是JAVA_HOME/lib/下的rt.class下的.class文件，是java程序运行时经常要用到的一些类，比如java.lang.⁎以及 java.util.⁎等
			
			2.运行时加载：虚拟机在用到一个.class文件时，首先会去内存中查找这个.class文件有没有被加载，没有被加载会根据这个类的全限定名去加载。

		- 在代码使用到类时，会从.class文件加载到JVM内存，顺便执行静态代码

			- 如果代码包含main函数，JVM进程启动之后，就加载到内存，并执行这段代码逻辑

	- 验证

		- 验证加载进来的.class文件是否符合JVM规范

	- 准备

		- 为成员变量开辟内存空间

	- 解析

		- 符号引用替换为直接引用

	- 初始化

		- new的时候，或者执行mian函数的时候会初始化，给成员变量赋值

	- 使用

	- 卸载

		- 只有满足三个条件的类，才能被GC回收，也就是卸载(unload)类

			- 该类所有的实例都已经被GC，也就是JVM中不存在该Class的任何实例

			- 加载该类的类加载器已经被GC

			- 该类的java.lang.Class对象没有在任何地方被引用，如不能在任何地方通过反射访问该类的方法

### Class类

- 基本性质

	- Class类的对象不能以new Shapes()的方式创建，它的对象只能由JVM创建，因为这个类没有public构造函数

	- Class类的作用是运行时提供或获得某个对象的类型信息

	- Class类的对象内容是你创建的类的类型信息，比如你创建一个Shapes类，那么，Java会生成一个内容是Shapes的Class类的对象

- 获得它

	- Class.forName(“Shapes");

		- 拓展：newInstance

			1.必须保证类已经被加载
			
			2.弱类型,低效率,只能调用无参构造

	- 调用getClass()

	- String.class

## IO

### 零拷贝

- FileChannel.transferTo

### BIO

- 步步阻塞IO

### NIO

- 不阻塞IO

### AIO

- 异步不阻塞IO

## jdk面试题

### HashMap1.7&1.8差异

1.1.8比1.7多了个链条长度超过8便转红黑树

2.1.7是头插，1.8是尾插  

3.1.7实体是Entry<k, v>，1.8实体是实现了Map.Entry的Node<k, v>，和1.7的Entry<k, v>等价  

4.1.8对于resize在获取新下标的时候做了优化，如果新增与运算为0，元素位置保持不变，如果新增与运算为1，元素新位置=原始位置+原容量

### [ConcurrentHashMap1.7&1.8差异](https://www.jianshu.com/p/e694f1e868ec)

1.1.7采用Segment + HashEntry + ReentrantLock，1.8使用Node + CAS + Synchronized

2.put:  
1.7：先定位Segment，再定位桶，put全程加锁，没有获取锁的线程提前找桶的位置，并最多自旋64次获取锁，超过则挂起。  

1.8：由于移除了Segment，类似HashMap，可以直接定位到桶，拿到first节点后进行判断，1、为空则CAS插入；2、为-1则说明在扩容，则跟着一起扩容；3、else则加锁put（类似1.7）  

3.size:  
1.7：很经典的思路：计算两次，如果不变则返回计算结果，若不一致，则锁住所有的Segment求和。  

1.8：用baseCount来存储当前的节点个数，这就设计到baseCount并发环境下修改的问题（LongAdder）  

4.resize:  
1.7：跟HashMap步骤一样，只不过是搬到单线程中执行，避免了HashMap在1.7中扩容时死循环的问题，保证线程安全。  

1.8：支持并发扩容，HashMap扩容在1.8中由头插改为尾插（为了避免死循环问题），ConcurrentHashmap也是，迁移也是从尾部开始，扩容前在桶的头部放置一个hash值为-1的节点，这样别的线程访问时就能判断是否该桶已经被其他线程处理过了。

- LongAdder

	1.一个baseCount和一个数组
	
	2.没有竞争的时候直接更新baseCount
	
	3.有竞争的情况下，hash线程到数组，如何通过cas更新数组所在元素  
	
	4.求和时统计baseCount和数组值，此过程不加锁，所以如果有数据在更新可能出现数据不准确

- 扩容

	触发扩容：  
	1. 添加新元素后，元素个数达到扩容阈值触发扩容。
	
	2. 某个槽内的链表长度达到 8，但是数组长度小于 64 时候触发扩容。
	
	扩容：  
	首先每个线程承担不小于 16 个槽中的元素的扩容，然后从右向左划分 16 个槽给当前线程去迁移，每当开始迁移一个槽中的元素的时候，线程会锁住当前槽中列表的头元素，假设这时候正好有 get 请求过来会仍旧在旧的列表中访问，如果是插入、修改、删除、合并、compute 等操作时遇到 ForwardingNode，当前线程会加入扩容大军帮忙一起扩容，扩容结束后再做元素的更新操作

### [threadlocal内存泄漏问题](https://www.jianshu.com/p/48b9e1c1d555)

0.什么是弱引用：只具有弱引用的对象拥有短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存

1.内存泄漏的原因：当前线程threadLocalMap持有threadLocal弱引用作为key，当threadLocal设置为null时，线程内部threadLocalMap的key在GC之后不可访问，但value还没设置为null，占据内存

2.为什么不用强引用：当threadLocal设置为null，但线程内部threadLocalMap的key持有threadLocal的引用，除非线程死亡，GC无法回收  

3.为什么用弱引用：引用的threadLocal的对象被回收了，由于threadLocalMap持有threadLocal的弱引用，即使没有手动删除，threadLocal也会被回收。value在下一次threadLocalMap调用set, get，remove的时候会被清除（只要threadLocal没被回收，因为threadLocal对象是强引用）  

4.最佳实践  

每次使用完ThreadLocal，都调用它的remove()方法，清除数据。  

在使用线程池的情况下，没有及时清理ThreadLocal，不仅是内存泄漏的问题，更严重的是可能导致业务逻辑出现问题。所以，使用ThreadLocal就跟加锁完要解锁一样，用完就清理。

- ThreadLocal ，ThreadLocalMap 和Thread 的关系

	Thread内部维护一个ThreadLocalMap，ThreadLocalMap是一个数组，其key是ThreadLocal

- 四种引用

	1.强引用：Object obj = new Object
	内存不足时及时抛出异常也不会回收。如果强引用定义在方法内，随着方法调用结束弹栈，强引用不可达，然后被回收  
	
	2.软引用：SoftReference<String> softReference = new SoftReference<String>(“foo”)  
	当内存空间充足的时候，不会回收这部分内存；当内存空间不足时，GC会回收软引用。。可用来实现内存敏感的高速缓存  
	
	3.弱引用：WeakReference<String> weakReference = new WeakReference<>(“foo”)  
	不管内存状态，只有GC发现弱引用就回收  
	
	4.虚引用：  
	
	引用类型    被垃圾回收时间    用途           			生存时间  
	强引用       从来不会             对象的一般状态    		JVM停止运行时终止  
	软引用       当内存不足时       对象缓存      			内存不足时终止  
	弱引用       正常垃圾回收时    对象缓存       			垃圾回收后终止  
	虚引用       正常垃圾回收时    跟踪对象的垃圾回收	垃圾回收后终止

- 应用

	1.SimpleDateFormate线程不安全
	
	2.传递参数时，跨越的方法过多

### sleep和wait区别

1.sleep是静态方法，使当前线程睡眠一定时间，不让出锁。当睡眠结束便会进入就绪状态，等待CPU资源。
2.wait是实例方法，让出锁资源，只能在同步代码块内执行，只有调用notify/notifyall才能被唤醒。

### 线程状态

1..初始(NEW)：新创建了一个线程对象，但还没有调用start()方法。
2. 运行(RUNNABLE)：Java线程中将就绪（ready）和运行中（running）两种状态笼统的称为“运行”。
线程对象创建后，其他线程(比如main线程）调用了该对象的start()方法。该状态的线程位于可运行线程池中，等待被线程调度选中，获取CPU的使用权，此时处于就绪状态（ready）。就绪状态的线程在获得CPU时间片后变为运行中状态（running）。
3. 阻塞(BLOCKED)：表示线程阻塞于锁。
4. 等待(WAITING)：进入该状态的线程需要等待其他线程做出一些特定动作（通知或中断）。
5. 超时等待(TIMED_WAITING)：该状态不同于WAITING，它可以在指定的时间后自行返回。
6. 终止(TERMINATED)：表示该线程已经执行完毕。

- join()

	Join方法内部是wait()方法，被synchronized修饰，锁住的对象是this。在其他线程唤醒的时候会唤醒这类线程（基于JVM底层）

### interface和abstract区别

1.abstract是类，只能单继承；interface是接口，可以多实现。
2.abstract修饰的方法不能有final，private，static，因为这些关键字不能被重写；interface的方法都是public，且不能有具体实现，除了default方法  
3.有abstract方法的类就是abstract类，需要用abstract修饰类定义。abstract修饰的方法没有方法体，如果子类实现了所有的abstract方法，就可以实例该子类，不然子类依然是abstract类，不能被实例

### 拦截器和过滤器的区别

1.拦截器是基于java反射机制的，而过滤器是基于函数回调。
2.拦截器不依赖于Servlet容器，而过滤器依赖于servlet容器。
3.拦截器只能对action请求起作用，而过滤器可以对几乎所以的请求起作用。
4.拦截器可以访问action上下文，值栈里的对象，而过滤器不能。
5.在Action的生命周期周，拦截器可以被多次调用，而过滤器只能在容器初始化的时候被调用一次	

### 重写和重载的区别

## 核心概念

### 容器

- [treeMap](https://www.toutiao.com/i6753185294389871115/)

- [hashmap](https://mp.weixin.qq.com/s?__biz=MzIxMjE5MTE1Nw==&mid=2653191907&idx=1&sn=876860c5a9a6710ead5dd8de37403ffc&chksm=8c990c39bbee852f71c9dfc587fd70d10b0eab1cca17123c0a68bf1e16d46d71717712b91509&scene=21#wechat_redirect)

	第二篇：https://juejin.im/post/5a224e1551882535c56cb940

	- 线程不安全的原因resize()

- [linkedHashMao](https://juejin.im/post/5a4b433b6fb9a0451705916f)

	- LRU缓存

- [concurrentHashMap](https://www.toutiao.com/i6768455461000708611/)

### 关键字

- final关键字

	1.修饰普通变量：初始化之后不能重新赋值
	2.修饰类变量：初始化之后不能指向其他对象，但可以改变其成员变量，值得一提的是，final修饰的String初始化之后，编译器会把它当作常量使用
	3.修饰类：不可被继承，并且所有方法均为final方法
	4.修饰方法：方法不可被重写，类的private方法被final隐式修饰

	- final，finally，finalize，system.gc区别

		final 用于声明属性,方法和类, 分别表示属性不可变, 方法不可覆盖, 类不可继承.
		
		finally 是异常处理语句结构的一部分，表示总是执行.
		
		finalize 是Object类的一个方法，在垃圾收集器执行的时候会调用被回收对象的此方法来执行资源清理。可以覆盖此方法提供垃圾收集时的其他资源回收，例如关闭文件等. JVM不保证此方法总被调用。（不建议这么做）
		
		system.gc建议JVM回收垃圾，只是建议

- static关键字

	1.修饰方法：没什么好说的
	2.修饰类：没什么好说的
	3.修饰变量：没什么好说的  
	4.修饰方法块：实例类的时候执行该方法块

- abstract关键字

	1.修饰类：不可被实力化，同时所有方法均是抽象方法
	2.修饰方法：抽象方法所在类必然是抽象类  
	3.应用：模版方法设计模式：当功能内部一部分实现是确定的，一部分实现是不确定的，可以把不确定的部分暴露出去，让子类去实现

- synchronized关键字

- 修饰符关键字

## JDK1.8新特性

### map操作

- toMap

- countBy

### Lambda

- () -> new Employee()

- (x) -> System.out.println(x)

- (x, y) -> Integer.compare(x, y)

- (x) -> new String[x]

### 函数式编程

- 方法引用

	- 对象::实例方法

		Consumer<Integer> con = (x) -> System.out.println(x)  
		————————————————  
		Consumer<Integer> con2 = System.out::println;  
		con2.accept(200);

	- 类名::实例方法名

		BiFunction<Integer, Integer, Integer> biFun = (x, y) -> Integer.compare(x, y);  
		————————————————  
		BiFunction<Integer, Integer, Integer> biFun = (x, y) -> Integer.compare(x, y);  
		Integer result = biFun2.apply(100, 200);

	- 类名::静态方法名

		BiFunction<String, String, Boolean> fun1 = (str1, str2) -> str1.equals(str2);  
		————————————————  
		BiFunction<Integer, Integer, Integer> biFun2 = Integer:compare  
		Integer result = biFun2.apply(100, 200);

- 构造器引用

	- className::new

		Supplier<Employee> sup = () -> new Employee();  
		————————————————  
		Supplier<Employee> sup2 = Employee::new;

- 数组引用

	- Type[]::new

		Function<Integer, String[]> fun = (x) -> new String[x];  
		————————————————  
		Function<Integer, String[]> fun2 = String[]::new;  
		String[] strArray = fun2.apply(10);

### 函数式接口

仅仅只包含一个抽象方法的接口  
public class FunctionalInterfaceMain {  
	public static void main(String[] args) {  
		Function<String,String> function1 = item -> item +"返回值";  
		Consumer<String> function2 = iterm -> {System.out.println(iterm);};  
		Predicate<String> function3 = iterm -> "".equals(iterm);  
		Supplier<String> function4 = () -> new String("");  
		List<String> list = Arrays.asList("zhangsan","lisi","wangwu","xiaoming","zhaoliu");  
		list.stream()  
			.map(value -> value + "1") //传入的是一个Function函数式接口  
			.filter(value -> value.length() > 2) //传入的是一个Predicate函数式接口  
			.forEach(value -> System.out.println(value)); //传入的是一个Consumer函数式接口  
	}	  
}

- Consumer<T>：消费型接口，有参无返回值

	public void test(){  
	        changeStr("hello",(str) -> System.out.println(str));  
	}  
	
	    public void changeStr(String str, Consumer<String> con){  
	        con.accept(str);  
	}

	- BiConsumer<T , U>：接收T类型和U类型的两个参数，不返回值

	- andThen()

		- return (T t) -> { accept(t); after.accept(t); };

- Supplier<T>：供给型接口，无参有返回值

	public void test2(){  
	        String value = getValue(() -> "hello");  
	        System.out.println(value);  
	}  
	
	public String getValue(Supplier<String> sup){  
	        return sup.get();  
	}

- Function<T, R>：函数式接口，有参有返回值

	public void test3(){  
	        Long result = changeNum(100L, (x) -> x + 200L);  
	        System.out.println(result);  
	}  
	
	public Long changeNum(Long num, Function<Long, Long> fun){  
	        return fun.apply(num);  
	}

	- BiFunction<T, U, R>：接收T类型和U类型的两个参数，返回一个R类型的结果

	- 方法

		- andThen()

			- 先执行自身的apply()，该方法返回值作为参数apply()的输入参数

		- compose()

			- 接收一个Function类型的参数，返回一个值，内部也有一个apply()

			- 先执行参数的apply()，该方法返回值作为内部apply()的输入参数

- Predicate<T>： 断言型接口，有参有返回值，返回值是boolean类型

	public void test4(){  
	        boolean result = changeBoolean("hello", (str) -> str.length() > 5);  
	        System.out.println(result);  
	}  
	
	public boolean  changeBoolean(String str, Predicate<String> pre){  
	        return pre.test(str);  
	}

	- BiPredicate<T, U>：接收T类型和U类型的两个参数，返回一个boolean类型的结果

	- 方法

		- and

			- &&

		- or

			- ||

		- negate

			- !

- @FunctionalInterface

### [时间API](https://lw900925.github.io/java/java8-newtime-api.html)

- Period

	- 表示年月日时间段

- Instant

	- 表示时间戳

- Duration

	// 2017-01-05 10:07:00  
	LocalDateTime from = LocalDateTime.of(2017, Month.JANUARY, 5, 10, 7, 0);  
	
	// 2017-02-05 10:07:00  
	LocalDateTime to = LocalDateTime.of(2017, Month.FEBRUARY, 5, 10, 7, 0);  
	
	// 表示从 2017-01-05 10:07:00 到 2017-02-05 10:07:00 这段时间
	Duration duration = Duration.between(from, to);  
	
	// 这段时间的总天数
	long days = duration.toDays();  
	
	// 这段时间的小时数
	long hours = duration.toHours();  
	
	// 这段时间的分钟数
	long minutes = duration.toMinutes();  
	
	// 这段时间的秒数
	long seconds = duration.getSeconds();  
	
	// 这段时间的毫秒数
	long milliSeconds = duration.toMillis();  
	
	// 这段时间的纳秒数
	long nanoSeconds = duration.toNanos();

	- 表示时间段

- LocalDate

	 // 初始化一个日期：2017-01-04
	LocalDate localDate = LocalDate.of(2017, 1, 4);   
	
	// 年份：2017
	int year = localDate.getYear();   
	
	// 月份：JANUARY
	Month month = localDate.getMonth();                   
	
	// 月份中的第几天：4
	int dayOfMonth = localDate.getDayOfMonth();   
	
	 // 一周的第几天：WEDNESDAY
	DayOfWeek dayOfWeek = localDate.getDayOfWeek();  
	
	// 月份的天数：31
	int length = localDate.lengthOfMonth();        
	
	// 是否为闰年：false
	boolean leapYear = localDate.isLeapYear();  
	
	// 获取当前时间
	LocalDate now = LocalDate.now()

	- 表示一个具体的日期，但不包含具体时间，也不包含时区信息

- LocalTime

	// 初始化一个时间：17:23:52
	LocalTime localTime = LocalTime.of(17, 23, 52);  
	
	// 时：17
	int hour = localTime.getHour();  
	
	// 分：23
	int minute = localTime.getMinute();  
	
	// 秒：52
	int second = localTime.getSecond();

	- 和LocalDate类似，区别是LocalDate不包含具体时间，LocalTime包含具体时间

- LocalDateTime

	LocalDateTime ldt1 = LocalDateTime.of(2017, Month.JANUARY, 4, 17, 23, 52);  
	
	LocalDate localDate = LocalDate.of(2017, Month.JANUARY, 4);  
	
	LocalTime localTime = LocalTime.of(17, 23, 52);  
	
	LocalDateTime ldt2 = localDate.atTime(localTime);  
	
	向LocalDate和LocalTime转化：
	
	LocalDate date = ldt1.toLocalDate();  
	
	LocalTime time = ldt1.toLocalTime();

	- LocalDate和LocalTime的结合体，可以通过of()方法直接创建，也可以调用LocalDate的atTime()方法或LocalTime的atDate()方法将LocalDate或LocalTime合并成一个LocalDateTime

- 日期的操作和格式化

	- 增加和减少日期

		// 2017-01-05  
		LocalDate date = LocalDate.of(2017, 1, 5);  
		
		// 修改为 2016-01-05
		LocalDate date1 = date.withYear(2016);  
		
		// 修改为 2017-02-05
		LocalDate date2 = date.withMonth(2);  
		
		// 修改为 2017-01-01
		LocalDate date3 = date.withDayOfMonth(1);  
		
		// 增加一年 2018-01-05
		LocalDate date4 = date.plusYears(1);  
		
		// 减少两个月 2016-11-05
		LocalDate date5 = date.minusMonths(2);  
		
		// 增加5天 2017-01-10
		LocalDate date6 = date.plus(5, ChronoUnit.DAYS);  
		
		// 返回下一个距离当前时间最近的星期日
		LocalDate date7 = date.with(nextOrSame(DayOfWeek.SUNDAY));  
		
		// 返回本月最后一个星期六
		LocalDate date9 = date.with(lastInMonth(DayOfWeek.SATURDAY));

	- 格式化日期

		LocalDateTime dateTime = LocalDateTime.now();  
		
		// 20170105  
		String strDate1 = dateTime.format(DateTimeFormatter.BASIC_ISO_DATE);  
		
		// 2017-01-05  
		String strDate2 = dateTime.format(DateTimeFormatter.ISO_LOCAL_DATE);  
		
		// 14:20:16.998  
		String strDate3 = dateTime.format(DateTimeFormatter.ISO_LOCAL_TIME);  
		
		// 2017-01-05  
		String strDate4 = dateTime.format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));  
		
		// 今天是：2017年 一月 05日 星期四
		String strDate5 = dateTime.format(DateTimeFormatter.ofPattern("今天是：YYYY年 MMMM DD日 E", Locale.CHINESE));

	- 时区：ZoneId

		// 自定义时区
		ZoneId shanghaiZoneId = ZoneId.of("Asia/Shanghai");  
		
		// 获取默认时区  
		ZoneId systemZoneId = ZoneId.systemDefault();  
		
		// 获取时区
		ZoneOffset zoneOffset = ZoneOffset.of("+09:00");  
		
		LocalDateTime localDateTime = LocalDateTime.now();  
		
		OffsetDateTime offsetDateTime = OffsetDateTime.of(localDateTime, zoneOffset);

	- 获取昨天

		ZoneId zoneId = ZoneId.systemDefault();  
		Instant  instant = date.toInstant();  
		LocalDate   localDate = instant.atZone(zoneId).toLocalDate();  
		LocalTime localTime = LocalTime.of(0, 00, 00);  
		LocalDateTime localDateTime = LocalDateTime.of(localDate, localTime);  
		localDateTime = localDateTime.plusDays(-1);  
		instant = localDateTime.atZone(zoneId).toInstant();  
		return Date.from(instant);

### 并行流和串行流

## JVM内存模型

### 方法区：永生代是方法区的一个实现

1.7永生代存在方法区 
1.8 元空间metaspace取代永生区 字符串常量池移到堆
当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常

- 线程共享，存放被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等

- 运行时常量池

	- [字符串常量池](https://droidyue.com/blog/2014/12/21/string-literal-pool-in-java/)

		- intern()

			动态扩充常量池。当一个String实例str调用intern()方法时，jvm查找常量池中是否有相同Unicode的字符串常量，如果有，则返回其的引用，如果没有，则在常量池中增加一个Unicode等于str的字符串并返回它的引用
			
			public class HelloWorld {  
				public static void main(String[] args) {  
					// String s2 = "java1”; //return false if uncomment  
					String s1 = new StringBuilder().append("ja").append("va1").toString();  
					System.out.println(s1.intern() == s1); //true  
				}  
			}

	- [字符串拼接](https://droidyue.com/blog/2014/08/30/java-details-string-concatenation/)

	- String类

- 1.8后取消永生代，方法区作为概念上的区域仍然存在。原先永生代中类的元信息会被放入本地内存（元数据区，metaspace），将类的静态变量和内部字符串放入到java堆中

	- 元信息：程序元素如方法、字段、类和包上的额外信息

	- 由此可见，字符串常量存放的是字符串的引用，字符串对象存放在堆内存中，而如果基本数据类型声明为全局变量，那它也存在堆中

### 堆

所有线程共享，存放对象实例和数组。当堆无法扩展时，将会抛出OutOfMemoryError

- 新生代(刚new出来对象)

	- Eden区

		- 当Eden区没有足够空间进行分配时，虚拟机会发起一次Minor GC，Minor GC相比Major GC更频繁，回收速度也更快。通过Minor GC之后，Eden会被清空，Eden区中绝大部分对象会被回收，而那些无需回收的存活对象，将会进到Survivor的 From区(若From区不够，则直接进入Old区)

	- 16次Minor GC还能在新生代中存活的对象，才会被送到老年代

	- Survivor区

		- 每次Minor GC，会将之前Eden区和From区中的存活对象复制到To区域（如果To区不够，则直接进入Old区）。第二次Minor GC时，From与To 职责对换，这时候会将Eden区和To区中的存活对象再复制到From区域，以此反复

			- 防止内存碎片化，效率高

		- From区

		- To区

- 老年代

	- 直接进入老年代

		- 大对象直接进入

		- 长期存活对象

			- 对象在Survivor区中每经历一次 Minor GC，年龄就增加1岁。当年龄增加到15岁时，这时候就会被转移到老年代

### java虚拟机栈

线程私有，生命随着线程发展。为每个即将运行的java方法创建“栈帧(方法运行完毕弹出)”的区域，用于存放方法运行过程中所需要的一些信息：
1.局部变量表：存放编译器已知基本数据类型变量、对象引用
2.操作数栈
3.动态链接
4.方法出口信息等

StackOverflowError：线程请求的栈深度大于虚拟机所允许的深度  

OutOfMemoryError：动态扩展时无法申请到足够的内存

- 内存泄露/内存溢出

	- 常发性内存泄漏

		发生内存泄漏的代码会被多次执行到，每次被执行的时候都会导致一块内存泄漏

	- 偶发性内存泄漏

		发生内存泄漏的代码只有在某些特定环境或操作过程下才会发生。常发性和偶发性是相对的。对于特定的环境，偶发性的也许就变成了常发性的。所以测试环境和测试方法对检测内存泄漏至关重要

	- 一次性内存泄漏

		发生内存泄漏的代码只会被执行一次，或者由于算法上的缺陷，导致总会有一块仅且一块内存发生泄漏。比如，在类的构造函数中分配内存，在析构函数中却没有释放该内存，所以内存泄漏只会发生一次

	- 隐式内存泄漏

		程序在运行过程中不停的分配内存，但是直到结束的时候才释放内存。严格的说这里并没有发生内存泄漏，因为最终程序释放了所有申请的内存。但是对于一个服务器程序，需要运行几天，几周甚至几个月，不及时释放内存也可能导致最终耗尽系统的所有内存

	- 堆内存溢出

		public class HeapOutOfMemory {  
			public static void main(String[] args) {  
				List<TestCase> cases = new ArrayList<TestCase>();  
				while(true){  
					cases.add(new TestCase());  
				}  
			}  
		}  
		class TestCase{ }

		- 通过创建对象

	- 栈内存溢出

		public class StackOverFlow {  
			private int i ;  
			public void plus() {  
				i++;  
				plus();  
			}  
		public static void main(String[] args) {  
		 	StackOverFlow stackOverFlow = new StackOverFlow();  
		 	try {  
		 		stackOverFlow.plus();  
		 	} catch (Exception e) {  
		 		System.out.println("Exception:stack length:"+stackOverFlow.i);  
		 		e.printStackTrace();  
		 	} catch (Error e) {  
		 		System.out.println("Error:stack length:"+stackOverFlow.i);  
		 		e.printStackTrace();  
		 	}  
		 }  
		}

		- 通过方法调用

	- 常量池溢出

		public class ConstantOutOfMemory {   
			public static void main(String[] args) throws Exception {  
			 	try {List<String> strings = new ArrayList<String>();  
			 		int i = 0;  
			 		while(true){  
			 			strings.add(String.valueOf(i++).intern());  
			 		}  
			 	} catch (Exception e) {  
			 		e.printStackTrace();throw e;  
			 	}   
			 }   
		}

	- 方法区溢出

		public class MethodAreaOutOfMemory {  
			public static void main(String[] args) {  
				while(true){  
					Enhancer enhancer = new Enhancer();  
					enhancer.setSuperclass(TestCase.class);  
					enhancer.setUseCache(false);  
					enhancer.setCallback(new MethodInterceptor() {  
						@Override  
						public Object intercept(Object arg0, Method arg1, Object[] arg2,MethodProxy arg3) throws Throwable {  
							return arg3.invokeSuper(arg0, arg2);  
						}});  
					enhancer.create();  
				}}  
		}  
		class TestCase{ }

		- 通过创建类（动态代理或者CGLib）

### 程序计数器

- 记录着当前线程正在执行的那一条字节码指令的地址，但如果执行的是本地方法，计数器的值为空

- 每条线程私有，JVM中唯一不会出现OutOfMemoryError的区域。生命跟随线程发展

- 方便线程切换之后，能够知道自己执行到哪步了

### 本地方法栈

- 其跟java虚拟机栈差不多，只不过这块是运行本地方法的

### JVM参数

-Xms设置堆的最小空间大小。

-Xmx设置堆的最大空间大小。

-XX:NewSize设置新生代最小空间大小。

-XX:MaxNewSize设置新生代最大空间大小。

-XX:PermSize设置永久代最小空间大小。

-XX:MaxPermSize设置永久代最大空间大小。

-Xss设置每个线程的堆栈大小。

没有直接设置老年代的参数，但是可以设置堆空间大小和新生代空间大小两个参数来间接控制。  

老年代空间大小=堆空间大小-年轻代大空间大小

### 垃圾回收

- 定义垃圾

	- 引用计数算法

		对象头中分配一个空间来保存该对象被引用的次数，如果该对象被其它对象引用，则它的引用计数加1，如果删除对该对象的引用，那么它的引用计数就减1，当该对象的引用计数为0时，那么该对象就会被回收

		- 弃用原因是对象内部相互引用导致计数永远不为0

			public class ReferenceCountingGC {  
			 public Object instance;  
			 public ReferenceCountingGC(String name){}  
			}  
			public static void testGC(){  
			 ReferenceCountingGC a = new ReferenceCountingGC("objA");  
			 ReferenceCountingGC b = new ReferenceCountingGC("objB");  
			 a.instance = b;  
			 b.instance = a;  
			 a = null;  
			 b = null;  
			}

	- 可达性分析算法

		可达性分析算法的基本思路是，通过一些被称为引用链的对象作为起点，从这些节点开始向下搜索，搜索走过的路径被称为(Reference Chain)，当一个对象到GC Roots没有任何引用链相连时(即从 GC Roots 节点到该节点不可达)，则证明该对象是不可用的

		- GC Root对象

			所有Java线程当前活跃的栈帧里指向GC堆里的对象的引用；换句话说，当前所有正在被调用的方法的引用类型的参数/局部变量/临时值

			- 本地方法栈中JNI(即一般说的Native方法)引用的对象

				任何Native接口都会使用某种本地方法栈，实现的本地方法接口是使用C连接模型的话，那么它的本地方法栈就是C栈。当线程调用Java方法时，虚拟机会创建一个新的栈帧并压入Java栈。然而当它调用的是本地方法时，虚拟机会保持Java栈不变，不再在线程的Java栈中压入新的帧，虚拟机只是简单地动态连接并直接调用指定的本地方法

			- 虚拟机栈(栈帧中的本地变量表)中引用的对象

				public class StackLocalParameter {  
				 public StackLocalParameter(String name){}  
				}  
				public static void testGC(){  
				 StackLocalParameter s = new StackLocalParameter("localParameter");  
				 s = null;  
				}  
				
				此时的s，即为GC Root，当s置空时localParameter对象也断掉了与GC Root的引用链，将被回收

			- 方法区中类静态属性引用的对象

				public class MethodAreaStaicProperties {  
				 public static MethodAreaStaicProperties m;  
				 public MethodAreaStaicProperties(String name){}  
				}  
				public static void testGC(){  
				 MethodAreaStaicProperties s = new MethodAreaStaicProperties("properties");  
				 s.m = new MethodAreaStaicProperties("parameter");  
				 s = null;  
				}  
				
				s为GC Root，s置为null，经过GC后，s所指向的properties对象由于无法与GC Root建立关系被回收。
				而m作为类的静态属性，也属于GC Root，parameter对象依然与GC root建立着连接，所以此时parameter对象并不会被回收

			- 方法区中常量引用的对象

				public class MethodAreaStaicProperties {  
				 public static final MethodAreaStaicProperties m = MethodAreaStaicProperties("final");  
				 public MethodAreaStaicProperties(String name){}  
				}  
				public static void testGC(){  
				 MethodAreaStaicProperties s = new MethodAreaStaicProperties("staticProperties");  
				 s = null;  
				}  
				
				m即为方法区中的常量引用，也为GC Root，s置为null后，final对象也不会因没有与GC Root建立联系而被回收

		- 并发标记带来的问题

			浮动垃圾：原本消亡的对象错误的标记为存活，这不是好事，但是其实是可以容忍的，只不过产生了一点逃过本次回收的浮动垃圾而已，下次清理就可以。  
			
			一种是把原本存活的对象错误的标记为已消亡，这就是非常严重的后果了，一个程序还需要使用的对象被回收了，那程序肯定会因此发生错误

			- 三色标记

				白色：表示对象尚未被垃圾回收器访问过。显然，在可达性分析刚刚开始的阶段，所有的对象都是白色的，若在分析结束的阶段，仍然是白色的对象，即代表不可达。  
				
				黑色：表示对象已经被垃圾回收器访问过，且这个对象的所有引用都已经扫描过。黑色的对象代表已经扫描过，它是安全存活的，如果有其它的对象引用指向了黑色对象，无须重新扫描一遍。黑色对象不可能直接（不经过灰色对象）指向某个白色对象。  
				
				灰色：表示对象已经被垃圾回收器访问过，但这个对象至少存在一个引用还没有被扫描过。

- 回收垃圾

	- 标记-清除算法

		标记清除算法(Mark-Sweep)是最基础的一种垃圾回收算法，它分为2部分，先把内存区域中的这些对象进行标记，哪些属于可回收标记出来，然后把这些垃圾拎出来清理掉。就像图片一样，清理掉的垃圾就变成未使用的内存区域，等待被再次使用

		- 缺点-产生碎片问题

			图片中等方块的假设是2M，小一些的是1M，大一些的是4M。等我们回收完，内存就会切成了很多段。我们知道开辟内存空间时，需要的是连续的内存区域，这时候我们需要一个2M的内存区域，其中有2个1M是没法用的。这样就导致，其实我们本身还有这么多的内存的，但却用不了

	- 复制算法

		复制算法(Copying)是在标记清除算法上演化而来，解决标记清除算法的内存碎片问题。它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。保证了内存的连续可用，内存分配时也就不用考虑内存碎片等复杂情况，逻辑清晰，运行高效

		- 缺点-内存利用率低

	- 标记整理算法

		标记整理算法(Mark-Compact)标记过程仍然与标记--清除算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，再清理掉端边界以外的内存区域。标记整理算法一方面在标记-清除算法上做了升级，解决了内存碎片的问题，也规避了复制算法只能利用一半内存区域的弊端

		- 缺点-它对内存变动更频繁，需要整理所有存活对象的引用地址，在效率上比复制算法要差很多

	- 分代收集算法

		- 新生代(刚new出来的对象)

			- 每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集

		- 老年代

			- 对象存活率高、没有额外空间对它进行分配担保，就必须使用标记-整理算法来进行回收

		- GC

			- Minor GC

				- 作用于新生代，Eden区域满了，或者新创建的对象大小 > Eden所剩空间

			- Major GC

				- 作用于老年代，MajorGC是由MinorGC触发的，所以有时候很难将MajorGC和MinorGC区分开

			- Full GC

				- 清理整个堆空间，包括年轻代和永久代。Full GC一般消耗的时间比较长，远远大于Minor GC

					- 触发Full GC的两种情况

						- 之前每次晋升的对象的平均大小 > 老年代剩余空间

						- Minor GC后存活的对象超过了老年代剩余空间

			- GC调优通常就是为了改善stop-the-world的时间

- [垃圾收集器](https://juejin.im/post/5bade237e51d450ea401fd71)

	1.Serial new针对新生代复制算法
	2.Parallel new针对新生代复制算法  
	3.Parallel scavenge针对新生代复制算法  
	
	4.Serial old针对老年代标记整理  
	5.Parallel old针对老年代标记整理算法  
	
	6.CMS基于标记清理  
	
	7.G1整体基于标记整理，局部用复制算法  
	
	新生代基本复制算法，老年代基本标记整理算法，CMS采用标记清除算法。整个JAVA回收是新生代和老年代的协作，是分代回收算法

	- 新生代

		- Serial New收集器

			单线程执行垃圾回收。当需要执行垃圾回收时，程序会暂停一切手上的工作，然后单线程执行垃圾回收。  
			
			因为新生代的特点是对象存活率低，所以收集算法用的是复制算法，把新生代存活对象复制到老年代，复制的内容不多，性能较好

		- Parallel New收集器

			ParNew同样用于新生代，是Serial的多线程版本，并且在参数、算法（同样是复制算法）上也完全和Serial相同。
			Par是Parallel的缩写，但它的并行仅仅指的是收集多线程并行，并不是收集和原程序可以并行进行。ParNew也是需要暂停程序一切的工作，然后多线程执行垃圾回收。
			
			参数控制：-XX:+UseParNewGC  ParNew收集器
			
			-XX:ParallelGCThreads 限制线程数量

		- Parallel Scavenge收集器

			新生代的收集器，同样用的是复制算法，也是并行多线程收集。与ParNew最大的不同，它关注的是垃圾回收的吞吐量。
			这里的吞吐量指的是 总时间与垃圾回收时间的比例。这个比例越高，证明垃圾回收占整个程序运行的比例越小。
			Parallel Scavenge收集器提供两个参数控制垃圾回收的执行：
			
			-XX:MaxGCPauseMillis，最大垃圾回收停顿时间。这个参数的原理是空间换时间，收集器会控制新生代的区域大小，从而尽可能保证回收少于这个最大停顿时间。简单的说就是回收的区域越小，那么耗费的时间也越小。
			所以这个参数并不是设置得越小越好。设太小的话，新生代空间会太小，从而更频繁的触发GC。
			-XX:GCTimeRatio，垃圾回收时间与总时间占比。这个是吞吐量的倒数，原理和MaxGCPauseMillis相同。
			
			因为Parallel Scavenge收集器关注的是吞吐量，所以当设置好以上参数的时候，同时不想设置各个区域大小（新生代，老年代等）。可以开启**-XX:UseAdaptiveSizePolicy**参数，让JVM监控收集的性能，动态调整这些区域大小参数。

	- 老年代

		- Serial Old收集器

			老年代的收集器，与Serial一样是单线程，不同的是算法用的是标记-整理（Mark-Compact）。

		- Parallel Old收集器

			老年代的收集器，是Parallel Scavenge老年代的版本。其中的算法替换成Mark-Compact。

		- CMS收集器

			CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。目前很大一部分的Java应用都集中在互联网站或B/S系统的服务端上
			
			基于“标记-清除”算法实现，它的运作过程相对于前面几种收集器来说要更复杂一些，整个过程分为4个步骤，包括： 
			
			初始标记（CMS initial mark）
			
			并发标记（CMS concurrent mark）
			
			重新标记（CMS remark）
			
			并发清除（CMS concurrent sweep）
			
			其中初始标记、重新标记这两个步骤仍然需要“Stop The World”。初始标记仅仅只是标记一下GC Roots能直接关联到的对象，速度很快，并发标记阶段就是进行GC Roots Tracing的过程，而重新标记阶段则是为了修正并发标记期间，因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短。 
			由于整个过程中耗时最长的并发标记和并发清除过程中，收集器线程都可以与用户线程一起工作，所以总体上来说，CMS收集器的内存回收过程是与用户线程一起并发地执行。老年代收集器（新生代使用ParNew）
			
			有人会觉得既然Mark Sweep会造成内存碎片，那么为什么不把算法换成Mark Compact呢？
			答案其实很简答，因为当并发清除的时候，用Compact整理内存的话，原来的用户线程使用的内存还怎么用呢？要保证用户线程能继续执行，前提的它运行的资源不受影响嘛。Mark Compact更适合“Stop the World”这种场景下使用。
			  优点:并发收集、低停顿   
			
			   缺点：产生大量空间碎片、并发阶段会降低吞吐量，清除期间用户线程还在执行，会产生浮动垃圾，浮动垃圾只能下一次GC清除
			
			   参数控制：-XX:+UseConcMarkSweepGC  使用CMS收集器
			
			             -XX:+ UseCMSCompactAtFullCollection Full GC后，进行一次碎片整理；整理过程是独占的，会引起停顿时间变长
			
			            -XX:+CMSFullGCsBeforeCompaction  设置进行几次Full GC后，进行一次碎片整理
			
			            -XX:ParallelCMSThreads  设定CMS的线程数量（一般情况约等于可用CPU数量）

	- 跨越整个堆

		- G1收集器

			G1是目前技术发展的最前沿成果之一，HotSpot开发团队赋予它的使命是未来可以替换掉JDK1.5中发布的CMS收集器。与CMS收集器相比G1收集器有以下特点：
			
			1. 空间整合，G1收集器采用标记整理算法，不会产生内存空间碎片。分配大对象时不会因为无法找到连续空间而提前触发下一次GC。
			
			2. 可预测停顿，这是G1的另一大优势，降低停顿时间是G1和CMS的共同关注点，但G1除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为N毫秒的时间片段内，消耗在垃圾收集上的时间不得超过N毫秒，这几乎已经是实时Java（RTSJ）的垃圾收集器的特征了。
			
			上面提到的垃圾收集器，收集的范围都是整个新生代或者老年代，而G1不再是这样。使用G1收集器时，Java堆的内存布局与其他收集器有很大差别，它将整个Java堆划分为多个大小相等的独立区域（Region），虽然还保留有新生代和老年代的概念，但新生代和老年代不再是物理隔阂了，它们都是一部分（可以不连续）Region的集合。

			- G1对Heap的划分

				G1的新生代收集跟ParNew类似，当新生代占用达到一定比例的时候，开始出发收集。和CMS类似，G1收集器收集老年代对象会有短暂停顿。

			- 收集步骤

				1、标记阶段，首先初始标记(Initial-Mark),这个阶段是停顿的(Stop the World Event)，并且会触发一次普通Mintor GC。对应GC log:GC pause (young) (inital-mark)
				
				2、Root Region Scanning，程序运行过程中会回收survivor区(存活到老年代)，这一过程必须在young GC之前完成。
				
				3、Concurrent Marking，在整个堆中进行并发标记(和应用程序并发执行)，此过程可能被young GC中断。在并发标记阶段，若发现区域对象中的所有对象都是垃圾，那个这个区域会被立即回收(图中打X)。同时，并发标记过程中，会计算每个区域的对象活性(区域中存活对象的比例)。
				
				4、Remark, 再标记，会有短暂停顿(STW)。再标记阶段是用来收集 并发标记阶段 产生新的垃圾(并发阶段和应用程序一同运行)；G1中采用了比CMS更快的初始快照算法:snapshot-at-the-beginning (SATB)。  
				
				5、Copy/Clean up，多线程清除失活对象，会有STW。G1将回收区域的存活对象拷贝到新区域，清除Remember Sets，并发清空回收区域并把它返回到空闲区域链表中。  
				
				6、复制/清除过程后。回收区域的活性对象已经被集中回收到深蓝色和深绿色区域。

	- 总结

		- 收集器组合

- 什么时候回收

	- 两次标记

		如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选，筛选的条件是此对象是否有必要执行finalize（）方法。当对象没有覆盖finalize（）方法，或者finalize（）方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。
		
		如果这个对象被判定为有必要执行finalize（）方法，那么这个对象将会放置在一个叫做F-Queue的队列之中，并在稍后由一个由虚拟机自动建立的、低优先级的Finalizer线程去执行它。这里所谓的“执行”是指虚拟机会触发这个方法，但并不承诺会等待它运行结束，这样做的原因是，如果一个对象在finalize（）方法中执行缓慢，或者发生了死循环（更极端的情况），将很可能会导致F-Queue队列中其他对象永久处于等待，甚至导致整个内存回收系统崩溃。finalize（）方法是对象逃脱死亡命运的最后一次机会，稍后GC将对F-Queue中的对象进行第二次小规模的标记，如果对象要在finalize（）中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可，譬如把自己（this关键字）赋值给某个类变量或者对象的成员变量，那在第二次标记时它将被移除出“即将回收”的集合；如果对象这时候还没有逃脱，那基本上它就真的被回收了

		- 第一次：被可达性分析算法标记

		- 第二次：标记后没有复活的对象

		- 复活：在finalize方法重新引用对象

	- 优化联想

		1.拆分方法体，方法体不宜过长
		
		2.局部变量会随着方法执行完毕弹栈被回收，但如果局部变量之后有耗时操作，在该局部变量不需要使用的情况下，手动置为null，帮助GC回收  
		
		public void info(){  
		    Object object=new Object();  
		    System.out.println(object.toString());  
		    System.out.println(object.hashCode());  
		    object=null;  
		    //执行耗时，耗内存操作  
		    //或者调用耗时，耗内存的方法  
		}  
		
		3.避免循环中创建对象或者引用  
		4.考虑用软引用SoftReference

