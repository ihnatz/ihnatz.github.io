---
layout: post
title:  "Fluent interface in Ruby"
date:   2015-11-08 17:07:12
categories: ruby object-oriented-api
published: false
---

Let's imagine that you want to write your implementation of Rails `where` method. What for you need it? For example, if you have old RoR 2 project and you have no any wish to write where statements in pure SQL. What are the requirements of such service? You should have ability to chain `where` statements and transform all this into conditions for Rails 2 ActiveRecord.

You can see example of Ruby on Rails 2 ActiveRecord [here](<http://guides.rubyonrails.org/v2.3.11/active_record_querying.html#array-conditions>). Example:

{% highlight ruby %}
Client.first(:conditions =>
  ["orders_count = ? AND locked = ?", params[:orders], false])
{% endhighlight %}

What can help us? [Fluent interface](<https://en.wikipedia.org/wiki/Fluent_interface>). The main idea of this pattern is to return operating object on each method call.

For out implementation we will use two arrays, `conditions` and `arguments` to store corrseponding values. Additionaly we need a grammatical conjunction `keyword` to concatenate statements.

{% highlight ruby %}
def initialize(keyword = "AND")
  @keyword = keyword

  @conditions = []
  @arguments  = []
end
{% endhighlight %}

Then we need `where` method that will collect our conditions. As I said before there is no any magick, just store fields, arguments and return self.

{% highlight ruby %}
def where(condition, argument)
  return self if argument.nil?
  @conditions << "#{condition} = (?)"
  @arguments  << argument
  self
end
{% endhighlight %}

When we have all this we need to transform conditions into RoR Array Condition. We will join our arguments with keyword.

{% highlight ruby %}
def prepare
  return [] if @conditions.empty?
  [contatinate_statements, *@arguments]
end

private
def contatinate_statements
  @conditions.join(" #{@keyword} ")
end
{% endhighlight %}

And that's all. Full code

{% highlight ruby %}
class CollectionFilteringService
  def initialize(keyword = "AND")
    @keyword = keyword

    @conditions = []
    @arguments  = []
  end

  def where(condition, argument)
    @conditions << "#{condition} = (?)"
    @arguments  << argument
    self
  end

  def prepare
    return [] if @conditions.empty?
    [contatinate_statements, *@arguments]
  end

  private
  def contatinate_statements
    @conditions.join(" #{@keyword} ")
  end
end
{% endhighlight %}

And example of how it works.

{% highlight ruby %}
conditions = CollectionFilteringService
  .where(:field1, :value1)
  .where(:field2, :value2)
  .prepare
Client.first(:conditions => conditions)
{% endhighlight %}

You can easily add any kind of method that you want. For example if you are using MicrosoftSQL and you need to receive fields by date even if it stored in datetime you can realize your `#where_date` method. As MicrosoftSQL make some internal magick with dates escaping we will use a little trick and pass arguments directly:

{% highlight ruby %}
def where_date(field, argument)
  @conditions << "datediff(day, #{field}, '#{argument}') = 0"
  self
end
{% endhighlight %}

And use it. If you have a lot of patience, with this technic in mind you can reailise full analog of ActiveRecord. And don't forget to cover all this with tests.

{% highlight ruby %}
require 'minitest/autorun'
require_relative 'collection_filtering_service'

describe CollectionFilteringService do
  subject { CollectionFilteringService.new }

  it "transform where condition into SQL query" do
    assert_equal subject.where(:field1, :value).prepare,
      ["field1 = (?)", :value]
  end

  it "transform where_date condition into SQL query" do
    assert_equal subject.where_date(:datefield, :value).prepare,
      ["datediff(day, datefield, 'value') = 0"]
  end

  it "allows to chain `where` statements" do
    assert_equal subject.where(:field1, :value1).where(:field2, :value2).prepare,
      ["field1 = (?) AND field2 = (?)", :value1, :value2]
  end

  it "allows to chain `where_date` statements" do
    assert_equal subject.where(:field1, :value1).where_date(:datefield, :value).prepare,
      ["field1 = (?) AND datediff(day, datefield, 'value') = 0", :value1]
  end

  it "return empty array for query withoud conditions" do
    assert subject.prepare.empty?
  end
end
{% endhighlight %}