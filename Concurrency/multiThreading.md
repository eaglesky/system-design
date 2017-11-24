# Multithreading

## Thread basics

### Thread and process
* Thread library mappings, including Java. For Java, the mapping is many-to-many on Tru64 UNIX and later releases of the JVM. Refer to the book *Operating System Concepts* by Abraham, Peter and Greg.
* Similar goals: Split up workload into multiple parts and partition tasks into different, multiple tasks for these multiple actors. Two common ways of doing this are multi-threaded programs and multi-process systems. 
* Differences

| Criteria  |  Thread |  Process  |
| --------------------- |:-------------:| -----:|
| Def | A thread exists within a process and has less resource consumption | A running instance of a program |
| Resources | Multiple threads within the same process will share the same heap space but each thread still has its own registers and its own stack. | Each process has independent system resources. Inter process mechanism such as pipes, sockets, sockets need to be used to share resources. |
| Overhead for creation/termination/task switching  | Faster due to very little memory copying (just thread stack). Faster because CPU caches and program context can be maintained | Slower because whole process area needs to be copied. Slower because all process area needs to be reloaded |
| Synchronization overhead | Shared data that is modified requires special handling in the form of locks, mutexes and primitives | No synchronization needed |
| Use cases  | Threads are a useful choice when you have a workload that consists of lightweight tasks (in terms of processing effort or memory size) that come in, for example with a web server servicing page requests. There, each request is small in scope and in memory usage. Threads are also useful in situations where multi-part information is being processed – for example, separating a multi-page TIFF image into separate TIFF files for separate pages. In that situation, being able to load the TIFF into memory once and have multiple threads access the same memory buffer leads to performance benefits. | Processes are a useful choice for parallel programming with workloads where tasks take significant computing power, memory or both. For example, rendering or printing complicated file formats (such as PDF) can sometimes take significant amounts of time – many milliseconds per page – and involve significant memory and I/O requirements. In this situation, using a single-threaded process and using one process per file to process allows for better throughput due to increased independence and isolation between the tasks vs. using one process with multiple threads. |

* Context switch comparison: 
	* https://stackoverflow.com/questions/5440128/thread-context-switch-vs-process-context-switch
	* https://www.quora.com/How-does-thread-switching-differ-from-process-switching-What-is-the-performance-difference
* PCB vs TCB.
	- TCB: https://en.wikipedia.org/wiki/Thread_control_block
* Memory model for a process and its threads. 
	- Operating system level -- stack, heap, data and text.
	- JVM: http://tutorials.jenkov.com/java-concurrency/java-memory-model.html
		+ Mapping between JVM memory model and OS memory model.
		+ For variables shared by threads, why we might need to make them volatile or synchronized.


### Create threads
Refer to *Oracle Certified Professional Java SE 7 Programmer Exams 1Z0-804 and 1Z0-805 by S.G Ganesh & Tushar sharma*

#### Extending the Thread class
* We can create a thread by extending the Thread class. This will almost always mean that we override the run() method, and the subclass may also call the thread constructor explicitly in its constructor. 

```java
class MyThread1 extends Thread {
    public void run() {
        try {
          sleep(1000);
        } catch (InterruptedException ex) {
          ex.printStackTrace();
          // ignore the InterruptedException - this is perhaps the one of the
          // very few of the exceptions in Java which is acceptable to ignore
				}
        System.out.println("In run method; thread name is: "+getName());
    }
    public static void main(String args[])  {
      Thread myThread=new MyThread1();
      myThread.start();
      System.out.println("In main method; thread name is: "+
          Thread.currentThread().getName());
    } 			
}
```

#### Implementing the Runnable interface
* The runnable interface has the following very simple structure

```java
public interface Runnable
{
    void run();
}
```

* Steps
	- Create a class which implements the Runnable interface. An object of this class is a Runnable object
	- Create an object of type Thread by passing a Runnable object as argument to the Thread constructor. The Thread object now has a Runnable object that implements the run() method. 
	- The start() method is invoked on the Thread object created in the previous step. 

```java
class MyThread2 implements Runnable {
    public void run() {
    System.out.println("In run method; thread name is: "+ Thread.currentThread(	).getName());
    }

    public static void main(String args[]) throws Exception { 
      Thread myThread=new Thread(new MyThread2()); 
      myThread.start();
      System.out.println("In main method; thread name is: " + 
        Thread.currentThread().getName());
    } 
}
```

