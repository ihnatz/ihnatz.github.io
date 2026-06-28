---
layout: post
title:  "Tree Traversal in PostgreSQL: ltree, GiST, and Recursive CTEs"
date:   2015-07-21 09:49:35
tags: postgres ltree recursive
---

PostgreSQL provides you ltree extension to organize tree-like structures. You can enable this extension with command `CREATE EXTENSION IF NOT EXISTS ltree;` or you can wrap it with ActiveRecord

{% highlight ruby %}
def up
  execute "CREATE EXTENSION IF NOT EXISTS ltree"
end
{% endhighlight %}


In Ruby, you can use `ltree_hierarchy` gem to simplify main operations for manipulating with tree's nodes. It updates path column, that contains all hierarchy for each node. For example, if node parents are 1, 3, 5 and node id is 9, then node path will contain `1.3.5.9`. It allows you to easily filter results. Note that the path format differs slightly from the raw [PostgreSQL documentation](<www.postgresql.org/docs/9.4/static/ltree.html>).

As sandbox we will use simple database that contains two tables, one for hierarchical structure and second for some additional data connected to each node of tree. To emulate `ltree_hierarchy` gem behaviour we fill the path column manually. Don't forget to enable `ltree` extension.

{% highlight sql %}
create temp table properties (
  id bigserial primary key,
  name varchar(256) NOT NULL,
  path ltree,
  parent_id integer
);

create temp table reports (
  id bigserial primary key,
  property_id integer,
  value integer
);

insert into properties (name, path) VALUES ('A', '1'); -- NULL parent_id for root
insert into properties (name, path, parent_id) VALUES ('B', '1.2', 1);
insert into properties (name, path, parent_id) VALUES ('C', '1.3', 1);
insert into properties (name, path, parent_id) VALUES ('D', '1.4', 1);
insert into properties (name, path, parent_id) VALUES ('E', '1.2.5', 2);
insert into properties (name, path, parent_id) VALUES ('F', '1.2.6', 2);
insert into properties (name, path, parent_id) VALUES ('G', '1.3.7', 3);

insert into reports (property_id, value) VALUES (1, 1);
insert into reports (property_id, value) VALUES (2, 4);
insert into reports (property_id, value) VALUES (3, 19);
insert into reports (property_id, value) VALUES (4, 21);
insert into reports (property_id, value) VALUES (5, 9);
insert into reports (property_id, value) VALUES (6, 11);
{% endhighlight %}

