---
layout: post
title:  "Lazy piping ES6 generators"
date:   2015-11-25 02:51:33
categories: javascript es6 generators
published: false
---

If you familiar with languages like Elixir you should now about lazy steams. Such abstraction gives you ability to modify enless steams without calculating results on each step. For example

{% highlight elixir %}
(1..1000000) |> Stream.map(fn(x) -> x * 2 end) |> Enum.take 5
# [2, 4, 6, 8, 10]
{% endhighlight %}

Today I was learning about JS generators and I was surprised that there is no such powerfull functionality in JS. Let's try to fix it and provide some kind of lazy piping on ES6. It will be a little hard for me as I'm nore Ruby developer but any way.

As we love TDD let's prepare our enviroment. I will use [jasmine-node](<https://github.com/mhevery/jasmine-node>) for running tests.
Create file `main-spec.js` (or something like it, it's just a draft). Spec suffix used to say `jasmine-node` to use for testing.


{% highlight javascript %}
'use strict'

describe("tests", function() {
  it("works", function() {
    expect(true).toBe(true);
  });
});
{% endhighlight %}

and run it

{% highlight bash %}
jasmine-node main-spec.js
{% endhighlight %}

{% highlight bash %}
.

Finished in 0.006 seconds
1 test, 1 assertion, 0 failures, 0 skipped
{% endhighlight %}

Nice start. To automize this process let's add post save hook for vim that will start tests after each file saving.


{% highlight bash %}
:autocmd BufWritePost * !jasmine-node %
{% endhighlight %}

Now we need write tests for our `fibonacci` generator. For our aims there is no any difference what kind of generator to use but just use simple counter is too boring.

{% highlight javascript %}
describe("fibonacci", function() {
  it("returns one as result on first and second call", function() {
    const subject = fibonacci();
    expect(subject.next().value).toBe(1);
    expect(subject.next().value).toBe(1);
  });

  it("all elements after first two is sum of two previous", function() {
    const subject = fibonacci();
    subject.next();
    subject.next();
    expect(subject.next().value).toBe(2);
    expect(subject.next().value).toBe(3);
    expect(subject.next().value).toBe(5);
  });
});
{% endhighlight %}

As now our tests don't work we need to provide implementation for `fibonacci` generator. Let's try with such example

{% highlight javascript %}
function *fibonacci() {
  let firstValue = 1;
  let secondValue = 1;
  while(true) {
    [firstValue, secondValue] = [secondValue, firstValue + secondValue];
    yield firstValue;
  }
};
{% endhighlight %}

It's a little dirty variant but it's very simple and passes out tests. To modify our sequence we need some function. We can use anonymous function but...

{% highlight javascript %}
describe("x2", function() {
  it("multipy passed value to 2", function() {
    expect(x2(25)).toBe(50);
  });
});
{% endhighlight %}

Ok, seems we have all to formulate requirements for our idea. Let's call wrapper for generators LazyGen and define scpes for it.

{% highlight javascript %}
describe("LazyGen", function() {
  it("allows you to chain methods for generators", function() {
    const subject = LazyGen(fibonacci).map(x2);
    expect(subject.next().value).toBe(2);
    expect(subject.next().value).toBe(2);
    expect(subject.next().value).toBe(4);
  });
});
{% endhighlight %}

So, as chaining it's about states we need some object to store it. LazyGen will just return such object and will be not more that syntax sugar.

{% highlight javascript %}
const LazyGen = function(generator) {
  return new LazyObject(generator);
};
{% endhighlight %}

Now we need wrapper for our generator.

{% highlight javascript %}
function LazyObject(generator) {
  const runnedGenerator = generator();
  return this;
};
{% endhighlight %}

There we will store our generator. For providing ability to chain we need to return `this` on each LazyObject method call. We wil l use same techinq as in previous [post](</ruby/object-oriented-api/2015/11/08/fluent-interface.html>) Currently jasmine-node will return failure with next explanation:

```
Message:
    TypeError: LazyGen(...).map is not a function
```

So, to fix we need to provide `map` function. We know, taht it should return self and store modificators.

{% highlight javascript %}
function LazyObject(generator) {
  const maps = [];
  const runnedGenerator = generator();
  this.map = function(func) {
    maps.push(func)
    return this;
  };
  return this;
};
{% endhighlight %}

```
Message:
     TypeError: subject.next is not a function
```

Now we need to realize `next` function. Now it seems pretty easy

{% highlight javascript %}
  this.next = function() {
    const result = runnedGenerator.next();
    maps.forEach(filter => result.value = filter(result.value));
    return { done: result.done, value: result.value }
  };
{% endhighlight %}

With such realisation we have ability to chain modifiers in such way

{% highlight javascript %}
LazyGen(fibonacci).map(x2).map(function(x) { return ++x });
{% endhighlight %}

Write test for it

{% highlight javascript %}
  it("allows you to chain more then one methods for generators", function() {
    const subject = LazyGen(fibonacci).map(x2).map(function(x) { return ++x });
    expect(subject.next().value).toBe(3);
    expect(subject.next().value).toBe(3);
    expect(subject.next().value).toBe(5);
  });
{% endhighlight %}

Now we need to implement full [Array interface](<https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array>) to provide user ability using this as plain array. I will not realize all this methods here (it's just a metter of time), but will realize `take` method that retrives passed count of values from generator.

So, start from test:

{% highlight javascript %}
it("provides ability to take n arguments", function() {
    const subject = LazyGen(fibonacci).map(x2).map(up);
    expect(subject.take(3)).toEqual([3, 3, 5]);
});
{% endhighlight %}

and implementation:

{% highlight javascript %}
this.take = function(count) {
    let result = [];
    for(let i = 0; i < count; ++i) {
        result.push(this.next().value);
    }
    return result;
};
{% endhighlight %}


Pretty easy. What's next? Next we need to realize exception throwing, ability to not only map elements but filters and so on, currenlty I'm thinking about storing all passed filters in cortege `{:indexNumber, :operation, :function}` and then process it, but it will be later. Now I just realized what I want and keep it so for a while.

P.S. All work on this topic is stored on [github](<https://github.com/ignat-z/jslazygen>).
P.P.S. Instead of using vim callback you can just pass --autotest argument into jasmine-node
