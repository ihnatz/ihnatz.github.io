---
layout: post
title:  "Ruby Natural Language"
date:   2015-09-07 09:09:31
tags: ruby natural
published: false
---

In Martin Fowler's book "Refactoring: Improving the Design of Existing Code" you can read nice statement:

>  Any fool can write code that a computer can understand. Good programmers write code that humans can understand.

Ruby provides a lot of abilities to write readable code. A lot of principles you can retrieve from best practices, for example iterating with `each` instead of construction `for .. in`.

Compare:

{% highlight ruby %}
for box in boxes
   # do something
end
# and
boxes.each do |box|
  # do something
end
{% endhighlight %}

In natural language you will never use such structure, when you need to interact with each object you are using `each`. When you are working with some business logic you should use this logic terms.

{% highlight ruby %}
class Year
  attr_reader :value

  def initialize(value)
    @value = value
  end
end

puts "Leap" if year.value % 4 == 0 && (!(year.value % 100 == 0) || year.value % 400 == 0)
{% endhighlight %}

It's unreadable. With a little changes

{% highlight ruby %}
class Year
  attr_reader :value

  def initialize(value)
    @value = value
  end

  def leap?
    divisble_by?(4) && (!divisble_by?(100) || divisble_by?(400))
  end

  private

  def divisble_by?(divisor)
    (value % divisor) == 0
  end
end

puts "Leap" if year.leap?
{% endhighlight %}

It should be obviously. But you can go even more deeply. Ruby and Rails `ActiveSupport` provides you a lot of syntax sugar that you can use in your code.

Some examples. You need to check is such color in your colors collection. Compare

{% highlight ruby %}
# default way
["red", "black", "green"].include?("brown")
# natural way
"brown".in? ["red", "black", "green"]
{% endhighlight %}

Or you need to collect all boxes colors:

{% highlight ruby %}
# default way
boxes.map(&:color)
# natural way
boxes.collect(&:color)
{% endhighlight %}

See? It's break ruby best practices but makes code more readable and natural.

And about `ActiveSupport` inquiry strings. When did you last time used `Rails.env == "production"`? I hope never :).




