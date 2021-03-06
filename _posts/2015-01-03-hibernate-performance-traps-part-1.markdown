---
layout: post
title: "Performance Traps in Hibernate - Part 1"
date: 2015-01-03 16:54:46
categories:
- Hibernate
published: true
---
#### List of all articles in this series:
* Part 1 - One by One Processing
* [Part 2 - Batch Insert]({% post_url 2015-01-07-hibernate-performance-traps-part-2 %})
* [Part 3 - HiLo id allocation]({% post_url 2015-01-11-hibernate-performance-traps-part-3 %})

Hibernate is the most popular Object-Relational Mapping (ORM) library for Java. 
It provides a framework for mapping object-oriented domain models to the underlying relational database and also generates SQL for retrieving and persisting the data.
This convenience allows developers to focus on business logic without having to worry too much about data access details, allowing more rapid development.
However, this often comes with a hidden price, as it is easy to write suboptimal code with problems that usually manifest in production with larger scale of data and when it's very costly to fix them. 
In the next series of posts, I will share with you some of the most common performance traps or anti-patterns in Hibernate that are easy to fix and can provide huge instant performance gains. 
Keep in mind that those features don't indicate faults in Hibernate, but incorrect usage of the tool and in some cases limitations of the ORM frameworks in general.<!--more-->  

## Operating on objects instead of data sets

Traditional relational databases are optimized to operate on sets instead of single rows. 
While it is natural to write such operations in SQL, Java is an object-oriented language where the only way to manipulate collections of data is to iterate and process elements one by one.
In case of persistent collections, consequences are:    
  
* A large number of database round trips (proportional to the size of collection).    
* In cases where application and database are on different machines, network latency can also become a problem.  

Let's create a simplistic model which we will use for our experiments:

![_config.yml]({{ site.baseurl }}/assets/img/user_to_mails.png)

{% highlight Java %}
@Entity(name = "users")
public class User {
  @Id
  @GeneratedValue
  private int id;
		
  private String username;
		
  @OneToMany
  @JoinColumn(name = "user_id")
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
	
  /** getters, setters, etc. **/
}
{% endhighlight %}

It is a basic one-to-many relationship, modelling users that have a list of mails. 
Now, suppose we have a "mark all as read" button in our applications that goes simply marks all mail as read in one step.
A common approach to accomplish this would be:
{% highlight java %}
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

User user = getCurrentUser();

for( Mail mail : user.getMail() ) {
  if( !mail.isRead() ) {
    mail.setIsRead(true);
    session.update(mail);
  }
}

tx.commit();
session.close();
{% endhighlight %}

If we look at queries Hibernate generated, we would notice:

* 1 SELECT that loads all users mail into memory
* 1 UPDATE for every unread mail

The problems arise when we try doing this for a user that has a ton of mails - we have to load all of the mails into memory even when we don't need a single one!
This can cause slower performance due to more garbage collections or even page swaps. Another problem is in the number of update queries done for setting each mail to read.
Even if this can be accomplished through a simple update sql query, Hibernate will create a new query for each mail being modified which can cause problems mentioned before in this text.

### Solution

My proposed solution is, once you detect you have a similar situation with a possible large number of rows, simply create a more optimized sql query to do the job and execute it instead.
This may be more work and it defeats the purpose of ORM in a way, but there is no tool that is perfect for everything.
Hibernate can be inefficient with big data and we need to get our hands dirty in such situations to keep the performance and responsiveness of our system at a reasonable level.

One way of doing this is to create a named query through User class annotation and add a method to execute the query:

{% highlight java %}
@NamedQueries({ 
  @NamedQuery(name = "markMailAsRead", 
  query = "UPDATE mails SET read = true WHERE user_id = :userId")
})
public class User {
  /** ... */
	
  public void markMailAsRead() {
    Query query = DAO.createQuery("markMailAsRead");
    query.setParameter("userId", this.id);
    query.executeUpdate();
  }
}
{% endhighlight %}

Comparison with 5000 unread mails:

{:.table .table-bordered}
| Method    | Memory footprint | Number of queries                           | Duration |
|-----------|------------------|---------------------------------------------|----------|
| Naive     | All user's mail  | 1 SELECT and 1 UPDATE for every unread mail | 404 ms   |
| Efficient | None             | 1 UPDATE                                    | 45 ms    |


The benefits are huge (almost 10 times faster and without loading a single unneeded object from database) and will grow proportional to number of mails the user has.
The problem with this approach is that the query will not affect in-memory state of the objects and it is recommended to reload the objects in a new session or execute such bulk operations in the beginning of a session before persistence cache caches any of affected objects.
You also lose some of the benefits Hibernate provides such as optimistic locking so better make sure you have enough data to justify it. This is best to be done when optimizing your code and profiling for bottlenecks as premature unneeded optimization is often a common source of problems.

#### List of all articles in this series:
* Part 1 - One by One Processing
* [Part 2 - Batch Insert]({% post_url 2015-01-07-hibernate-performance-traps-part-2 %})
* [Part 3 - HiLo id allocation]({% post_url 2015-01-11-hibernate-performance-traps-part-3 %})