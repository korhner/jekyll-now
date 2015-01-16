---
layout: post
title: "PostgreSQL Query Tuning - Indices"
date: 2015-01-16 16:54:46
categories:
- PostgreSQL
published: true
---

Even with hardware becoming more powerful everyday, it is essential to keep the response time small for complex applications which use large amounts of data. 
Many database performance problems can be addressed by analyzing and optimizing SQL queries, which many developers avoid to learn or do. 
Part of the problem is that there is no magic bullet that we can read and follow - a query that runs locally just fine can fail miserably on production system.
The problems that you solve may or may not manifest on a similar system, and vice versa. 
When I first started with PostreSQL, analyzing queries looked like black magic but once you get the basics it becomes a fun process.
<!--more--> 

## Sample Schema

To assure you can follow the rest of the article, I will first define a simple schema and upgrade it to our needs after each problem we identify.

{% highlight Sql %}
CREATE TABLE "users" (
	"id" SERIAL NOT NULL,
	"username" text NOT NULL,
	PRIMARY KEY("id")
);

CREATE TABLE "mails" (
	"id" SERIAL NOT NULL,
	"subject" text,
	"body" text,
	"read" bool NOT NULL,
	"user_id" int4,
	PRIMARY KEY("id")
);

ALTER TABLE "mails" ADD CONSTRAINT "Ref_mail_to_user" FOREIGN KEY ("user_id")
	REFERENCES "users"("id")
	MATCH SIMPLE
	ON DELETE CASCADE
	ON UPDATE NO ACTION
	NOT DEFERRABLE;
{% endhighlight %}

Next, we will create a simple stored procedure to insert a bunch of test data for us so we can actually notice some of the common performance problems:

{% highlight Sql %}
CREATE OR REPLACE FUNCTION "fill_with_test_data" (
  p_user_count int4,
  p_mail_count int4			
) RETURNS "pg_catalog"."void" AS 
$body$
DECLARE
v_user_id int4;
BEGIN
  FOR i IN 1..p_user_count LOOP
    INSERT INTO "users" ("username") 
    VALUES (md5(random()::text)) 
    RETURNING "id" INTO v_user_id;
  
    FOR i IN 1..p_mail_count LOOP
      INSERT INTO "mails" ("subject", "body", "read", "user_id")
      VALUES (md5(random()::text), md5(random()::text), FALSE, v_user_id);
    END LOOP;
  END LOOP;
END;
$body$
LANGUAGE 'plpgsql';
{% endhighlight %}

We will call this with:

{% highlight Sql %}
SELECT * FROM "fill_with_test_data" (10, 100000);
{% endhighlight %}

Feel free to select a lower number as this can take some time to finish (it took about 5 minuts on my machine).

## Using EXPLAIN

This will be our main tool for examining queries. 
By executing EXPLAIN [STATEMENT], we will get an execution plan that shows how tables referenced by the stateent will be scanned, which indices and join algorithms will be used, in case we join multiple tables.
It also displays estimated execution costs in units that are relative to the actual time.
We can also use EXPLAIN ANALYZE [STATEMENT] to actually execute the statement and get actual times also. Beware that this can be tedious for long queries but provides more accurate results than just using EXPLAIN.
You can also use EXPLAIN on INSERT, UPDATE, DELETE, CREATE TABLE AS, or EXECUTE statement. As those statements are actually executed, you can wrap them in a transaction block and rollback:

{% highlight Sql %}
BEGIN;
EXPLAIN ANALYZE ...;
ROLLBACK;
{% endhighlight %}

Lets see a few examples:

{% highlight Sql %}
EXPLAIN ANALYZE SELECT * FROM "users";
{% endhighlight %}

<pre>
Seq Scan on users  (cost=0.00..3.05 rows=5 width=36) (actual time=0.025..0.046 rows=10 loops=1)
Total runtime: 0.069 ms
</pre>

In the first row, we see that the planner choose to do a seq scan which means iterate sequentially over the whole table.
This can often be a red flag in big tables, but in this case it's actually logical as we requested the whole dataset.

{% highlight Sql %}
EXPLAIN ANALYZE SELECT * FROM "mails" WHERE id = 100;
{% endhighlight %}

