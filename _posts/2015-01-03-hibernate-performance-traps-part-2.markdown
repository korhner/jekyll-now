---
layout: post
title: "Performance Traps in Hibernate - Part 2"
date: 2015-01-07 16:54:46
categories:
- Hibernate
published: true
---

#### List of all articles in this series:
* [Part 1 - One by One Processing]({% post_url 2015-01-03-hibernate-performance-traps-part-1 %})
* Part 2 - Batch Insert

In case you missed the previous [article]({% post_url 2015-01-03-hibernate-performance-traps-part-1 %}), I really recommend to read it. 
The series are best followed when read in order and each will contain one common problem with proposed solutions, measured evidence and pros/cons of the solutions.
I talked about one possible performance trap using hibernate and that was how to update a large number of objects in a loop which can cause severe performance drops and large memory overhead if done wrong.
In this post, we will tackle a similar requirement - how to achieve best performance when inserting a large number of records.<!--more-->

## Batch insert in Hibernate

Remember our data model from [previous article]({% post_url 2015-01-03-hibernate-performance-traps-part-1 %})? If you missed it, you can see it down below. It's a overly simple schema for storing users and their mail:

![_config.yml]({{ site.baseurl }}/assets/img/user_to_mails.png)

Now, suppose our users requested to be able to import their old mail from their old providers. Let's start once more with the naive approach and see why it isn't going to work for us.
We will modify our mapping a bit and make ManyToOne the owning side of user-mail association.
As database relations are non bidirectional - we store the association only in one place (in this case mail has the foreign key to user).
Since in java we can make this association redundant for easier processing, Hibernate still needs to know which reference is the real one, or owner of the association to avoid trying to save redundant data.
Our entities will look like this now:

{% highlight Java %}
@Entity(name = "users")
public class User {
  @Id
  @GeneratedValue
  private int id;
		
  private String username;
		
  @OneToMany(mappedBy = "user")
  private Set<Mail> mail = new HashSet<Mail>();
		
  /** getters, setters, etc. **/
}

@Entity(name = "mails")
public class Mail {

  @Id
  @GeneratedValue
  private int id;
	
  private String subject;
  private String body;
  private boolean read;
  
  @ManyToOne
  @JoinColumn(name = "user_id")
  private User user;
	
  /** getters, setters, etc. **/
}
{% endhighlight %}

Without this "owning side" concept, Hibernate would treat this bidirectional association as two different - mail with a foreign key to user + a third table for association of user and mail.
By using mappedBy parameter of the @OneToMany association, we indicate that this is just a memory representation and that @ManyToOne field in Mail is the owner.
Now we can insert mails and link them to user directly by setting user reference.

So, let's say we have an api ready to insert the mails. 
We will simulate this by inserting mails with random strings as I'm a lazy person and I lied about having the api. 
Simplest and most common approach is of course to loop and insert all the messages in a single transaction. Let's profile this naive approach first and check the performance: 

{% highlight Java %}
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

User user = getCurrentUser();

for (int j = 0; j < 50000; j++) {
  Mail m = new Mail();
  m.setUser(user);
  m.setSubject(randomString(20));
  m.setBody(randomString(1000));
  session.save(m);
}

tx.commit();
session.close();
{% endhighlight %}

We created 50,000 mails and filled them with random strings. Insertion took 13,5 seconds and on the chart bellow you can see the heap explosion during this operation.
Because Hibernate keeps a reference in it's cache on all attached objects in a session, we will keep all 50,000 emails in memory during the session.
With a multi-user environment you can see that is just a matter of few users and this system would collapse.
We need a better way and we will discuss two possible work arounds.

![_config.yml]({{ site.baseurl }}/assets/img/bulk_insert_naive.png)

### Clearing the session manually

