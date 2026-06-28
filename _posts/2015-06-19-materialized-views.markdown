---
layout: post
title:  "Materialized Views and Concurrent Refresh"
date:   2015-06-19 17:49:35
tags: rails materialized views
---

Materialized view is an object that contains the query's results. Unlike a database table it doesn't support `INSERT`/`UPDATE`/`DELETE` operations. Since these operations are unsupported, to update a materialized view you need to call a refresh operation. In PostgreSQL materialized views support was introduced in version 9.3.
From [PostgreSQL documentation](<http://www.postgresql.org/docs/9.3/static/sql-creatematerializedview.html>) you can see how to create materialized view. So, you need query that you want to materialize and... that all.

Let's assume you have next structure: `books`, `authors` and `feedbacks`. Each `author` has many `books`, each `book` has many `feedbacks`

{% highlight ruby %}
# author.rb
class Author < ActiveRecord::Base
  has_many :books
  has_many :feedbacks, through: :books
end
{% endhighlight %}

{% highlight ruby %}
# book.rb
class Book < ActiveRecord::Base
  belongs_to :author
  has_many :feedbacks
end
{% endhighlight %}

{% highlight ruby %}
# feedback.rb
class Feedback < ActiveRecord::Base
  belongs_to :book
end
{% endhighlight %}

And let's assume you need to we need to show top authors by their feedbacks. We have Rails `ActiveRecord` and it seems easy

{% highlight ruby %}
Author.joins(:feedbacks, :books)
      .group("authors.name")
      .order("sum(feedbacks.mark) DESC")
      .sum("feedbacks.mark")
# {"Alexander Pushkin"=>360, "Mikhail Lermontov"=>330, "Nikolai Gogol"=>180}
{% endhighlight %}

But three `INNER JOINS`, isn't too much? Let's use our materialized views! Starting from migration:

{% highlight ruby %}
class CreateAuthorsFeedbacks < ActiveRecord::Migration
  def self.up
    query = %Q{
      CREATE MATERIALIZED VIEW authors_feedbacks  AS
      #{Author.joins(:feedbacks, :books)
          .select("sum(feedbacks.mark)", "authors.name", "authors.id as author_id")
          .group("authors.name", "authors.id")
          .order("sum(feedbacks.mark) DESC")
          .to_sql};
    }
    execute query
    add_index :authors_feedbacks, :author_id, unique: true
  end

  def self.down
    execute <<-SQL
      DROP MATERIALIZED VIEW IF EXISTS authors_feedbacks;
    SQL
  end
end
{% endhighlight %}

To be honest, I'm not a big fan of using Rails `to_sql` method and so on and prefer to write pure SQL for such migrations. But it's sample and we will keep it so.
Each selected column will be `materialized view` column, that's why we used `as` for `authors.id`, in our table "authors.id" will be stored in "author_id" column. Unlike simple views, we can index any materialized view column, additionaly, we will make it index unique. As I said before, to actualize data in view we need to call refresh view. In pure `PostgreSQL` it will be:

{% highlight sql %}
REFRESH MATERIALIZED VIEW authors_feedbacks
{% endhighlight %}

But in `PostgreSQL` 9.4 we can do it concurrently! It will refresh the materialized view without locking out concurrent selects on the materialized view, but... You need uniq index on materialized view for this. Wait a minute, we alredy have one!

{% highlight sql %}
REFRESH MATERIALIZED VIEW CONCURRENTLY authors_feedbacks
{% endhighlight %}

As I can see, concurrently refreshing much more quick if your index very simple. With complex index it can be even slower then unconcurrent refreshing. As it behave like table we can even use it as `ActiveRecord` model.

{% highlight ruby %}
# authors_feedback.rb
class AuthorsFeedback < ActiveRecord::Base
  include ReadOnlyModel

  belongs_to :author

  # Refresh materialized view by reaggregating data from each connected table
  def self.refresh_view
    connection = ActiveRecord::Base.connection
    connection.execute("REFRESH MATERIALIZED VIEW #{table_name}")
  end

  # Concurrently refresh materialized view by reaggregating data from each
  # connected table
  def self.refresh_view_concurrently
    connection = ActiveRecord::Base.connection
    connection.execute("REFRESH MATERIALIZED VIEW CONCURRENTLY #{table_name}")
  end
end
{% endhighlight %}

As I said before, materialized view doesn't support any editing operations, for this we will use concern `ReadOnlyModel`

{% highlight ruby %}
module ReadOnlyModel
  extend ActiveSupport::Concern

  included do
    # Required to block update_attribute and update_column
    attr_readonly(*column_names)
  end

  def readonly?
    # Does not block destroy or delete
    true
  end

  def destroy
    raise ActiveRecord::ReadOnlyRecord
  end

  def delete
    raise ActiveRecord::ReadOnlyRecord
  end
end
{% endhighlight %}

And we need to add association for authors

{% highlight ruby %}
class Author < ActiveRecord::Base
  has_many :books
  has_many :feedbacks, through: :books
  has_one  :authors_feedback
end
{% endhighlight %}

And that's all, just use it.

{% highlight ruby %}
Author.last.authors_feedback
# => #<AuthorsFeedback sum: 180, name: "Nikolai Gogol", author_id: 3>
AuthorsFeedback.all
#=> #<ActiveRecord::Relation [
  #<AuthorsFeedback sum: 360, name: "Alexander Pushkin", author_id: 1>,
  #<AuthorsFeedback sum: 330, name: "Mikhail Lermontov", author_id: 2>,
  #<AuthorsFeedback sum: 180, name: "Nikolai Gogol", author_id: 3>
#]>
{% endhighlight %}

P.S. Don't forget to change your dump format, `schema.rb` can't store materialized view structure, so, you need to store all this in SQL. Change in `application.rb`
{% highlight ruby %}
config.active_record.schema_format = :sql
{% endhighlight %}