<pre>
Index Scan using mails_pkey on mails  (cost=0.00..8.91 rows=1 width=75) (actual time=0.041..0.041 rows=0 loops=1)
  Index Cond: (id = 100)
Total runtime: 0.065 ms
</pre>

Here the planner did an index scan which means a lookup data structure is used to filter our rows.
Now we will try several scenarios that will cause suboptimal performance and we will try to fix things up by either changing the query or the database schema.

### Cascade Delete Without Foreign Key Index

Lets try to delete the user with maximum id. Since we are selecting the user based on primary key, which is always indexed, this query should be super fast, right?

{% highlight Sql %}
BEGIN;
EXPLAIN ANALYZE DELETE FROM "users" WHERE "id" = (SELECT MAX(id) FROM "users");
ROLLBACK;
{% endhighlight %}

<pre>
Delete on users  (cost=3.07..6.13 rows=1 width=6) (actual time=0.059..0.059 rows=0 loops=1)
  InitPlan 1 (returns $0)
    ->  Aggregate  (cost=3.06..3.07 rows=1 width=4) (actual time=0.025..0.025 rows=1 loops=1)
          ->  Seq Scan on users  (cost=0.00..3.05 rows=5 width=4) (actual time=0.002..0.019 rows=10 loops=1)
  ->  Seq Scan on users  (cost=0.00..3.06 rows=1 width=6) (actual time=0.040..0.041 rows=1 loops=1)
        Filter: (id = $0)
Trigger for constraint Ref_mail_to_user: time=8219.577 calls=1
Total runtime: 8219.677 ms
</pre>

Whoa, 8 seconds for deleting a single user? So lets examine what happened here. 
Lets look at the first part - we are doing a seq scan on users and aggregate. This is the part the calculates max(id).
So, why seq scan if this can be found efficiently using the index? The answer is that we only have 10 users and with numbers this small, planner can choose to scan through the whole table as this can be more efficient.
Similar with the second scan - it decided to scan the table to find which rows to delete.
The guilty line is the last one - Trigger for constraint. If you look at foreign key definition, we set it to ON DELETE CASCADE, which means that if we delete an user, all of his mails will automatically be deleted.
Lets try to implement this manually and analyze the query to try to figure what is going on(I will not actually execute it as it takes too long so lets just use EXPLAIN):

{% highlight Sql %}
EXPLAIN DELETE FROM "mails" WHERE "user_id" = (SELECT MAX(id) FROM "users");
{% endhighlight %}

<pre>
Delete on mails  (cost=3.07..173624.46 rows=358110 width=6)
  InitPlan 1 (returns $0)
    ->  Aggregate  (cost=3.06..3.07 rows=1 width=4)
          ->  Seq Scan on users  (cost=0.00..3.05 rows=5 width=4)
  ->  Seq Scan on mails  (cost=0.00..173621.39 rows=358110 width=6)
        Filter: (user_id = $0)
</pre>

After a quick scan on users to find max we see a seq scan on mails which is a huge table. 
This means that, in order to delete users mails, we need to scan through the whole table to filter all mails that belong to user.
We could solve this by adding an index to user_id. Lets t do that and analyze user deletion one more time:

{% highlight Sql %}
CREATE INDEX "mails_user_id_idx" ON "mails" ("user_id");
{% endhighlight %}

Deletion took 3 seconds and we can also fetch all users mails efficiently.

### Wrong Usage of WHERE

The sooner you stop to trust the query planner too much the better and faster queries you will end up writing.
A lot of developers just hope that the query is going to be fast, that their index is going to be used without even checking and than the surprise often strikes at production under heaviest loads.

Consider the following query:

{% highlight Sql %}
SELECT * FROM "mails" where "user_id" % 100 = 0;
{% endhighlight %}

Explaining this query shows we are doing a seq scan. B tree indices are tree structures and can efficiently do only few operations - less than, less than or equal, equal, greater than or equal and greater than.
We can add a functional index to solve this:

{% highlight Sql %}
CREATE INDEX "mails_user_id_even_idx" ON "mails" (mod("user_id", 100));
{% endhighlight %}

Now consider those 2 logically identical queries:

{% highlight Sql %}
EXPLAIN ANALYZE SELECT * FROM "mails" where "user_id" % 100 = 0;
EXPLAIN ANALYZE SELECT * FROM "mails" where mod("user_id",100) = 0;
{% endhighlight %}

