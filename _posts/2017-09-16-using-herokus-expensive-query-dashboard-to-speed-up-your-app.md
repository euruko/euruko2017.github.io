---
layout: post
title:  "Using Heroku's Expensive Query Dashboard to Speed up your App"
date:   2017-09-23 09:00:00
author: "Richard Schneeman"
image:  "/assets/heroku-cover.png"
excerpt: "I recently demonstrated how you can use Rack Mini Profiler to find and fix slow queries. It’s a valuable tool for well-trafficked pages, but sometimes the slowdown is happening on a page you don't visit often, or in a worker task that isn't visible via Rack Mini Profiler. How can you find and fix those slow queries?"
post_id: "using-herokus-expensive-query-dashboard-to-speed-up-your-app"
---

# Using Heroku's Expensive Query Dashboard to Speed up your App

<p>I recently <a href="https://schneems.com/2017/06/22/a-tale-of-slow-pagination/">demonstrated how you can use Rack Mini Profiler to find and fix slow queries</a>. It’s a valuable tool for well-trafficked pages, but sometimes the slowdown is happening on a page you don&#39;t visit often, or in a worker task that isn&#39;t visible via Rack Mini Profiler. How can you find and fix those slow queries?</p>

<p>Heroku has a feature called <a href="https://devcenter.heroku.com/articles/expensive-queries">expensive queries</a> that can help you out.  It shows historical performance data about the queries running on your database: most time consuming, most frequently invoked, slowest execution time, and slowest I/O.</p>

<img class="post-content__embedded-image" src="https://heroku-blog-files.s3.amazonaws.com/posts/1499377005-expensive_queries.png" alt="expensive_queries">

<p>Recently, I used this feature to identify and address some slow queries for a site I run on Heroku named <a href="https://www.codetriage.com">CodeTriage</a> (the best way to get started contributing to open source). Looking at the expensive queries data for CodeTriage, I saw this:</p>

<img class="post-content__embedded-image" src="https://heroku-blog-files.s3.amazonaws.com/posts/1499374279-Screenshot%202017-06-22%2015.16.40.png" alt="Code Triage Project Expensive Query Screenshot">

<p>On the right is the query, on the left are two graphs; one graph showing the number of times the query was called, and another beneath that showing the average time it took to execute the query. You can see from the bottom graph that the average execution time can be up to 8 seconds, yikes! Ideally, I want my response time averages to be around 50 ms and perc 95 to be sub-second time, so waiting 8 seconds for a single query to finish isn&#39;t good.</p>

<p>To find this on your own apps you can follow directions on the <a href="https://devcenter.heroku.com/articles/expensive-queries">expensive queries documentation</a>. The documentation will direct you to <a href="https://data.heroku.com/">your database list page</a> where you can select the database you’d like to optimize. From there, scroll down and find the expensive queries near the bottom.</p>

<p>Once you&#39;ve chosen a slow query, you’ll need to determine why it&#39;s slow. To accomplish this use <code>EXPLAIN ANALYZE</code>:</p>

<pre><code class="sql">issuetriage::DATABASE=&gt; EXPLAIN ANALYZE
issuetriage::DATABASE-&gt; SELECT &quot;issues&quot;.*
issuetriage::DATABASE-&gt; FROM &quot;issues&quot;
issuetriage::DATABASE-&gt; WHERE &quot;issues&quot;.&quot;repo_id&quot; = 2151
issuetriage::DATABASE-&gt;         AND &quot;issues&quot;.&quot;state&quot; = &#39;open&#39;
issuetriage::DATABASE-&gt; ORDER BY  created_at DESC LIMIT 20 OFFSET 0;
                                                                       QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------------