If you want more data to fill difference between queries speed you can use [my file](<https://gist.github.com/ignat-zakrevsky/10a4e2024b9911c5c15f>) with 4.5k+ records. (You need download it, connect to schema through `psql` and run `\i path_to_file`)
Let's build named paths for each node. Since the path column stores all parent ids, we can easily filter unrelated nodes with `like` queries.

{% highlight sql %}
# select '1.2.3' like '1.2.3%';   -- t
# select '1.2.3.4' like '1.2.3%'; -- t
# select '1.2.3.5' like '1.2.3%'; -- t
# select '1.2.4.5' like '1.2.3%'; -- f
{% endhighlight %}

As `path` type is `ltree` we need to cast it to varchar with `::varchar`, because you can't concatenate text and ltree path column. Put it all together.

{% highlight sql %}
select "properties"."id",
       "properties"."name",
       "properties"."path",
       "subproperties"."name" as "named_path"
  from "properties"
  left join "properties" as "subproperties"
    on ("properties"."path"::varchar) like ("subproperties"."path"::varchar || '%')
  group by 1, 2, 3, 4 order by 1;
{% endhighlight %}

{% highlight sql %}
 id | name | path  | named_path
----+------+-------+------------
  1 | A    | 1     | A
  2 | B    | 1.2   | A
  2 | B    | 1.2   | B
  3 | C    | 1.3   | A
  3 | C    | 1.3   | C
  4 | D    | 1.4   | A
  4 | D    | 1.4   | D
  5 | E    | 1.2.5 | A
  5 | E    | 1.2.5 | B
  5 | E    | 1.2.5 | E
  6 | F    | 1.2.6 | A
  6 | F    | 1.2.6 | B
  6 | F    | 1.2.6 | F
  7 | G    | 1.3.7 | A
  7 | G    | 1.3.7 | C
  7 | G    | 1.3.7 | G
(16 rows)
{% endhighlight %}

We have all node names, but to get a named path we need to concatenate them. We can achieve it with [string_agg](<http://www.postgresql.org/docs/devel/static/functions-aggregate.html>) aggregate function.

> `string_agg(expression, delimiter)` - input values concatenated into a string, separated by delimiter

Seems like what we need

{% highlight sql %}
select "properties"."id",
       "properties"."name",
       "properties"."path",
        string_agg("subproperties"."name", '->') as "named_path"
  from "properties"
  left join "properties" as "subproperties"
    on ("properties"."path"::varchar) like ("subproperties"."path"::varchar || '%')
  group by 1, 2, 3 order by 1;
{% endhighlight %}

{% highlight sql %}
 id | name | path  | named_path
----+------+-------+------------
  1 | A    | 1     | A
  2 | B    | 1.2   | A->B
  3 | C    | 1.3   | A->C
  4 | D    | 1.4   | A->D
  5 | E    | 1.2.5 | A->B->E
  6 | F    | 1.2.6 | A->B->F
  7 | G    | 1.3.7 | A->C->G
{% endhighlight %}

Looks correct. Let's try it against the production database.

{% highlight sql %}
 id |   name   |     path    |           named_path
----+----------+-------------+------------------------------
  1 | a450ceb3 | 1           | a450ceb3
  2 | 91b2f902 | 1.2         | a450ceb3->91b2f902
  3 | e4f25325 | 1.2.3       | a450ceb3->91b2f902->e4f25325
  4 | d5fdd885 | 1.2.4       | 91b2f902->d5fdd885->a450ceb3
  5 | 45160d77 | 1.2.5       | a450ceb3->91b2f902->45160d77
{% endhighlight %}

Look at record with id 4 — the named path starts from the second record, not the root. Let's read the documentation more carefully.

> This ordering is unspecified by default, but can be controlled by writing an `ORDER BY` clause within the aggregate call

{% highlight sql %}
select "properties"."id",
       "properties"."name",
       "properties"."path",
        string_agg("subproperties"."name", '->'
          order by "subproperties"."path") as "named_path"
  from "properties"
  left join "properties" as "subproperties"
    on ("properties"."path"::varchar) like ("subproperties"."path"::varchar || '%')
  group by 1, 2, 3 order by 1;
{% endhighlight %}

{% highlight sql %}
 id |   name   |     path    |           named_path
----+----------+-------------+------------------------------
  1 | a450ceb3 | 1           | a450ceb3
  2 | 91b2f902 | 1.2         | a450ceb3->91b2f902
  3 | e4f25325 | 1.2.3       | a450ceb3->91b2f902->e4f25325
  4 | d5fdd885 | 1.2.4       | a450ceb3->91b2f902->d5fdd885
  5 | 45160d77 | 1.2.5       | a450ceb3->91b2f902->45160d77
{% endhighlight %}

Got it! All works, but... We have ltree column while we are using like query. A-ha! We can use ltree `@>` operator.

> `ltree @> ltree` `boolean` is left argument an ancestor of right (or equal)?

Let's enable timing to feel difference between queries.

{% highlight bash %}
Operating System
  \timing [on|off]       toggle timing of commands (currently off)
{% endhighlight %}

Current query on table with 4644 records takes 8584,278 ms. Ok, what will we have by replacing `like` operator with ltree `@>`?

{% highlight sql %}
select "properties"."id",
       "properties"."name",
       "properties"."path",
        string_agg("subproperties"."name", '->' order by "subproperties"."path") as "named_path"
  from "properties"
  left join "properties" as "subproperties"
    on "subproperties"."path" @> "properties"."path"
  group by 1, 2, 3 order by 1;
{% endhighlight %}

Two times faster. Good! Let's see, what happens.

{% highlight sql %}
                                                                   QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=772405.45..772733.09 rows=4644 width=1104) (actual time=5143.003..5164.441 rows=4644 loops=1)
   ->  Sort  (cost=772405.45..772459.37 rows=21567 width=1104) (actual time=5142.988..5146.275 rows=22902 loops=1)
         Sort Key: properties.id, properties.name, properties.path
         Sort Method: external sort  Disk: 2520kB
         ->  Nested Loop Left Join  (cost=0.00..750063.00 rows=21567 width=1104) (actual time=0.018..5095.446 rows=22902 loops=1)
               Join Filter: (subproperties.path @> properties.path)
               Rows Removed by Join Filter: 21543834
               ->  Seq Scan on properties  (cost=0.00..103.44 rows=4644 width=556) (actual time=0.008..0.511 rows=4644 loops=1)
               ->  Seq Scan on properties subproperties  (cost=0.00..103.44 rows=4644 width=548) (actual time=0.001..0.364 rows=4644 loops=4644)
 Time: 3610,234 ms
{% endhighlight %}

It filters records with `@>` operator. But we can use `GiST` index for such operator to make it faster.

> GiST index over `ltree[]: ltree[] <@ ltree, ltree @> ltree[], @, ~, ?`

Ok, create index.

{% highlight sql %}
CREATE INDEX gist_path_index ON "properties" USING gist("path");
{% endhighlight %}

And let's try again.

{% highlight sql %}
                                                                           QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=24341.43..24669.07 rows=4644 width=1104) (actual time=148.255..169.519 rows=4644 loops=1)
   ->  Sort  (cost=24341.43..24395.35 rows=21567 width=1104) (actual time=148.212..151.562 rows=22902 loops=1)
         Sort Key: properties.id, properties.name, properties.path
         Sort Method: external sort  Disk: 2520kB
         ->  Nested Loop Left Join  (cost=0.15..1998.98 rows=21567 width=1104) (actual time=0.033..98.501 rows=22902 loops=1)
               ->  Seq Scan on properties  (cost=0.00..103.44 rows=4644 width=556) (actual time=0.007..0.509 rows=4644 loops=1)
               ->  Index Scan using gist_path_index on properties subproperties  (cost=0.15..0.36 rows=5 width=548) (actual time=0.013..0.020 rows=5 loops=4644)
                     Index Cond: (path @> properties.path)
 Total runtime: 177.499 ms
{% endhighlight %}