The first one does a seq scan and takes about 2 seconds, while the second uses our index and takes 0.048 ms.

Another example:

{% highlight Sql %}
EXPLAIN ANALYZE SELECT * FROM "mails" where "user_id" - 20 < 0;
EXPLAIN ANALYZE SELECT * FROM "mails" where "user_id" < 20;
{% endhighlight %}

Again, even if we have an index on user_id, the planner won't utilize it if we don't makes his life easy with the second query.

### Incorrect Volatility of Stored Procedure

In PostreSQL, every function has a volatility with one of the next values: VOLATILE, STABLE or IMMUTABLE, with VOLATILE being the default.
You can think of it as a promise you make to the query optimizer about whether your function changes the database, or will it return same output for the same input everytime.
From the <a href="http://www.postgresql.org/docs/9.4/static/xfunc-volatility.html">docs</a>:

* A VOLATILE function can do anything, including modifying the database. It can return different results on successive calls with the same arguments. The optimizer makes no assumptions about the behavior of such functions. A query using a volatile function will re-evaluate the function at every row where its value is needed.

* A STABLE function cannot modify the database and is guaranteed to return the same results given the same arguments for all rows within a single statement. This category allows the optimizer to optimize multiple calls of the function to a single call. In particular, it is safe to use an expression containing such a function in an index scan condition. (Since an index scan will evaluate the comparison value only once, not once at each row, it is not valid to use a VOLATILE function in an index scan condition.)

* An IMMUTABLE function cannot modify the database and is guaranteed to return the same results given the same arguments forever. This category allows the optimizer to pre-evaluate the function when a query calls it with constant arguments. For example, a query like SELECT ... WHERE x = 2 + 2 can be simplified on sight to SELECT ... WHERE x = 4, because the function underlying the integer addition operator is marked IMMUTABLE.

What this means is, if your function doesn't change when input is constant, marking it as restrictive as possible can bring optimizations.
Suppose we have a function that maps an external user id to database defined as follows:

{% highlight Sql %}
CREATE OR REPLACE FUNCTION "get_transformed_user_id" (
  p_user_id int4	
) RETURNS int4 AS 
$body$
BEGIN
	RETURN p_user_id * 1000;
END;
$body$
IMMUTABLE
LANGUAGE 'plpgsql';
{% endhighlight %}

This function is created with default volatility - VOLATILE. 
This means the planner assumes the output can change randomly (examples of such function are random(), currval(), timeofday()) and it must execute it every time without caching the value.

Lets analyze the following query:

{% highlight Sql %}
EXPLAIN ANALYZE SELECT * FROM "mails" WHERE "user_id" = get_transformed_user_id(10); 
{% endhighlight %}

<pre>
Seq Scan on mails  (cost=0.00..2495834.00 rows=1000000 width=75) (actual time=10841.561..10841.561 rows=0 loops=1)
  Filter: (user_id = get_transformed_user_id(10))
Total runtime: 10841.592 ms
</pre>

As expected, we get a seq scan on mails which is unacceptable. By simple changing the volatility to IMMUTABLE like this:

{% highlight Sql %}
CREATE OR REPLACE FUNCTION "get_transformed_user_id" (
  p_user_id int4	
) RETURNS int4 AS 
$body$
BEGIN
	RETURN p_user_id * 1000;
END;
$body$
IMMUTABLE
LANGUAGE 'plpgsql';
{% endhighlight %}

The function is executed once and converted to a constant.

<pre>
Index Scan using mails_user_id_idx on mails  (cost=0.00..9.36 rows=1 width=75) (actual time=0.054..0.054 rows=0 loops=1)
  Index Cond: (user_id = 10000)
Total runtime: 0.069 ms
</pre>

Keep in mind that you can trick PostgreSQL by abusing this, but you will get a speed boost with incorrect results.
You need to actually make sure your function is following the restrictions specified for a selected volatility.

I hope this gave you a basic idea how to optimize and check your own queries.
In the next article I will tackle some of the more complex cases that include querying from multiple tables, so if you found this one useful, subscribe at the top of the page and don't miss the second part!
If you have any questions or suggestions, drop me a comment below and I will answer as soon as I can. Happy tuning!