In order to avoid keeping all those objects in memory, we can divide our session in batches of transactions. Hibernate documentation suggests 10-50 as a reasonable batch size, so let's pick 20.
After each 20 items, we will commit the transaction, which will cause Hibernate to flush the current session and open a new one for next batch.
Flushing the session runs a "dirty check" on all objects attached to it. 
This means that Hibernate needs to check all attached objects, determine if any property of the object has changed, and remember it in a map for later generating of necessary SQL to persist those changes.
A dirty check is usually an expensive operation as Hibernate also checks the state of every dependent object.
Once all objects are checked for changes, Hibernate executes necessary SQL for the updates while updating the query cache.
After commit, to be sure we released all the resources, we need to additionally clear the session. The consequence of this, is that any changes that were not flushed before calling clear, will be lost.
Keep in mind that just flushing and clearing the session might not be enough as our session will still hold lists of post-transaction executions which will be cleared only after committing the transaction.
I did an experiment with batch size to see how the size affects the total duration of insert:

![_config.yml]({{ site.baseurl }}/assets/img/bulk_insert_batch_size_graph.png)

We can see how the performance suffers when the batch size is less that 10 and than we gain very little by increasing it further. So will stick to our size of 20 and do the same test as before.
With 50,000 mails, insertion took a bit longer than before - 16 seconds, but we optimized heap usage greatly, which can be seen from the chart below. We saved about 300 MB of heap size!

{% highlight Java %}
Session session = sessionFactory.openSession();
Transaction tx;

int batchSize = 20;
int numberOfElements = 50000;
while (numberOfElements > 0) {
  int elementsInThisBatch = Math.min(numberOfElements, batchSize);
  numberOfElements -= elementsInThisBatch;
	
  tx = session.beginTransaction();
	
  for (int i = 0; i < elementsInThisBatch; i++) {	
	Mail m = new Mail();
	m.setUser(user);
	m.setSubject(randomString(20));
	m.setBody(randomString(1000));
	session.save(m);
  }
	
  tx.commit();
  session.clear();
}
{% endhighlight %}

![_config.yml]({{ site.baseurl }}/assets/img/bulk_insert_flush.png)

### Stateless sessions

There is another approach to doing efficient batch operations in Hibernate - using StatelessSession.
Contrary to regular Session, StatelessSession:

* Doesn't have a first-level cache nor it interacts with second level or query cache.
* Collections are ignored when persisting.
* No dirty checking is done.
* Hibernate's event model and interceptors do not work.

When invoking operations on stateless sessions(insert, delete, update) corresponding SQL is immediately executed on database and these commands are not equivalent to save, saveOrUpdate and delete defined by regular session.

{% highlight Java %}
StatelessSession session = sessionFactory.openStatelessSession();
Transaction tx = session.beginTransaction();

User user = getCurrentUser();

for (int j = 0; j < 50000; j++) {
  Mail m = new Mail();
  m.setUser(user);
  m.setSubject(randomString(20));
  m.setBody(randomString(1000));
  session.insert(m);
}

tx.commit();
session.close();
{% endhighlight %}

Running this on our test data took 14,4 seconds and the heap usage confirms the claims about not updating any cache:

![_config.yml]({{ site.baseurl }}/assets/img/bulk_insert_stateless.png)

This approach is the fastest and most memory efficient of all but it is less flexible and I recommend using it only when you are sure you can live with limitations and the performance gain is enough to justify it.


Let's make a summary of all our measurements below:

{:.table .table-bordered}
| Method    			| Memory footprint 	    | Duration | Comment                                                                     			 |
|-----------------------|-----------------------|----------|-----------------------------------------------------------------------------------------|
| Naive     			| ~ 400 MB 				| 13.5 s   | Not acceptable for large data sets. Simple to use for trivial cases.					 |
| Multiple Transactions | ~ 60 MB   			| 16 s     | Recommended way for handling larger data sets in most cases. A bit complex but flexible.|
| Stateless session     | ~ 50 MB 				| 14.4 s   | Can be used when high level featured that stateless sessions disable are not necessary. |

There is still room for improvement here - we can speed up out inserts by configuring hibernate to actually perform inserts as a batch instead of doint them one by one.
I will talk about that in the next part so stay tuned!

#### List of all articles in this series:
* [Part 1 - One by One Processing]({% post_url 2015-01-03-hibernate-performance-traps-part-1 %})
* Part 2 - Batch Insert