Wow! Instead of filtering records it's just using our index. It's 20 times faster than before. Keep it so.

Time goes on and you have new task. You need to find sum of each children for each node. Seems like not big problem, just little modify our previous query.

{% highlight sql %}
select "properties"."id",
       "properties"."name",
       "properties"."path",
        sum("reports"."value")
  from "properties"
  left join "properties" as "subproperties"
    on "subproperties"."path" @> "properties"."path"
  inner join "reports"
    on "reports"."property_id" = "subproperties"."id"
  group by 1, 2, 3 order by 1;
{% endhighlight %}

{% highlight sql %}
 id | path  | name | sum
----+-------+------+-----
  1 | 1     | A    |   1
  2 | 1.2   | B    |   5
  3 | 1.3   | C    |  20
  4 | 1.4   | D    |  22
  5 | 1.2.5 | E    |  14
  6 | 1.2.6 | F    |  16
  7 | 1.3.7 | G    |  20
{% endhighlight %}

No problems. 115,062 ms on 4.5k+ records. One day later you need both data, sum and named path. Argh! Roll up your sleeves.

{% highlight sql %}
select "properties"."id",
       "properties"."name",
       "properties"."path",
        sum("reports"."value"),
        string_agg("subproperties"."name", '->' order by "subproperties"."path") as "named_path"
  from "properties"
  left join "properties" as "subproperties"
    on "subproperties"."path" @> "properties"."path"
  inner join "reports"
    on "reports"."property_id" = "subproperties"."id"
  group by 1, 2, 3 order by 1;
{% endhighlight %}

{% highlight sql %}
 id | path  | name | named_path | sum
----+-------+------+------------+-----
  1 | 1     | A    | A          |   1
  2 | 1.2   | B    | A->B       |   5
  3 | 1.3   | C    | A->C       |  20
  4 | 1.4   | D    | A->D       |  22
  5 | 1.2.5 | E    | A->B->E    |  14
  6 | 1.2.6 | F    | A->B->F    |  16
  7 | 1.3.7 | G    | A->C->G    |  20
{% endhighlight %}

Exactly what we need. But let's look a little deeper at what's happening.

{% highlight sql %}
                                                                                 QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=24954.00..25332.63 rows=4644 width=1108) (actual time=102.756..125.912 rows=4644 loops=1)
   ->  Sort  (cost=24954.00..25007.43 rows=21372 width=1108) (actual time=102.738..106.177 rows=22901 loops=1)
         Sort Key: properties.id, properties.name, properties.path
         Sort Method: external sort  Disk: 2648kB
         ->  Nested Loop  (cost=352.44..2668.99 rows=21372 width=1108) (actual time=1.949..56.588 rows=22901 loops=1)
               ->  Merge Join  (cost=352.29..790.59 rows=4602 width=552) (actual time=1.924..5.101 rows=4643 loops=1)
                     Merge Cond: (subproperties.id = reports.property_id)
                     ->  Index Scan using properties_pkey on properties subproperties  (cost=0.28..357.94 rows=4644 width=556) (actual time=0.015..0.737 rows=4644 loops=1)
                     ->  Sort  (cost=352.01..363.51 rows=4602 width=8) (actual time=1.905..2.367 rows=4644 loops=1)
                           Sort Key: reports.property_id
                           Sort Method: quicksort  Memory: 410kB
                           ->  Seq Scan on reports  (cost=0.00..72.02 rows=4602 width=8) (actual time=0.009..0.961 rows=4644 loops=1)
               ->  Index Scan using gist_path_index on properties  (cost=0.15..0.36 rows=5 width=556) (actual time=0.009..0.010 rows=5 loops=4643)
                     Index Cond: (subproperties.path @> path)
 Total runtime: 132.957 ms
(15 rows)
{% endhighlight %}

