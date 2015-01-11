---
layout: post
title: "Performance Traps in Hibernate - Part 3"
date: 2015-01-11 16:54:46
categories:
- Hibernate
published: true
---

#### List of all articles in this series:
* [Part 1 - One by One Processing]({% post_url 2015-01-03-hibernate-performance-traps-part-1 %})
* [Part 2 - Batch Insert]({% post_url 2015-01-07-hibernate-performance-traps-part-2 %})
* Part 3 - HiLo id allocation

In [my previous post]({% post_url 2015-01-07-hibernate-performance-traps-part-2 %}), we explored ways how we can optimize batch inserts with Hibernate and keeping our heap memory consumption at a reasonable level.
If you played with that code yourself and looked at SQL it generates, you would notice two things:

* Even if we are committing our transaction in batches, Hibernate still generates inserts for one row at a time
* Before every insert, Hibernate does a database round trip to select the next available sequencer value 

In this article, we are going to see how we can change that and measure how much can we gain. If your application is doing lots of inserts, you can optimize sequence allocation by using HiLo algorithm.
<!--more--> 

## Id allocation using HiLo

HiLo algorithm is best understood by looking at how and what it does. The client contacts the server and requests a "Hi" value - which is nothing more than a plain integer. 
We define the "Lo" value which effectively means how many keys we can generate without getting a new "Hi" value from database. This could be our transaction size for example.
Let's suppose we define the "Lo" value as 20. We query the database and get 1 as "Hi". Now we are free to use the following interval of sequences which is calculated offline without querying the database:

[ ((Hi-1) * Low) + 1, (Hi*Lo) + 1 )

For Hi = 1, Lo = 20 this equals [1,21) which means we can use:
1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,19,20
safely without worrying about unique constraint as long as all clients are following the same algorithm.
If, at the mid time, other clients contact database for their "Hi" values, the will get [2,31) and [3,41) and so on.

The advantages of HiLo are that it is totally database-independent and you do not need to modify your inserting code to use it, it's just a matter of changing sequence generator configuration.
Be careful about using a large "Lo" value as the "Hi" is updated whenever a new session factory is created and if you do not have enough inserts and session factory is frequently restarted, you will have gaps in your ids.

Once more, we are going to use the model from previous articles to do our tests. We will modify the Mail entity to configure the Hilo generator and measure the insert of 50,000 mails with and without HiLo.

![_config.yml]({{ site.baseurl }}/assets/img/user_to_mails.png)

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
  @GeneratedValue(generator = "mails_id_seq", strategy = GenerationType.SEQUENCE)
  @SequenceGenerator(name = "mails_id_seq", sequenceName = "mails_id_seq", allocationSize = 20)
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

For an explanation of GeneratorType strategies, you can refer to 
<a href="http://docs.jboss.org/hibernate/annotations/3.5/reference/en/html_single/#entity-mapping-identifier">documentation</a>.

{:.table .table-bordered}
| Method    			 	    | Without HiLo | Lo = 20 | Lo = 50,000 |
|-------------------------------|--------------|---------|-------------|
| Regular session with flushing | 16.9 s       | 10.6 s  | 9.4 s       |
| StatelessSession  		    | 13.8 s       | 8.6 s   | 8 s         |

Regular and stateless session were explained and tested in [part 2]({% post_url 2015-01-07-hibernate-performance-traps-part-2 %}) so take a look if you missed it.
I also tried a more extreme setting with "Lo" = 50,000 which I really don't recommend as the gain comparing to "Lo" = 20 is not much and we would risk making big gaps in our ids if we don't insert so much mails every time.
However, with using HiLo with our proposed parameters, improvement is really noticeable. 

## Making hibernate batch inserts

Even with this improvement, we could still buy time by making hibernate use JDBC batch api. This is highly dependent on your driver and how it handles it and you may or may not see any performance gains.
I tested with PostgreSQL and although Hibernate logs said it was batching the inserts, the driver was still sending the same SQL to database as with batching disabled.
As this is easy to set up and might work for you, I think it is worth mentioning.
Just add hibernate.jdbc.batch_size = 20 to your hibernate.properties file and set Hibernate to log everything while testing. You can do this by editing log4j.properties and setting log4j.logger.org.hibernate=trace.
You can analyse the logs around inserts as Hibernate will log whether or not it is batching statements.
I also recommend to profile the database and check the generated SQL before and after enabling batching. 
If you succeed to take advantage of this for your database, leave a comment and I will update the post. Thanks!

#### List of all articles in this series:
* [Part 1 - One by One Processing]({% post_url 2015-01-03-hibernate-performance-traps-part-1 %})
* [Part 2 - Batch Insert]({% post_url 2015-01-07-hibernate-performance-traps-part-2 %})
* Part 3 - HiLo id allocation