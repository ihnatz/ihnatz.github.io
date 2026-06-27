---
layout: post
title:  "PostgreSQL HTTP API"
date:   2015-05-30 22:12:35
categories: postgres api postgrest
published: false
---

I’m a huge PostgreSQL fun and when I saw post about [MySQL HTTP API](<http://ilyabylich.svbtle.com/experimental-mysql-http-api-and-ruby>) I was very impressed. So, I decided to find something similar for PostgreSQL and it's [here](<https://wiki.postgresql.org/wiki/HTTP_API>). At the post buttom I find link to [PostgresREST](<https://github.com/begriffs/postgrest>). Let's play with it.
![Alt text](https://github.com/begriffs/postgrest/raw/master/static/logo.png)

Some words about **postgrest**. From github page you can see, that it's fully written in Haskell and uses Warp  server which is [one of the most powerfull web-servers](<http://www.yesodweb.com/static/benchmarks/2011-03-17/extra-large.png>). More, Warp compiles all sources into single executable file (now, you wiln't so impressed by content of **postgrest** archive).

# Installation

To install **postgrest** you need to download curresponding version from github page (current version [here](<https://github.com/begriffs/postgrest/releases/tag/v0.2.9.0>)) and simply extract it from package. As I said before, Warp compiles all files into single executable file, so, you need just to start it.

Let's start **postgrest** server

{% highlight bash %}
./postgrest-0.2.9.0  --db-host localhost  --db-port 5432     \
    --db-name engine_development          --db-user postgres \
    --db-pass secret_password             --db-pool 200      \
    --anonymous postgres --port 3000                         \
    --v1schema public
{% endhighlight %}

# Prepare PostgreSQL

We are starting from creating simple table named `beer` using PostgreSQL client:

{% highlight sql %}
CREATE TABLE beer (
    id bigserial primary key,
    name varchar(256) NOT NULL,
    description text NOT NULL,
    alcohol real NOT NULL
);
{% endhighlight %}

Let's see what PostgreSQL generated for us

{% highlight sql %}
=# \d+ beer;
                                                       Table "public.beer"
   Column    |          Type          |                     Modifiers                     | Storage  | Stats target | Description
-------------+------------------------+---------------------------------------------------+----------+--------------+-------------
 id          | bigint                 | not null default nextval('beer_id_seq'::regclass) | plain    |              |
 name        | character varying(256) | not null                                          | extended |              |
 description | text                   | not null                                          | extended |              |
 alcohol     | real                   | not null                                          | plain    |              |
Indexes:
    "beer_pkey" PRIMARY KEY, btree (id)
Has OIDs: no
{% endhighlight %}

# Playing with postgrest
Let's make simple request for recrods list

{% highlight bash %}
curl --ipv4
     --url http://localhost:3000/beer
{% endhighlight %}

{% highlight json %}
[
{"id":1, "name":"Krinitsa Porter",   "description":"Dark beer with ...",            "alcohol":6.8},
{"id":2, "name":"Druzya Statut 1529","description":"Named in honor of ...",         "alcohol":4.9},
{"id":3, "name":"Alivariya Porter",  "description":"Special sort of dark beer ...", "alcohol":6.8},
{"id":4, "name":"Žatecký Gus Černý", "description":"Zatecky Gus is a lager ...",    "alcohol":3.5}
]
{% endhighlight %}

Ok, what about `WHERE` queries?

{% highlight bash %}
curl --ipv4
     --url http://localhost:3000/beer\?alcohol\=lt.4.0\&alcohol\=gt.3.2
{% endhighlight %}

It works fine:

{% highlight json %}
[{"id":4,"name":"Žatecký Gus Černý","description":"Zatecky Gus (Zhatetsky goose) is a light lager with a light traditional flavor. This beer is brewed on the original recipe with addition of famous Czech aromatic hop of Zhatetsky variety. This hop gives the beer its specific aroma and slight bitterness. The new beer drinks easily. Zatecky Gus is a unique opportunity to enjoy the beer, which is not expensive and brewed in the best Czech traditions. The Zhatetsky hop has been grown in the area of Zhatets town in northern Czechia for more than 700 years. From the old times the hops has been famous not only in Czechia, but as well in the neighbouring regions. Zhatets is often called the world hop capital — Zhatetsky hop is one of the highest-quality hops in the world, it has been grown by the Institute of Hop growing.","alcohol":3.5}]%
{% endhighlight %}

Hm, what about `INSERT` queries?

{% highlight bash %}
curl --ipv4
     -H "Content-Type: application/json"
     -X POST
     -d '{"name":"Žatecký Gus", "description":"Zatecky Gus (Zhatetsky goose)..."}'
     http://localhost:3000/beer
{% endhighlight %}

Let's see result of it.  **postgrest** supports some kind of pagination with using `Range` headers ([some words about it](<http://begriffs.com/posts/2014-03-06-beyond-http-header-links.html>)). Is it interesting? Yes. Is it conventional? Hm, not for me

{% highlight bash %}
curl  --header "Range: 0-0"
      --ipv4
      --url http://localhost:3000/beer\?order\=id.desc
{% endhighlight %}

It will provide query like:
{% highlight sql %}
SELECT * FROM beer ORDER BY id desc LIMIT 1;
{% endhighlight %}

Result:
{% highlight json %}
[{"id":5,"name":"Žatecký Gus","description":"Zatecky Gus (Zhatetsky goose) is ..."}]
{% endhighlight %}

You can find more about statements [here](<https://github.com/begriffs/postgrest/wiki/Routing>)

# Using
How you can use it? As example, for [this](<https://github.com/marmelab/ng-admin>). It provides AngularJS admin GUI to any RESTful API. In our case -- for PostgreSQL. You can see how it works [here](http://marmelab.com/ng-admin-postgrest/) (source [here](https://github.com/marmelab/ng-admin-postgrest)).

**postgrest** uses a little strange API that can't be used out of box with any existing libraries for it (such ruby `her` or `activesupport`). You need to write your wrapper and it will increase development time. From the another side there is no any need in such libraries for backend languages, we just need JS version. As we can see from [ng-admin-postgrest source code](https://github.com/marmelab/ng-admin-postgrest/blob/master/main.js), **postgrest** can be used with [Restangular](<https://github.com/mgonto/restangular>) but there is no separated ready-to-use library.

# Conclusion
**postgrest** is interesting tool for providing PostgreSQL REST API. It seems cool for some simple applications. With PostgreSQL views and materialized views it provides a wide range of possibilities but still experimental.

_P.S. It's interesting tendention with backend's thinning and using frameworks such as Angular or Ember. Will it replace some part of simple rails apps? Time will tell._