`loops=4643` — a self join processes N² records. We need something else, and PostgreSQL has it: [WITH Queries](<http://www.postgresql.org/docs/9.4/static/queries-with.html>). Pay attention to the `RECURSIVE` keyword.

> The optional `RECURSIVE` modifier changes `WITH` from a mere syntactic convenience into a feature that accomplishes things not otherwise possible in standard SQL. Using `RECURSIVE`, a `WITH` query can refer to its own output. The general form of a recursive `WITH` query is always a non-recursive term, then `UNION` (or `UNION ALL`), then a recursive term, where only the recursive term can contain a reference to the query's own output

We have a tree, we have roots that can be used as initial term and children that can be traversed recursive.

{% highlight sql %}
with recursive nodes("id", "name", "path", tree_sum, named_path) as (
  select roots."id", roots."name", roots."path", "reports"."value" as tree_sum, cast (roots."name" AS varchar(255)) as named_path
    from properties roots
    inner join reports on ("property_id" = roots."id")
    where (roots."parent_id" is NULL)
  union all
  select children."id", children."name", children."path", (tree_sum + "reports"."value"), cast (nodes.named_path ||'->'|| children."name" AS varchar(255))
    from properties children
    inner join nodes on (nodes."id"= children."parent_id")
    inner join reports on (reports."property_id" = children."id")
  )
  select * from nodes order by 1;
{% endhighlight %}

{% highlight sql %}
 id | name | path  | tree_sum | named_path
----+------+-------+----------+------------
  1 | A    | 1     |        1 | A
  2 | B    | 1.2   |        5 | A->B
  3 | C    | 1.3   |       20 | A->C
  4 | D    | 1.4   |       22 | A->D
  5 | E    | 1.2.5 |       14 | A->B->E
  6 | F    | 1.2.6 |       16 | A->B->F
{% endhighlight %}

{% highlight sql %}
                                                                               QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=66902.48..67034.87 rows=52953 width=1076) (actual time=31.521..32.130 rows=4643 loops=1)
   Sort Key: nodes.id
   Sort Method: external sort  Disk: 576kB
   CTE nodes
     ->  Recursive Union  (cost=103.73..11729.62 rows=52953 width=1080) (actual time=0.817..26.962 rows=4643 loops=1)
           ->  Hash Join  (cost=103.73..193.29 rows=23 width=560) (actual time=0.814..2.210 rows=48 loops=1)
                 Hash Cond: (reports.property_id = roots.id)
                 ->  Seq Scan on reports  (cost=0.00..72.02 rows=4602 width=8) (actual time=0.011..0.675 rows=4644 loops=1)
                 ->  Hash  (cost=103.44..103.44 rows=23 width=556) (actual time=0.788..0.788 rows=48 loops=1)
                       Buckets: 1024  Batches: 1  Memory Usage: 3kB
                       ->  Seq Scan on properties roots  (cost=0.00..103.44 rows=23 width=556) (actual time=0.008..0.775 rows=48 loops=1)
                             Filter: (parent_id IS NULL)
                             Rows Removed by Filter: 4596
           ->  Hash Join  (cost=359.76..1047.73 rows=5293 width=1080) (actual time=1.000..3.967 rows=766 loops=6)
                 Hash Cond: (children.parent_id = nodes_1.id)
                 ->  Merge Join  (cost=352.29..790.59 rows=4602 width=564) (actual time=0.273..3.054 rows=4643 loops=6)
                       Merge Cond: (children.id = reports_1.property_id)
                       ->  Index Scan using properties_pkey on properties children  (cost=0.28..357.94 rows=4644 width=560) (actual time=0.006..0.638 rows=4644 loops=6)
                       ->  Sort  (cost=352.01..363.51 rows=4602 width=8) (actual time=0.266..0.557 rows=4644 loops=6)
                             Sort Key: reports_1.property_id
                             Sort Method: quicksort  Memory: 410kB
                             ->  Seq Scan on reports reports_1  (cost=0.00..72.02 rows=4602 width=8) (actual time=0.004..0.807 rows=4644 loops=1)
                 ->  Hash  (cost=4.60..4.60 rows=230 width=528) (actual time=0.216..0.216 rows=774 loops=6)
                       Buckets: 1024  Batches: 1  Memory Usage: 207kB
                       ->  WorkTable Scan on nodes nodes_1  (cost=0.00..4.60 rows=230 width=528) (actual time=0.001..0.094 rows=774 loops=6)
   ->  CTE Scan on nodes  (cost=0.00..1059.06 rows=52953 width=1076) (actual time=0.820..29.086 rows=4643 loops=1)
 Total runtime: 36.068 ms
{% endhighlight %}

4x time faster. And it will not slow down with table growing, it only depends on tree height.

What is the point? You need to use suitable instruments. If you need traverse tree -- use `WITH RECURSIVE`, if you need to work with single node relations -- `ltree` is what you need.