#### Extending the Thread Class vs Implementing the Runnable Interface
* Implementing runnable is the preferrable way. 
	- Java does not support multiple inheritance. Therefore, after extending the Thread class, you can't extend any other class which you required. A class implementing the Runnable interface will be able to extend another class. 
	- A class might only be interested in being runnable, and therefore, inheriting the full overhead of the Thread class would be excessive. 
* However, extending the Thread class is more convenient in many cases. In the example you saw for getting the name of the thread, you had to use Thread.currentThread().getName() when implementing the Runnable interface whereas you just used the getName() method directly while extending Thread since MyThread1 extends Thread. so, extending Thread is a little more convenient in this case.
* Thread and Runnable are complement to each other for multithreading not competitor or replacement. Because we need both of them for multi-threading.
	- For Multi-threading we need two things:
		+ Something that can run inside a Thread (Runnable).
		+ Something That can start a new Thread (Thread).
	- So technically and theoretically both of them is necessary to start a thread, one will run and one will make it run (Like Wheel and Engine of motor vehicle).



## Concurrency problems

### Thread safety
* Refer to *Java concurrency in Practice*.
* Definition: 
  > A class is thread-safe if it behaves correctly when accessed from multiple threads, regardless of the scheduling or interleaving of the execution of those threads by the runtime environment, and with no additional syn- chronization or other coordination on the part of the calling code.
* Problem 1: race condition.
  - Multiple compound actions that write to the same shared variable. E.g. read-modify-write (hit counter), or check-then-act (lazy initialization). Best solution is to make the shared variable use AtomicReference/AtomicInteger to ensure atomic accesses. Using lock in the application code is not good since you have to add it to all of them, and they have to be on the same object.
  - Compound action that writes to multiple shared variables that participate in an invariant. It might result in inconsistency between those variables if another read happens in the middle of the compound action. In this case, just making access to a single variable atomic is not enough. We have to make the compound action in the application code atomic by using lock, or **Synchronized** in Java. Also, for every invariant that involves more than one variable, all the variables involved in that invariant must be guarded by the same lock(for each mutable state variable that may be accessed by more than one thread, all accesses to that variable must be performed with the same lock held. In this case, we say that the variable is guarded by that lock).
    + Synchronized in Java
      + Intrinsic lock/monitor lock. Act as mutexes(mutual exclusion locks). Each object is associated with a monitor, which a thread can lock or unlock using synchronized keyword. Only one thread at a time may hold a lock on a monitor. Any other threads attempting to lock that monitor are **blocked** until they can obtain a lock on that monitor. 
      http://tutorials.jenkov.com/java-concurrency/synchronized.html
      https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html
      + Reentrancy. If a thread tries to acquire a lock that it already holds, the request succeeds. This can prevent deadlock and facilitates encapsulation of locking behavior.
* Problem 2: memory visibility issue(data race).
  - A data race occurs when a variable is read by more than one thread, and written by at least one thread, but the reads and writes are not ordered by happens-before, resulting in reading stale data issue.
  - Note that when a thread reads a variable without synchronization, it may see a stale value, but at least it sees a value that was actually placed there by some thread rather than some random value. Reads and writes are atomic for reference variables and for most primitive variables (all types except long and double, since they are 64-bit, and JVM is permitted to treat a 64-bit read or write as two separate 32-bit operations). Reads and writes are atomic for all variables declared volatile (including long and double variables). https://docs.oracle.com/javase/tutorial/essential/concurrency/atomic.html
  - *Java Language Specification* (or more specifically, the Java Memory Model part) allows the compiler to optimize the java bytecode by reordering and storing content in cache and registers instead of writing immediately to the main memory. It also specifies ways of limiting this optimization, i.e. establishing happens-before order between actions(like using locks and volatile). Details can be found in:
    - https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.4.5
    - *Java concurrency in Practice* 16.1.3.
    - https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/package-summary.html#MemoryVisibility
  
    The results of a write by one thread are guaranteed to be visible to a read by another thread only if the write operation happens-before the read operation. Any implementation of the java compiler or runtime must obey this, using means like enforcing write to the memory instead of local register or something else. 
* How to ensure thread safety?
  - Using thread confinement to make sure data is only accessed from a single thread. There are two ways of doing it:
    + Using stack confinement -- local variables.
    + Using ThreadLocal. Each thread object in Java has a field storing all the thread local variables associated with that thread. If a thread calls ThreadLocal.set, the value will be put into the thread local variables asscociated with that thread. ThreadLocal.get will get the most recent value of that thread local variable associated with that thread.
  - Using immutable object or holder.