Limit  (cost=27359.98..27359.99 rows=20 width=1232) (actual time=82.800..82.802 rows=20 loops=1)
   -&gt;  Sort  (cost=27359.98..27362.20 rows=4437 width=1232) (actual time=82.800..82.801 rows=20 loops=1)
         Sort Key: created_at
         Sort Method: top-N heapsort  Memory: 31kB
         -&gt;  Bitmap Heap Scan on issues  (cost=3319.34..27336.37 rows=4437 width=1232) (actual time=27.725..81.220 rows=5067 loops=1)
               Recheck Cond: (repo_id = 2151)
               Filter: ((state)::text = &#39;open&#39;::text)
               Rows Removed by Filter: 13817
               -&gt;  Bitmap Index Scan on index_issues_on_repo_id  (cost=0.00..3319.12 rows=20674 width=0) (actual time=24.293..24.293 rows=21945 loops=1)
                     Index Cond: (repo_id = 2151)
Total runtime: 82.885 ms
</code></pre>

<p>In this case, I&#39;m using <a href="https://www.codetriage.com/kubernetes/kubernetes">Kubernetes</a> because they currently have the highest issue count, so querying on that page will likely give me the worst performance.</p>

<p>We see the total time spent was 82 ms, which isn&#39;t bad for one of the &quot;slowest&quot; queries, but we&#39;ve seen that some can be way worse. Most single queries should be aiming for around a 1 ms query time.</p>

<p>We see that before the query can be made it has to sort the data, this is because we are using an <code>order</code> on an <code>offset</code> clause. Sorting is a very expensive operation, you can see that it says the &quot;actual time&quot; can take between <code>27.725</code> ms and <code>81.220</code> ms just to sort the data, which is pretty tough. If we can get rid of this sort then we can drastically improve our query.</p>

<p>One way to do this is... you guessed it, add an index. Unlike <a href="https://schneems.com/2017/06/22/a-tale-of-slow-pagination/">last week</a> though, the issues table is HUGE. While the table we indexed last week only had around 2K entries, each of those entries can have a virtually unbounded number of issues. In the case of Kubernetes there are 5K+ issues, and that&#39;s only the <code>state=open</code> ones. The closed issue count is much larger than that, and it will only grow over time. We want to be mindful of taking up too much database size, so instead of indexing ALL the data, we can instead apply a partial index. I&#39;m almost never querying for <code>state=closed</code> when it comes to issues, so we can ignore those while building our index. Here&#39;s the migration I used to add a partial index:</p>

<pre><code class="ruby">class AddCreatedAtIndexToIssues &lt; ActiveRecord::Migration[5.1]
  def change
    add_index :issues, :created_at, where: &quot;state = &#39;open&#39;&quot;
  end
end
</code></pre>

<p>What&#39;s the result of adding this index? Let&#39;s look at that same query we analyzed before:</p>

<pre><code class="sql">
issuetriage::DATABASE=&gt; EXPLAIN ANALYZE
issuetriage::DATABASE-&gt; SELECT &quot;issues&quot;.*
issuetriage::DATABASE-&gt; FROM &quot;issues&quot;
issuetriage::DATABASE-&gt; WHERE &quot;issues&quot;.&quot;repo_id&quot; = 2151
issuetriage::DATABASE-&gt;         AND &quot;issues&quot;.&quot;state&quot; = &#39;open&#39;
issuetriage::DATABASE-&gt; ORDER BY  created_at DESC LIMIT 20 OFFSET 0;
                                                                         QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------
Limit  (cost=0.08..316.09 rows=20 width=1232) (actual time=0.169..0.242 rows=20 loops=1)
   -&gt;  Index Scan Backward using index_issues_on_created_at on issues  (cost=0.08..70152.26 rows=4440 width=1232) (actual time=0.167..0.239 rows=20 loops=1)
         Filter: (repo_id = 2151)
         Rows Removed by Filter: 217
Total runtime: 0.273 ms
</code></pre>

<p>Wow, from 80+ ms to less than half a millisecond. That&#39;s some improvement. The index keeps our data already sorted, so we don&#39;t have to re-sort it on every query. All elements in the index are guaranteed to be <code>state=open</code> so the database doesn&#39;t have to do more work there. The database can simply scan the index removing elements where <code>repo_id</code> is not matching our target.</p>

<p>For this case it is EXTREMELY fast, but can you imagine a case where it isn&#39;t so fast?</p>

<p>Perhaps you noticed that we still have to iterate over issues until we&#39;re able to find ones matching a given Repo ID. I&#39;m guessing that since this repo has the most issues, it&#39;s able to easily find 20 issues with <code>state=open</code>. What if we pick a different repo?</p>

<p>I looked up the oldest open issue and found it in <a href="https://www.codetriage.com/rails/journey">Journey</a>. Journey has an ID of 10 in the database. If we do the same query and look at Journey:</p>

<pre><code class="sql">issuetriage::DATABASE=&gt; EXPLAIN ANALYZE
issuetriage::DATABASE-&gt; SELECT &quot;issues&quot;.*
issuetriage::DATABASE-&gt; FROM &quot;issues&quot;
issuetriage::DATABASE-&gt; WHERE &quot;issues&quot;.&quot;repo_id&quot; = 10
issuetriage::DATABASE-&gt;         AND &quot;issues&quot;.&quot;state&quot; = &#39;open&#39;
issuetriage::DATABASE-&gt; ORDER BY  created_at DESC LIMIT 20 OFFSET 0;
                                                                     QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=757.18..757.19 rows=20 width=1232) (actual time=21.109..21.110 rows=6 loops=1)
   -&gt;  Sort  (cost=757.18..757.20 rows=50 width=1232) (actual time=21.108..21.109 rows=6 loops=1)
         Sort Key: created_at
         Sort Method: quicksort  Memory: 26kB
         -&gt;  Index Scan using index_issues_on_repo_id on issues  (cost=0.11..756.91 rows=50 width=1232) (actual time=11.221..21.088 rows=6 loops=1)
               Index Cond: (repo_id = 10)
               Filter: ((state)::text = &#39;open&#39;::text)
               Rows Removed by Filter: 14
 Total runtime: 21.140 ms
</code></pre>

<p>Yikes. Previously we&#39;re only using 0.27 ms, now we&#39;re back up to 21 ms. This might not have been the &quot;8 second&quot; query we were seeing before, but it&#39;s definitely slower than the first query we profiled. </p>

<p>Even though we&#39;ve got an index on <code>created_at</code> Postgres has decided not to use it. It&#39;s reverting back to a sorting algorithm and using an index on <code>repo_id</code> to pull the data. Once it has issues then it iterates over each to remove where the state is not open.</p>

<p>In this case, there are only 20 total issues for Journey, so grabbing all the issues and iterating and sorting manually was deemed to be faster. Does this mean our index is worthless? Well considering this repo only has 1 subscriber, it&#39;s not the case we need to be optimizing for. Also if lots of people visit that page (maybe because of this article), then Postgres will speed up the query by using the cache. The second time I ran the exact same explain query, it was much faster:</p>

<pre><code> Total runtime: 0.092 ms
</code></pre>

<p>Postgres already had everything it needed in the cache. Does this mean we&#39;re totally out of the woods then? Going back to my expensive queries page after a few days, I saw that my 8 second worst case is gone, but I still have a 2 second query every now and then.</p>

<img class="post-content__embedded-image" src="https://heroku-blog-files.s3.amazonaws.com/posts/1499377612-Screenshot%202017-06-26%2010.47.20.png" alt="Expensive Queries Screenshot 2">

<p>This is still a 75% performance increase (in worst case performance) so the index is still useful. One really useful feature of Postgres is the ability to <a href="https://www.postgresql.org/docs/8.3/static/indexes-bitmap-scans.html">combine multiple indexes</a>. In this case, even though we have an index on <code>created_at</code> and an index on <code>repo_id</code>, Postgres does not seem to think it&#39;s faster to combine the two and use that result. To fix this issue we can add an index that has both <code>created_at</code> and <code>repo_id</code>, which maybe I&#39;ll explore in the future.</p>

<p>Before we go, I want to circle back to how we found our slow query test case. I had to know a bit about the data and make some assumptions about the worst case scenarios. I had to guess that <a href="https://www.codetriage.com/kubernetes/kubernetes">Kubernetes</a> was our worst offender, which ended up not being true. Is there a better way than guess and check?</p>

<p>It turns out that Heroku will <a href="https://devcenter.heroku.com/articles/postgres-logs-errors#log-duration-3-565-s">output slow queries into your app&#39;s logs</a>. Unlike the expensive queries, these logs also contain the parameters used in the query, and not just the query. If you have a logging addon such as <a href="https://elements.heroku.com/addons/papertrail">Papertrail</a>, you can search your logs for <code>duration</code> and get a result like this:</p>

<pre><code>Jun 26 06:36:54 issuetriage app/postgres.29339:  [DATABASE] [39-1] LOG:  duration: 3040.545 ms  execute &lt;unnamed&gt;: SELECT COUNT(*) FROM &quot;issues&quot; WHERE &quot;issues&quot;.&quot;repo_id&quot; = $1 AND &quot;issues&quot;.&quot;state&quot; = $2
Jun 26 06:36:54 issuetriage app/postgres.29339:  [DATABASE] [39-2] DETAIL:  parameters: $1 = &#39;696&#39;, $2 = &#39;open&#39;
Jun 26 08:26:25 issuetriage app/postgres.29339:  [DATABASE] [40-1] LOG:  duration: 9087.165 ms  execute &lt;unnamed&gt;: SELECT COUNT(*) FROM &quot;issues&quot; WHERE &quot;issues&quot;.&quot;repo_id&quot; = $1 AND &quot;issues&quot;.&quot;state&quot; = $2
Jun 26 08:26:25 issuetriage app/postgres.29339:  [DATABASE] [40-2] DETAIL:  parameters: $1 = &#39;1245&#39;, $2 = &#39;open&#39;
Jun 26 08:49:40 issuetriage app/postgres.29339:  [DATABASE] [41-1] LOG:  duration: 2406.615 ms  execute &lt;unnamed&gt;: SELECT  &quot;issues&quot;.* FROM &quot;issues&quot; WHERE &quot;issues&quot;.&quot;repo_id&quot; = $1 AND &quot;issues&quot;.&quot;state&quot; = $2 ORDER BY created_at DESC LIMIT $3 OFFSET $4
Jun 26 08:49:40 issuetriage app/postgres.29339:  [DATABASE] [41-2] DETAIL:  parameters: $1 = &#39;1348&#39;, $2 = &#39;open&#39;, $3 = &#39;20&#39;, $4 = &#39;760&#39;
</code></pre>

<p>In this case, we can see that our 2.4 second query (the last query in the logs above) is using a repo id of <code>1348</code> and an offset of <code>760</code>, which brings up another important point. As the offset goes up, the cost of scanning our index will also go up, so it turns out that we had a worse case than my initial guess (Kubernetes) and my second guess (Journey). It is likely that this repo has lots of issues that are old, and this query isn&#39;t made often, so that the data is not in cache. By using the logs we can find the exact worst case scenario without all the guessing.</p>

<p>Before you start writing that comment message, yes, I know that offset pagination is broken and <a href="https://www.citusdata.com/blog/2016/03/30/five-ways-to-paginate/">there are other ways to paginate</a>. I may start to look at alternative pagination options, or even getting rid of some of the pagination on the site altogether. </p>

<p>I did go back and <a href="https://github.com/codetriage/codetriage/commit/ce3ac6a59f92891e4a42b85927c852f074a0be3f">add an index to both the <code>created_at</code> and <code>repo_id</code> columns</a>. With the addition of those two indexes my &quot;worst case&quot; of 2.4 seconds is now down to 14 ms:</p>

<pre><code class="sql">issuetriage::DATABASE=&gt; EXPLAIN ANALYZE SELECT  &quot;issues&quot;.*
issuetriage::DATABASE-&gt; FROM &quot;issues&quot;
issuetriage::DATABASE-&gt; WHERE &quot;issues&quot;.&quot;repo_id&quot; = 1348
issuetriage::DATABASE-&gt; AND &quot;issues&quot;.&quot;state&quot; = &#39;open&#39;
issuetriage::DATABASE-&gt; ORDER BY created_at DESC
issuetriage::DATABASE-&gt; LIMIT 20 OFFSET 760;
                                                                                QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=1380.73..1417.06 rows=20 width=1232) (actual time=14.515..14.614 rows=20 loops=1)
   -&gt;  Index Scan Backward using index_issues_on_repo_id_and_created_at on issues  (cost=0.08..2329.02 rows=1282 width=1232) (actual time=0.061..14.564 rows=780 loops=1)
         Index Cond: (repo_id = 1348)
 Total runtime: 14.659 ms
