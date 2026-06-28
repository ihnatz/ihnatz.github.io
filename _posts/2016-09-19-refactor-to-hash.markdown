---
layout: post
title:  "Refactoring from case statement to lazy hash"
date:   2016-09-19 18:11:33
tags: ruby lambda refactoring
published: false
---

There are a lot of cases when you need to implement logic with complex case statements.
When you are using case statements it causes a lot of problems, rubocop cries about complexity, you need to store somewhere your keys, it's hard to test and etc.

{% highlight ruby %}
KEYS = %w(a b c)
case x
when "a" then puts 1
when "b" then puts 2
when "c" then puts 3
else raise("can't define proper case")
end
{% endhighlight %}

So, what can you do with this? I prefer to convert case statements to hashes. It's perfectly works for simple values, which you can simply calculate

{% highlight ruby %}
puts({
"a" => 1,
"b" => 2,
"c" => 3
}[x] || raise("can't define proper case"))
{% endhighlight %}

But, no, when you really need something complex it sucks. And here lambdas come:

{% highlight ruby %}
{
  "a" => -> { puts 1 },
  "b" => -> { puts 2 },
  "c" => -> { puts 3 }
}.fetch(x) { raise("can't define proper case") }.call
{% endhighlight %}

And to make it really superusable

{% highlight ruby %}
MAPPING = {
  "a" => -> { puts 1 },
  "b" => -> { puts 2 },
  "c" => -> { puts 3 }
}
MAPPING.fetch(x) { raise("can't define proper case") }.call
{% endhighlight %}

Rubocop happy and you get `MAPPING.keys` as bonus for your views or smth like it. Additionally you can use `MAPPING` as default argument for your class and then inject proper test mapping in tests.