### Liveness hazard

#### Deadlock
Continue refering to *Oracle Certified Professional Java SE 7 Programmer Exams 1Z0-804 and 1Z0-805 by S.G Ganesh & Tushar sharma*
##### Def
* A deadlock is a situation where a thread is waiting for an object lock that another thread holds, and this second thread is waiting for an object lock that the first thread holds. Since each thread is waiting for the other thread to relinquish a lock, they both remain waiting forever. 

##### Conditions
* **Mutal Exclusion**: Only one process can access a resource at a given time. (Or more accurately, there is limited access to a resource. A deadlock could also occur if a resource has limited quantity. )
* **Hold and Wait**: Processes already holding a resource can request additional resources, without relinquishing their current resources. 
* **No Preemption**: One process cannot forcibly remove another process' resource.
* **Circular Wait**: Two or more processes form a circular chain where each process is waiting on another resource in the chain. 

##### Example code
See the Oracle Java book.

#### Live lock and lock starvation.
See the Oracle Java book.

### Performance issues
* Mainly refer to *Java concurrency in practice*.
* Rule of thumb: Avoid premature optimization. First make it right, then make it fast—if it is not already fast enough.
* Sources of overhead:
  * Context switch. If there are more runnable threads than CPUs, eventually the OS will preempt one thread so that another can use the CPU. This causes a context switch, which requires saving the execution context of the currently running thread and restoring the execution context of the newly scheduled thread. This is one overhead of multithreaded programs.
    * Context switch could happen when the current thread is waiting for I/O, synchronization or because of an interrupt or timeout. The first two can be implemented by issuing a system call that does context switch, including saving the current context in stack and select a new PCB/TCB(there are many cpu scheduling algorithms that does it, like FIFO or round-robin) and load it in. The last two can be implemented using interrupt handlers, also piece of code stored in memory that does things including context switch, and triggered when there is an interrupt of that type. https://en.wikipedia.org/wiki/Context_switch#Interrupt_handling
    * As is mentioned earlier, a Java thread will block if it is not able to acquire an intrinsic lock used for synchronization. The JVM can implement blocking either via spin-waiting (repeatedly trying to acquire the lock until it succeeds) or by suspending the blocked thread through the operating system. Which is more efficient depends on the relationship between context switch overhead and the time until the lock becomes available; spin-waiting is preferable for short waits and suspension is preferable for long waits. Some JVMs choose between the two adaptively based on profiling data of past wait times, but most just suspend threads waiting for a lock, which entails two additional context switches and all the attendant OS and cache activity. -- the blocked thread is switched out before its quantum has expired, and is then switched back in later after the lock or other resource becomes available. (Blocking due to lock contention also has a cost for the thread holding the lock: when it releases the lock, it must then ask the OS to resume the blocked thread.)
  * Memory synchronization. This is the other source of overhead. The visibility guarantees provided by **synchronized** and **volatile** may entail using special instructions called memory barriers that can flush or invalidate caches, flush hardware write buffers, and stall execution pipelines. Memory barriers may also have indirect performance consequences because they inhibit other compiler optimizations; most operations cannot be reordered with memory barriers.
* Thread contention might make the concurrency level lower than what we expect. There are three ways of reducing the contention:
  - Reduce the duration for which locks are held
  - Reduce the frequency with which locks are requested
  - Replace exclusive locks with coordination mechanisms that permit greater concurrency.

## Java concurrency APIs.
### Thread.yield()
A hint to the scheduler that the current thread is willing to yield its current use of a processor. The scheduler is free to ignore this hint. Rarely used in production code.

### Object.wait(), Object.notify(), Object.notifyAll()
* According to *Operating System Concepts with Java, 8th edition*, the monitor lock of each object has an entry set and a wait set associated with it. 
* When a thread t tries to obtain the lock of an synchrnoized object m that has already been owned by another thread, it is put into the entry set; if after it owns the lock of m and calls m.wait, its state becomes WAITING/TIMED_WAITING and is put into the wait set, and releases ownership of this monitor and waits until another thread notifies threads waiting on m's monitor to wake up either through a call to the notify method or the notifyAll method. When another thread p calls m.notify, one of the threads in m's wait set(say u) is selected arbitrarily and waken up, and u's state is set to BLOCKED(this is according to https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.State.html, definition of BLOCKED. It is different from what is mentioned in that book and I think the latter is wrong), put into the entry set of m. According to https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html#notify-- , u will compete in the usual manner with any other threads that might be actively competing to synchronize on m. 
* More in https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.2

