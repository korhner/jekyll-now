---
layout: post
title: "Performance Traps in Hibernate - Part 2"
date: 2015-01-07 16:54:46
categories:
- Hibernate
published: true
---

In case you missed the previous [article]({% post_url 2015-01-03-hibernate-performance-traps-part-1 %}), I really recommend to read it. 
The series are best followed when read in order and each will contain one common problem with proposed solutions, measured evidence and pros/cons of the solutions.
I talked about one possible performance trap using hibernate and that was how to update a large number of objects in a loop which can cause severe performance drops and large memory overhead if done wrong.
In this post, we will tackle a similar requirement - how to achieve best performance when inserting a large number of records.<!--more-->

## Batch insert in Hibernate

Remember our data model from <LINK TO PART 1>? If you missed it, you can see it down below. It's a overly simple schema for storing users and their mail:

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

Without this "owning side" concept, Hibernate would treat this bidirectional association as independent - mail with a foreign key to user + a third table for association of user and mail.
By using mappedBy parameter of the @OneToMany association, we indicate that this is just a memory representation and that @ManyToOne field in Mail is the owner.
Now we can insert mails and link them to user directly by setting user reference.

Lets profile the naive approach first and check the performance: 

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

We created 50000 mails and filled them with random strings. Insertion took 13,5 seconds and in the image bellow you can see the heap explosion during this operation:

![_config.yml]({{ site.baseurl }}/assets/img/bulk_insert_naive.png)

### Clearing the session manually

In order to avoid keeping all those objects in memory, we can divide our session in transaction of the size of a batch. Hibernate documentation suggests 20 as a reasonable batch size so we can try with that.
After each 20 items, we will commit the transaction, which will cause Hibernate to flush the current session. 
Flushing the session runs a "dirty check" on all objects attached to the it. 
This means that Hibernate needs to check all attached objects, determine if any property of the object has changed, and remember it in a map for later generating of necessary SQL to persist those changes.
A dirty check is usually an expensive operation as Hibernate also checks the state of every dependent object.
Once all objects are checked for changes, Hibernate executes necessary SQL for the updates while updating the query cache.
After commit, to be sure we released all the resources, we need to additionally clear the session. The consequence of this is that any changes that were not flushed before calling clear will be lost.
Keep in mind that just flushing and clearing the session might not be enough as our session will still hold post-transaction executions which will be cleared only after committing the transaction.
With a multi-user environment you can see that is just a matter of few users and this system would collapse. So, lets try to break this into multiple transactions as suggested and see what we get.
I did an experiment with batch size to see how it: 

![_config.yml]({{ site.baseurl }}/assets/img/bulk_insert_batch_size_graph.png)

We can how the performance suffers when the batch size is less that 10 and than we gain very little by increasing it further. So will stick to our size of 20 and do the same test as before.
With 50000 mails, insertion took a bit longer that before - 16 seconds, but we optimized heap usage greatly which can be seen from the chart below. We saved about 300 mb of heap size!

![_config.yml]({{ site.baseurl }}/assets/img/bulk_insert_flush.png)

### Stateless sessions

There is another approach to doing efficient batch operations in Hibernate - using StatelessSession.
Contrary to regular Session, StatelessSession doesn't have a first-level cache nor does interact with second level or query cache,
collections are ignored when persisting, no dirty checking is done, and even Hibernate's event model and interceptors do not work.
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



Comparison with 5000 unread mails:

{:.table .table-bordered}
| Method    | Memory footprint | Number of queries                           | Duration |
|-----------|------------------|---------------------------------------------|----------|
| Naive     | All user's mail  | 1 SELECT and 1 UPDATE for every unread mail | 404 ms   |
| Efficient | None             | 1 UPDATE                                    | 45 ms    |


The benefits are huge (almost 10 times faster and without loading a single unneeded object from database) and will grow proportional to number of mails the user has.
The problem with this approach is that the query will not affect in-memory state of the objects and it is recommended to reload the objects in a new transaction when needed.