(4 rows)
</code></pre>

<p>Here you can see that we&#39;re able to use our new index directly and find only the issues that are open and belonging to a specific repo id.</p>

<p>What did I learn from this experiment?</p>

<ul>
<li>You can find <a href="https://devcenter.heroku.com/articles/expensive-queries">slow queries using Heroku&#39;s expensive queries</a> feature.</li>
<li>The exact arguments matter a lot when profiling queries. Don&#39;t assume that you know the most expensive thing your database is doing, use metrics.</li>
<li>You can find the exact parameters that go with those expensive queries by <a href="https://devcenter.heroku.com/articles/postgres-logs-errors#log-duration-3-565-s">grepping your logs for the exact parameters of those queries</a>.</li>
<li>Indexes help a ton, but you have to understand the different ways your application will use them. It&#39;s not enough to profile with 1 query before and after, you need to profile a few different queries with different performance characteristics. In my case not only did I add an index, I went back to the expensive index page which let me know that my queries were still taking a long time (~2 seconds). </li>
<li>Performance tuning isn&#39;t about magic fixes, it&#39;s about finding a toolchain you understand, and iterating on a process until you get the results you want.</li>
</ul>

<p>
  This blog post was originally published on
  <a href="https://blog.heroku.com/expensive-query-speed-up-app">blog.heroku.com</a>.
</p>