### Thread.join(long millis, int nanos)
Waits at most millis milliseconds plus nanos nanoseconds for this thread to die. This implementation uses a loop of this.wait calls conditioned on this.isAlive. As a thread terminates the this.notifyAll method is invoked. It is recommended that applications not use wait, notify, or notifyAll on Thread instances.

### Thread.sleep(long millis)
Causes the currently executing thread to sleep (temporarily cease execution) for the specified number of milliseconds, subject to the precision and accuracy of system timers and schedulers. The thread does not lose ownership of any monitors. This is an efficient means of making processor time available to the other threads of an application or other applications that might be running on a computer system. The thread state is changed to TIMED_WAITING if it sleeps. 
https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#sleep-long-



## Java concurrency APIs -- more
* Thread basics - join, yield, future
* Executor services
* Semaphore/Mutex - locks, synchronized keyword
* Condition variables - wait, notify, condition
* Concurrency collections - CountDownLatch, ConcurrentHashMap, CopyOnWriteArrayList

## Java concurrency frameworks

### Fork/Join framework
* https://docs.oracle.com/javase/tutorial/essential/concurrency/forkjoin.html
* Simple example: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/RecursiveAction.html
* This algorithm is suitable when we are aggregating a number of elements in a for loop. It is essentially a divide-and-conquer algorithm.
* How to use correctly: https://homes.cs.washington.edu/~djg/teachingMaterials/spac/grossmanSPAC_forkJoinFramework.html
* Work stealing:
  - https://apurvasingh67.wordpress.com/2015/04/12/java-forkjoinpool-internals-how-it-beats-plain-old-thread-pools-from-executorservice/
  - https://stackoverflow.com/questions/7926864/how-is-the-fork-join-framework-better-than-a-thread-pool
  - https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html
  - More details of the design and implementation: http://gee.cs.oswego.edu/dl/papers/fj.pdf

### Adding parallelism to memoization using Futures
See https://www.coursera.org/learn/parallel-programming-in-java/home/week/2

### Collection.parallelStream()
* Stream basics in Java 8: 
  * http://www.oracle.com/technetwork/articles/java/ma14-java-se-8-streams-2177646.html
  * https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html#package.description
* More on aggregate operations: https://docs.oracle.com/javase/tutorial/collections/streams/
* Collectors: https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collectors.html
* When a stream executes in parallel, the Java runtime partitions the stream into multiple substreams. Aggregate operations iterate over and process these substreams in parallel and then combine the results. https://docs.oracle.com/javase/tutorial/collections/streams/parallelism.html
* Whether or not use this: https://blog.oio.de/2016/01/22/parallel-stream-processing-in-java-8-performance-of-sequential-vs-parallel-stream-processing/
* Grouping by examples: https://www.mkyong.com/java8/java-8-collectors-groupingby-and-mapping-example/
* Streaming on map examples: https://www.mkyong.com/java8/java-8-filter-a-map-examples/
* Map comparator: https://stackoverflow.com/questions/39538089/get-object-with-max-frequency-from-java-8-stream

### Forall and barrier
* Refer to https://www.coursera.org/learn/parallel-programming-in-java/home/week/3
* When the iterations in the loop are independent, we can create a task for each iteration that is executed asynchronously. If there are too many tasks, we should group them so that the number of task is approximately the same as the number of CPU cores. This can improve the performance if each task is simply done by a new thread.
* Use barrier statement to let each thread wait until the other threads finish certain statements. How to implement it in Java?


## Counters
* See src dir for details

## Singleton
* See src dir for details

## BoundedBlockingQueue
* See src dir for details

## Readers/writers lock [To be finished]

## Thread-safe producer and consumer
* See src dir for details

## Delayed scheduler

### Interfaces to be implemented

```java
public interface Scheduler
{
	void schedule( Task t, long delayMs );
}

public interface Task
{
	void run();
}

```

### Single thread
* Main thread is in Timedwaiting state for delayMs for each call of schedule()
* Only one thread, very low CPU utilization
* Also, this is not working as later call
* How about sleeping in other threads

