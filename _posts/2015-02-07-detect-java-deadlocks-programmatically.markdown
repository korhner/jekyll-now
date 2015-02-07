---
layout: post
title: "How to detect Java deadlocks programmatically"
date: 2015-02-07 16:54:46
categories:
- Java
- Multithreading
published: true
---
Deadlocks are situations in which two or more actions are waiting for the others to finish, making all actions in a blocked state forever.
They can be very hard to detect during development, and they usually require restart of the application in order to recover.
To make things worse, deadlocks usually manifest in production under the heaviest load, and are very hard to spot during testing.
The reason for this is it's not practical to test all possible interleavings of a program's threads.
Although some statical analysis libraries exist that can help us detect the possible deadlocks, 
it is still necessary to be able to detect them during runtime and get some information which can help us fix the issue or alert us so we can restart our application or whatever.
<!--more--> 

## Detect deadlocks programmatically using ThreadMXBean class

Java 5 introduced <a href="http://docs.oracle.com/javase/1.5.0/docs/api/java/lang/management/ThreadMXBean.html">ThreadMXBean</a> - an interface that provides various monitoring methods for threads. 
I recommend you to check all of the methods as there are many useful operations for monitoring the performance of your application in case you are not using an external tool.
The method of our interest is <a href="http://docs.oracle.com/javase/1.5.0/docs/api/java/lang/management/ThreadMXBean.html#findMonitorDeadlockedThreads%28%29">findMonitorDeadlockedThreads</a>,
or, if you are using Java 6, <a href="http://docs.oracle.com/javase/6/docs/api/java/lang/management/ThreadMXBean.html#findDeadlockedThreads%28%29">findDeadlockedThreads</a>.
The difference is that findDeadlockedThreads can also detect deadlocks caused by owner locks (java.util.concurrent), while findMonitorDeadlockedThreads can only detect monitor locks (i.e. synchronized blocks).
Since the old version is kept for compatibility purposes only, I am going to use the second version. The idea is to encapsulate periodical checking for deadlocks into a reusable component so we can just fire and forget about it.

One way to impement scheduling is through executors framework - a set of well abstracted and very easy to use multithreading classes. 

{% highlight Java %}
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
this.scheduler.scheduleAtFixedRate(deadlockCheck, period, period, unit);
{% endhighlight %}

Simple as that, we have a runnable called periodically after a certain amount of time determined by period and time unit.
Next, we want to make our utility is extensive and allow clients to supply the behaviour that gets triggered after a deadlock is detected.
We need a method that receives a list of objects describing threads that are in a deadlock:

{% highlight Java %}
void handleDeadlock(final ThreadInfo[] deadlockedThreads);
{% endhighlight %}

Now we have everything we need to implement our deadlock detector class.

{% highlight Java %}
public interface DeadlockHandler {
  void handleDeadlock(final ThreadInfo[] deadlockedThreads);
}

public class DeadlockDetector {

  private final DeadlockHandler deadlockHandler;
  private final long period;
  private final TimeUnit unit;
  private final ThreadMXBean mbean = ManagementFactory.getThreadMXBean();
  private final ScheduledExecutorService scheduler = 
  Executors.newScheduledThreadPool(1);
  
  final Runnable deadlockCheck = new Runnable() {
    @Override
    public void run() {
      long[] deadlockedThreadIds = DeadlockDetector.this.mbean.findDeadlockedThreads();
    
      if (deadlockedThreadIds != null) {
        ThreadInfo[] threadInfos = 
        DeadlockDetector.this.mbean.getThreadInfo(deadlockedThreadIds);
      
        DeadlockDetector.this.deadlockHandler.handleDeadlock(threadInfos);
      }
    }
  };
  
  public DeadlockDetector(final DeadlockHandler deadlockHandler, 
    final long period, final TimeUnit unit) {
    this.deadlockHandler = deadlockHandler;
    this.period = period;
    this.unit = unit;
  }
  
  public void start() {
    this.scheduler.scheduleAtFixedRate(
    this.deadlockCheck, this.period, this.period, this.unit);
  }
}
{% endhighlight %}

Let's test this in practice. First, we will create a handler to output deadlocked threads information to System.err. We could use this to send email in a real world scenario, for example:

{% highlight Java %}
public class DeadlockConsoleHandler implements DeadlockHandler {

  @Override
  public void handleDeadlock(final ThreadInfo[] deadlockedThreads) {
    if (deadlockedThreads != null) {
      System.err.println("Deadlock detected!");
      
      Map<Thread, StackTraceElement[]> stackTraceMap = Thread.getAllStackTraces();
      for (ThreadInfo threadInfo : deadlockedThreads) {
      
        if (threadInfo != null) {
      
          for (Thread thread : Thread.getAllStackTraces().keySet()) {
        
            if (thread.getId() == threadInfo.getThreadId()) {
              System.err.println(threadInfo.toString().trim());
                
              for (StackTraceElement ste : thread.getStackTrace()) {
                  System.err.println("\t" + ste.toString().trim());
              }
            }
          }
        }
      }
    }
  }
}
{% endhighlight %}

This iterates through all stack traces and prints stack trace for each thread info. This way we can know exactly on which line each thread is waiting, and for which lock.
This approach has one downside - it can give false alarms if one of the threads is waiting with a timeout which can actually be seen as a temporary deadlock. 
Because of that, original thread could no longer exist when we handle our deadlock and findDeadlockedThreads will return null for such threads. To avoid possible NullPointerExceptions, we need to guard for such situations.
Finally, lets force a simple deadlock and see our system in action:

{% highlight Java %}
DeadlockDetector deadlockDetector = new DeadlockDetector(new DeadlockConsoleHandler(), 5, TimeUnit.SECONDS);
deadlockDetector.start();

final Object lock1 = new Object();
final Object lock2 = new Object();

Thread thread1 = new Thread(new Runnable() {
  @Override
  public void run() {
    synchronized (lock1) {
      System.out.println("Thread1 acquired lock1");
      try {
        TimeUnit.MILLISECONDS.sleep(500);
      } catch (InterruptedException ignore) {
      }
      synchronized (lock2) {
        System.out.println("Thread1 acquired lock2");
      }
    }
  }

});
thread1.start();

Thread thread2 = new Thread(new Runnable() {
  @Override
  public void run() {
    synchronized (lock2) {
      System.out.println("Thread2 acquired lock2");
      synchronized (lock1) {
        System.out.println("Thread2 acquired lock1");
      }
    }
  }
});
thread2.start();
{% endhighlight %}

Output:  
Thread1 acquired lock1  
Thread2 acquired lock2  
<span style="color:red">
Deadlock detected!  
"Thread-1" Id=11 BLOCKED on java.lang.Object@68ab95e6 owned by "Thread-0" Id=10  
deadlock.DeadlockTester$2.run(DeadlockTester.java:42)    
&nbsp;&nbsp;java.lang.Thread.run(Thread.java:662)  
"Thread-0" Id=10 BLOCKED on java.lang.Object@58fe64b9 owned by "Thread-1" Id=11  
&nbsp;&nbsp;deadlock.DeadlockTester$1.run(DeadlockTester.java:28)  
&nbsp;&nbsp;java.lang.Thread.run(Thread.java:662)
</span>

Keep in mind that deadlock detection can be an expensive operation and you should test it with your application to determine if you even need to use it and how frequent you will check. 
I suggest an interval of at least several minutes as it is not crucial to detect deadlock more frequently than this as we don't have a recovery plan anyway - we can only debug and fix the error or restart the application and hope it won't happen again.
If you have any suggestions about dealing with deadlocks, or a question about this solution, drop a comment below. 
