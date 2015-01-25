---
layout: post
title: "Java Multithreading Made Easy - Expensive Object Pool"
date: 2015-01-25 16:54:46
categories:
- Java
- Multithreading
published: true
---

Since Java SE 5.0, developing multithreaded applications became much easier due to the task executor framework. 
Instead of working with low level synchronization constructs, the framework introduces the concepts of Task and Executors.
It also provides isolation between task submission and task execution, allowing to easily change execution policy without even touching submission logic.
Still, it doesn't save you from creating race conditions and other difficult to debug and discover bugs, so in order to use the framework to its full power, 
I recommend starting from basics. A great book that covers almost everything you need to know about multithreading in Java is Java Concurrency in Practice by Brian Goetz and its a must if you are developing in Java.
In this article, I will show you how to create a useful utility class for managing a pool of expensive objects, and how simple it is to create complex structures by reusing what Java offers.
<!--more--> 

Imagine we are writing an online multiplayer game and we want to offer several players to play together a <a href="http://en.wikipedia.org/wiki/Procedural_generation">procedural generated</a> level. 
Since the level generation is a very expensive operation, we do not want to affect our players experience and having them to wait too long for it to complete.
Instead, we want to maintain a pool of already generated levels that we can offer immediately. 
Also, we want to support parallel creation of additional levels to maintain our cache full.
With this things in mind, lets create a reusable component that supports the operations we need.

## ExpensiveObjectPool

There are a few built in classes we can use. First one is <a href="http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/BlockingQueue.html">BlockingQueue</a>.
BlockingQueue is usually used when we want to produce and consume its items from different threads. 
It can block the consuming thread while the queue is empty and even timeout after a specified amount of time to avoid waiting indefinitely.
In Java, there are several implementations of BlockingQueue, you can check them out yourself. 
For our example I decided to use <a href="http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/LinkedBlockingQueue.html">LinkedBlockingQueue</a>.

{% highlight Java %}
private final BlockingQueue<T> pool = new LinkedBlockingQueue<T>();
{% endhighlight %}

For refilling our cache, we will use a simple fixed thread pool with a configurable number of threads to produce new objects as soon as we consume.
Fixed in this case means that there will be a maximum number of threads, and that operations that are submitted will wait for a thread to become available. 
I recommend you to explore the other available executors as other classes may be better suited for your use case.

{% highlight Java %}
private final ExecutorService poolService = Executors.newFixedThreadPool(nThreads);
{% endhighlight %}

Now we are able to write our refill method:

{% highlight Java %}
private void refill(final int nObjects) {
  for (int i = 0; i < nObjects; i++) {
  Callable<T> callable = new Callable<T>() {
    @Override
	public T call() throws Exception {
	  return produce();
	}
  };
  
  this.poolService.submit(new CallbackTask(callable));
  }
}
{% endhighlight %}

This simply sumbits a number of tasks to our thread pool. CallbackTask is the task we will be using internally which will refill our queue upon finishing. 
If we did that in our refill method, the method would be blocked until our cache is full which is not efficient since we will be refilling after consuming each object.
Finally, we can write the method for requesting objects. It will return null in case of a timeout and refill if we returned and object:

{% highlight Java %}
public T requestObject() {
  try {
    T expensiveObject = this.pool.poll(this.timeout, this.timeoutUnit);
    refill(1);
      return expensiveObject;
  } catch (InterruptedException e) {
    return null;
  } 
}
{% endhighlight %}	

I think I covered everything needed to build the utility. You can find the whole class below, and if there is still something needed to explain, feel free to drop me a comment.

{% highlight Java %}
public abstract class ExpensiveObjectPool<T> {
  private class CallbackTask implements Runnable {
  	private final Callable<T> callable;
  
  	public CallbackTask(final Callable<T> callable) {
      this.callable = callable;
    }
  
    @Override
    public void run() {
      try {
  	    ExpensiveObjectPool.this.pool.add(this.callable.call());
  	  } catch (Exception e) {
        // do nothing
  	  }
    } 
  }

  private final int capacity;
  private final BlockingQueue<T> pool = new LinkedBlockingQueue<T>();
  private final ExecutorService poolService;
  private final TimeUnit timeoutUnit;
  private final long timeout;
  
  public ExpensiveObjectPool(final int capacity, final int nThreads, final long timeout, final TimeUnit timeUnit) {
  	this.capacity = capacity;
  	this.timeout = timeout;
  	this.timeoutUnit = timeUnit;
  	this.poolService = Executors.newFixedThreadPool(nThreads);
  	refill(this.capacity);
  }

  public T requestObject() {
    try {
  	  T expensiveObject = this.pool.poll(this.timeout, this.timeoutUnit);
  	  refill(1);
  	  return expensiveObject;
  	} catch (InterruptedException e) {
  	  return null;
  	} 
  }
  
  private void refill(final int nObjects) {
  	for (int i = 0; i < nObjects; i++) {
  	  Callable<T> callable = new Callable<T>() {
  	    @Override
  		public T call() throws Exception {
  		  return produce();
  		}
      };
  	
	  this.poolService.submit(new CallbackTask(callable));
    }
  }
  
  protected abstract T produce();
}
{% endhighlight %}	

Finally, lets create our level pool and test it.
As seen below, the only thing we have to do is configure capacity, number of threads and timeout, and override produce() to be able to produce our levels.
For simplicity, to simulate work, I added a sleep in our produce method.

{% highlight Java %}
public class LevelPool extends ExpensiveObjectPool<Integer> {

  public LevelPool() {
    super(5, 10, 5000, TimeUnit.MILLISECONDS);
  }

  @Override
  protected Integer produce() {
  	try {
	  // simulate work
  	  Thread.sleep(3000);
  	} catch (InterruptedException e) {

  	}
  	
  	return (int)(Math.random()*100);
  }
	
  public static void main (String[] args) {
  	LevelPool levelPool = new LevelPool();
  	
  	for( int i = 0; i < 50; i++) {
  	  System.out.println(levelPool.requestObject());
  	}
  }
}
{% endhighlight %}	


