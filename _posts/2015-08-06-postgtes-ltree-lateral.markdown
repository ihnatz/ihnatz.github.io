---
layout: post
title:  "Finding Leaf Nodes with ltree"
date:   2015-08-06 22:20:35
tags: postgres ltree lateral
---

Let's assume you want to find all node leaves. To keep it simple for all examples we will use DB from previous post. Leaf is a node without children. If you have `parent_id` it seems easy to find all leaves

{% highlight sql %}
SELECT * FROM "properties" WHERE "id" NOT IN
  (SELECT DISTINCT "parent_id" FROM "properties" WHERE "parent_id" IS NOT NULL);
{% endhighlight %}

We just excluding all nodes that are parents for any node and that's it.
But what if you are prefer to use pure `ltree` without `parent_id`? And here come troubles.
All that I can find by googling this problem is how to find node with most children count:

{% highlight sql %}
SELECT subpath("path", 0, 1), count(*)
  FROM "properties"
  GROUP BY 1 ORDER BY 2 DESC;
{% endhighlight %}

And to be honest, I envy whoever invented it. Pretty simple: with `subpath` we retrieve the root element for each node, group by it, and count how many such elements exist. But that is not our problem.

As I said before, leaf is a node without children. We can find all children nodes with `ltree <@ ltree`.

> `ltree <@ ltree`  `boolean` is left argument a descendant of right (or equal)?

It will return all children and self. As leaf have zero children and self it will be one result for leaf ancestors. As we have statement for filtering leaves we need to check each node. As I just learned `LATERAL` queries my first thought was to use it. From documentation:

> When a FROM item contains LATERAL cross-references, evaluation proceeds as follows: **for each row** of the FROM item providing the cross-referenced column(s)

Each row, see?

{% highlight sql %}
SELECT * FROM (
  SELECT "path", "children_count" FROM "properties",
  LATERAL (
    SELECT COUNT("path") AS "children_count"
    FROM  "properties" AS "subproperties"
    WHERE "properties"."path" @> "subproperties"."path"
  ) AS "subproperties"
) AS "path_with_children" WHERE "path_with_children"."children_count" = 1
GROUP BY 1, 2;
{% endhighlight %}

But it seems that there are too much overhead. Something is wrong here. Let's try to use `INNER JOIN` with statement instead of additional `WHERE`.

{% highlight sql %}
SELECT * FROM "properties"
  INNER JOIN LATERAL (
    SELECT COUNT("path") AS "children_count"
    FROM  "properties" AS "subproperties"
    WHERE "properties"."path" @> "subproperties"."path"
  ) AS "path_with_children" ON "path_with_children"."children_count" = 1 ;
{% endhighlight %}

Seems better but still complex. Wait, is it really necessary to calculate table and then filter values from it? Seems like we can just filter each node with simple query.

{% highlight sql %}
SELECT count("path") = 1 FROM "properties" WHERE "path" <@ '1'; -- f
SELECT count("path") = 1 FROM "properties" WHERE "path" <@ '1.4'; -- t
SELECT count("path") = 1 FROM "properties" WHERE "path" <@ '1.2.3'; -- f
{% endhighlight %}

And apply it for each node:

{% highlight sql %}
SELECT * FROM "properties" AS "main" WHERE (
  SELECT count("path") = 1 FROM "properties" WHERE "path" <@ "main"."path");
{% endhighlight %}

Simple and elegant way, what we need!

When I showed it to [Andrei](<https://github.com/sjke>), he noticed that it will not work for nodes with the same path. And if it has not so much sense for `path` with ids it's normal for `path` with hierarchical structures.

To reproduce this bug on our data we will add node with same path but another value.

{% highlight sql %}
insert into properties (name, path, parent_id) VALUES ('H', '1.3.7', 9);
{% endhighlight %}

{% highlight sql %}
 id | name | path  | parent_id
----+------+-------+-----------
  4 | D    | 1.4   |         1
  5 | E    | 1.2.5 |         2
  6 | F    | 1.2.6 |         2
{% endhighlight %}

We missed two leaves with path `1.3.7` because `<@` operator will return all ancestors including self and nodes with same paths. To fix it we just need to add `DISTINCT` keyword in right place.

{% highlight sql %}
SELECT * FROM "properties" AS "main" WHERE (
  SELECT count(DISTINCT "path") = 1 FROM "properties" WHERE "path" <@ "main"."path");
{% endhighlight %}

{% highlight sql %}
 id | name | path  | parent_id
----+------+-------+-----------
  4 | D    | 1.4   |         1
  5 | E    | 1.2.5 |         2
  6 | F    | 1.2.6 |         2
  7 | G    | 1.3.7 |         3
  8 | H    | 1.3.7 |         9
{% endhighlight %}

As a consequence: if your solution is too complex, it's usually not the right one.

_P.S. If you want to see nice example of using `LATERAL` keyword you can check [this](<http://blog.heapanalytics.com/postgresqls-powerful-new-join-type-lateral/>) article._