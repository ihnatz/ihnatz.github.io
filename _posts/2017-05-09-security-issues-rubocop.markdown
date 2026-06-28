---
layout: post
title:  "Resolving security issues with Rubocop"
date:   2017-05-09 18:45:13
tags: rails ast rubocop
published: false
---

Sometimes when you are observing your project source code you can see code like:

{% highlight ruby %}
Person.find(params[:id])
{% endhighlight %}

So, in simple Rails application it's ok, there is no any problems, it works like a charm. But it can broke your businesses logic, so by passing in `params` id of person who doesn't belong to your specific group you still will be able to explore his information. To avoid such problems you should use protected scope to find your person:

{% highlight ruby %}
current_group.people.find(params[:id])
{% endhighlight %}

How to avoid such problems? Seems like it is an issue for static code analyzer, `rubocop` in our case.

Ok, let's define code examples which should be reported:

{% highlight ruby %}
Person.find(params[:id])
Person.includes(:assoc).find(params[:id])
Person.includes(:assoc).where(id: 27).find(params[:id])
{% endhighlight %}

Now we will use `ruby-parse` to observe AST of this parts:

{% highlight lisp %}
;; Person.find(params[:id])
(send
  (const nil :Person) :find
  (send
    (lvar :params) :[]
    (sym :id)))

;; Person.includes(:assoc).find(params[:id])
(send
  (send
    (const nil :Person) :includes
    (sym :assoc)) :find
  (send
    (lvar :params) :[]
    (sym :id)))

;; Person.includes(:assoc).where(id: 27).find(params[:id])
(send
  (send
    (send
      (const nil :Person) :includes
      (sym :assoc)) :where
    (hash
      (pair
        (sym :id)
        (int 27)))) :find
  (send
    (lvar :params) :[]
    (sym :id)))
{% endhighlight %}


The third example it synthetic but we will use it just to illustrate concept. Now we are starting to write our cop:

{% highlight ruby %}
module RuboCop
  class ProtectedScopeCop < RuboCop::Cop::Cop
  end
end
{% endhighlight %}

Now we need to catch AST node where we are calling :find method. Ruby-parser provides a [list](<https://github.com/whitequark/parser/blob/master/lib/parser/ast/processor.rb>) of methods which you can use to catch corresponding AST nodes. Each node in `on_send` method consists of thee parts [receiver, method, args]. So, firstly we need to filter non-find methods.

{% highlight ruby %}
def on_send(node)
  _, method_name, _ = *node
  return unless method_name == :find
end
{% endhighlight %}

Easy enough. If we will explore all ASTs that we have we can detect pattern for calling :find on constants:

{% highlight lisp %}
(send
  (send
    (send
      (const nil _) _ _) _ _)
      :find _)
{% endhighlight %}

Each level of `(send ? _ _)` is method calling on constant. In rubocop you can use `#def_node_matcher`. But there is no any special matcher to define optional nesting AST (at least I can find it in [documentation](<http://www.rubydoc.info/gems/rubocop/RuboCop/NodePattern>). Ok, let's simple flatten on node children.

{% highlight ruby %}
def children(node)
  current_nodes = [node]
  while current_nodes.any? { |node| node.child_nodes.count != 0 } do
    current_nodes = current_nodes.flat_map { |node| node.child_nodes.count == 0 ? node : node.child_nodes }
  end
  current_nodes
end
{% endhighlight %}

It's pretty simple, if this node has child -- let's retentive it! And now we need to check is there any constant node?

{% highlight ruby %}
return unless children(node).any? { |node| node.type == :const }
{% endhighlight %}

Last action -- add warning:

{% highlight ruby %}
module RuboCop
  class ProtectedScopeCop < RuboCop::Cop::Cop
    def on_send(node)
      _, method_name, _ = *node
      return unless method_name == :find
      return unless children(node).any? { |node| node.type == :const }
      add_offense(node, :expression, "Dangerous Search!")
    end

    private

    def children(node)
      current_nodes = [node]
      while current_nodes.any? { |node| node.child_nodes.count != 0 } do
        current_nodes = current_nodes.flat_map { |node| node.child_nodes.count == 0 ? node : node.child_nodes }
      end
      current_nodes
    end
  end
end
{% endhighlight %}

It is good enough code which will help you to detect problems with unsafe searches. Some additional words:

- You should add your cope to `lib/`

{% highlight yaml %}
require:
  - ./protected_scope_cop.rb

RuboCop/ProtectedScopeCop:
  Include:
  - app/controllers/**/*.rb
{% endhighlight %}

- You should resolve somehow cases when you really need to call find on `all` scope. For example

{% highlight ruby %}
User.find(doorkeeper_token.resource_owner_id) if doorkeeper_token
{% endhighlight %}

How to deal with it? My suggest is to move constant to method which indicates that you are know what are you doing:

{% highlight ruby %}
protected_scope.find(doorkeeper_token.resource_owner_id) if doorkeeper_token

def protected_scope
  User
end
{% endhighlight %}

That's all.