```java
public class SchedulerImpl implements Scheduler
{
	public void schedule( Task t, long delayMs )
	{
		try
		{
			// sleep for delayMs, and then execute the task
			Thread.sleep( delayMs );
			t.run();
		}
		catch ( InterruptedException e )
		{
			// ignore
		}
	}

	public static void main( String[] args )
	{
		Scheduler scheduler = new SchedulerImpl();
		Task t1 = new TaskImpl( 1 );
		Task t2 = new TaskImpl( 2 );

		// main thread in timedwaiting state for 10000 ms
		scheduler.schedule( t1, 10000 );
		scheduler.schedule( t2, 1 );
	}
}

```

### One thread for each task
* No blocking when calling schedule
* What happens if we call schedule many times
	- A lot of thread creation overhead
* Call be alleviated by using a thread pool, but still not ideal

```java
public class SchedulerImpl implements Scheduler
{
	public void schedule( Task t, long delayMs )
	{
		Thread t = new Thread( new Runnable() {
			public void run()
			{
				try 
				{
					Thread.sleep( delayMs );
					t.run();
				}
				catch ( InterruptedException e )
				{
					// ignore;
				}
			}
		} );
		t.start();
	}
}
```

### PriorityQueue + A background thread

```java
package designThreadSafeEntity.delayedTaskScheduler;

import java.util.PriorityQueue;
import java.util.concurrent.atomic.AtomicInteger;

public class Scheduler
{
	// order task by time to run
	private PriorityQueue<Task> tasks;

	// 
	private final Thread taskRunnerThread;

	// State indicating the scheduler is running
	// Why volatile? As long as main thread stops, runner needs to has visibility.
	private volatile boolean running;

	// Task id to assign to submitted tasks
	// AtomicInteger: Threadsafe. Do not need to add locks when assigning task Ids
	// Final: Reference of atomicInteger could not be changed
	private final AtomicInteger taskId;
	
	public Scheduler()
	{
		tasks = new PriorityQueue<>();
		taskRunnerThread = new Thread( new TaskRunner() );
		running = true;
		taskId = new AtomicInteger( 0 );

		// start task runner thread
		taskRunnerThread.start();
	}
	
	public void schedule( Task task, long delayMs )
	{
		// Set time to run and assign task id
		long timeToRun = System.currentTimeMillis() + delayMs;
		task.setTimeToRun( timeToRun );
		task.setId( taskId.incrementAndGet() );

		// Put the task in queue
		synchronized ( this )
		{
			tasks.offer( task );
			this.notify(); // only a single background thread waiting
		}
	}
	
	public void stop( ) throws InterruptedException
	{
		// Notify the task runner as it may be in wait()
		synchronized ( this )
		{
			running = false;
			this.notify();
		}

		// Wait for the task runner to terminate
		taskRunnerThread.join();
	}
	
	private class TaskRunner implements Runnable
	{
		@Override
		public void run()
		{
			while ( running )
			{
				// Need to synchronize with main thread
				synchronized( Scheduler.this )
				{
					try 
					{
						// task runner is blocked when no tasks in queue
						while ( running && tasks.isEmpty() )
						{
							Scheduler.this.wait();
						}

						// check the first task in queue
						long now = System.currentTimeMillis();
						Task t = tasks.peek();

						// delay exhausted, execute task
						if ( t.getTimeToRun() < now )
						{
							tasks.poll();
							t.run();
						}			
						else
						{
							// no task executable, wait
							Scheduler.this.wait( t.getTimeToRun() - now );
						}
					}
					catch ( InterruptedException e )
					{
						Thread.currentThread().interrupt();	
					}
				}
			}
		}
	}
	
	public static void main( String[] args ) throws InterruptedException
	{
		Scheduler scheduler = new Scheduler();
		scheduler.schedule( new Task(), 1000000 );
		scheduler.schedule( new Task(), 1000 );
		Thread.sleep( 7000 );
		scheduler.stop();
	}
}

class Task implements Comparable<Task> 
{
	// When the task will be run
	private long timeToRun;
	private int id;
	
	public void setId( int id )
	{
		this.id = id;
	}
	
	public void setTimeToRun( long timeToRun )
	{
		this.timeToRun = timeToRun;
	}
	
	public void run()
	{
		System.out.println( "Running task " + id );
	}
	
	public int compareTo( Task other )
	{
		return (int) ( timeToRun - other.getTimeToRun() );
	}
	
	public long getTimeToRun()
	{
		return timeToRun;
	}
}
```