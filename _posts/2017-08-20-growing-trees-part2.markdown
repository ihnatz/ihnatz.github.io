---
layout: post
title:  "Parsing Ruby AST for Class Dependency Graphs"
date:   2017-08-20 11:45:13
categories: ruby graphviz trees ast
published: false
---

In the first article part we already learned how to render a tree. Let's try to use this knowledge on practice. We don't want to solve something trivial, so we should select suitable problem. So, we will render class relations diagram in Rails application. And there is no any boring models diagrams like <https://github.com/preston/railroady> or <https://github.com/voormedia/rails-erd>, no! We will make it with help of the syntax analyzing.

So, the syntax analyzing consists of two parts -- tokenization (building [AST](<https://en.wikipedia.org/wiki/Abstract_syntax_tree>)) and analyzing itself. Let's start to dive in AST analyzing from reviewing factorial function implementation AST. As AST is actually Tree and we already know how to render trees, we will use knowledge from the first part. It's better to use canonical Visitor pattern for AST traversing, but we do not want to extend existing AST classes and implement less smart variant:

```ruby
require 'parser/current'

class ASTRenderer
  def initialize(tree)
    @tree = tree
  end

  def call
    result = []
    result << "digraph graphname {"

    bfs(@tree).each do |node|
      result << ["#{node.object_id}", %Q{[label="#{present(node)}"]}].join(" ")
      node.children.each do |child|
        next unless child.is_a?(Parser::AST::Node)
        result << [[node.object_id, child.object_id].join(" -> ")].join
      end
    end

    result << "}"
    result.join("\n")
  end

  private

  #TODO: Add missing node types rendering
  def present(node)
    return node.to_s unless node.is_a?(Parser::AST::Node)
    case node.type
    when :begin  then  "(begin)"
    when :if     then  "(if)"
    when :array  then  "(array)"
    when :send   then  "(send :#{node.to_a[1]})"
    when :lvar   then  "(lvar :#{node.to_a[0]})"
    when :def    then  "(def :#{node.to_a[0]})"
    when :str    then  "(str :#{node.to_a[0]})"
    when :args   then  "(args)"
    when :return then  "(return)"
    else node.to_s
    end
  end

  def bfs(node)
    queue = [node]
    discovered = []
    while queue.length > 0 do
      current = queue.shift
      next unless current.is_a?(Parser::AST::Node)
      discovered << current
      current.children.each do |child|
        queue << child if child
      end
    end
    discovered
  end
end

source_code = <<RUBY
  def factorial(n)
    return 1 if [1, 2].includes?(n)
    factorial(n - 1) + factorial(n - 2)
  end
RUBY
ast = Parser::CurrentRuby.parse(source_code)
puts ASTRenderer.new(ast).call
```

Result:


![AST tree](</assets/growing-trees-2/factorial_ast.png>)

Result with comments (you could receive this version by minor changing code above):


![AST tree with comments](</assets/growing-trees-2/factorial_ast_comments.png>)

Factorial function source code:
```ruby
def factorial(n)
  return 1 if [1, 2].includes?(n)
  factorial(n - 1) + factorial(n - 2)
end
```

As you can see, there is no such changes comparing with the code from the first part. There is no rendering actually binary tree code, we are ignoring non-`Parser::AST::Node` nodes and change code which presents node internal value. Even on this small example we can see, that it's really hard to understand what exactly this code doing on high level because there is too much details. But we could cut off excess information.


Let's review node with class definition:

```ruby
class Foo < Bar
  def body; end
end
```

```lisp
s(:class,
  s(:const, nil, :Foo),
  s(:const, nil, :Bar),
  s(:def, :body,
    s(:args), nil))
```

In code above, s -- is [s-expressions](<https://en.wikipedia.org/wiki/S-expression>), notation for nested lists. Let's analyze class node content. There are three arguments in class node: first (`s(:const, nil, :Foo)`) -- node with constant which will be pointer for this class, second (`s(:const, nil, :Bar)`) -- node with constant, which is pointer for parent class, and last one (`s(:def, :body, s(:args), nil))`) -- class body definition, in code above we are using `body` method.

Let's start from displaying all defined classes. We will catch events on class definitions and store all defined classes in the special instance variable.

```ruby
class KlassDefinitionsProcessor < Parser::AST::Processor
  attr_reader :klass_definitions

  def initialize(*)
    super
    @klass_definitions = []
  end

  def on_class(node)
    klass_konst, _parrent, _body = *node
    _nested_konst, konst_name = *klass_konst
    @klass_definitions << konst_name
    super
  end
end
```

It's pretty simple, we are using already defined `AST::Processor`, which process our AST. We add code which responsible for class definitions storing. Let's run our code:

```ruby
source_code = <<RUBY
class Foo; end
class Bar; end
RUBY

ast = Parser::CurrentRuby.parse(source_code)
processor = KlassDefinitionsProcessor.new
processor.process(ast)
p processor.klass_definitions
# => [:Foo, :Bar]
```

Nice for start, but, as always, there are a lot of troubles ahed. What if we need to receive name of class with namespace, like `Foo::Bar::Baz`? Current code will amend namespace part, so we need to fix it. Instead of receiving last constant in class definition we could simply use this node mapping on source code with `klass_konst.loc.expression.source`.


So, our new code:

```ruby
  def on_class(node)
    klass_konst, _parrent, _body = *node
    @klass_definitions << klass_konst.loc.expression.source
    super
  end
```

```ruby
class Foo::Bar::Baz; end
class Oof::Zab; end
# => ["Foo::Bar::Baz", "Oof::Zab"]
```

Ok, let's finally start why are we here -- building relations diagram. We will add catching events on constants processing to the existing code for this:

```ruby
def on_const(node)
  konst_name = node.loc.expression.source
  super
end
```

Not too hard, but there's a problem: we can't determine which class a constant belongs to. We need not only the children of each node, but the parents too. Standard AST processing is top-down, but we need bottom-up context. The solution is to maintain a nesting stack — push the current node on entry, pop it on exit, much like a call stack. Every time `AST::Processor` processes a node it calls `process`, so we override it:

```ruby
  def process(node)
    @nesting.push(node)
    super
    @nesting.pop
  end
```

Now we can easily access the current node's parent:

```ruby
  def parent
    @nesting.last(2).first
  end
```

and find in which class we are processing current node:

```ruby
  def context_klass
    @nesting.reverse.select { |x| x.type == :class }.first
  end
```

Which gives us useful helpers:

```ruby
  def klass_name_for(node)
    klass_konst, _, _ = *node
    source_for(klass_konst)
  end

  def source_for(node)
    node.loc.expression.source
  end
```

Our new code. We are storing class definitions and storing used constants:

```ruby
  def on_class(node)
    @klass_definitions << klass_name_for(node)
    super
  end

  def on_const(node)
    konst_name = source_for(node)
    current_klass = klass_name_for(context_klass)
    (@klass_dependencies[current_klass] ||= []) << konst_name
    super
  end
```

Run this code on new test example:

```ruby
class Foo::Bar::Baz; end
class Oof::Zab; def call; Foo::Bar::Baz.new; end; end

# processor.klass_dependencies
# => {
# "Foo::Bar::Baz"=>["Foo::Bar::Baz", "Foo::Bar", "Foo"],
# "Oof::Zab"=>["Oof::Zab", "Oof", "Foo::Bar::Baz", "Foo::Bar", "Foo"]
# }
```


Not exactly what we expected, though... As we saw before, nested constants is commonly used and `on_const` method will be triggered for each constant. Additionally we could see, that every class has pointer on self, because constant, which is class name, is defining inside node with class definition. So, we need to ignore this two cases: when parent node is constant (nested constants) or class definition. Fix for our function:

```ruby
  def on_const(node)
    unless %i[class const].include?(parent.type)
      konst_name = source_for(node)
      current_klass = klass_name_for(context_klass)
      (@klass_dependencies[current_klass] ||= []) << konst_name
    end
    super
  end
```

```ruby
class Foo::Bar::Baz; end
class Oof::Zab; def call; Foo::Bar::Baz.new; end; end
# {"Oof::Zab"=>["Foo::Bar::Baz"]}
```

Ok, something more complex:

```ruby
class SomeKlass; end
class Foo::Bar::Baz; def call; SomeKlass.new end; end
SomeKlass.new
class Oof::Zab; def call; Foo::Bar::Baz.new; end; end
class Idd; def call; Foo::Bar::Baz.new; end; end
class Zoo; end
# relations.rb:47:in `source_for': undefined method `loc' for nil:NilClass (NoMethodError)
```

Pretty expected, we are calling `SomeKlass.new` outside of context, so `current_klass` is nil. Let's fix it and run again:

```ruby
  def on_const(node)
    unless %i[class const].include?(parent.type)
      konst_name = source_for(node)
      if context_klass
        current_klass = klass_name_for(context_klass)
        (@klass_dependencies[current_klass] ||= []) << konst_name
      end
    end
    super
  end
```

New result:

```ruby
processor.klass_dependencies # => {"Foo::Bar::Baz"=>["SomeKlass"], "Oof::Zab"=>["Foo::Bar::Baz"], "Idd"=>["Foo::Bar::Baz"]}
processor.klass_definitions # => ["SomeKlass", "Foo::Bar::Baz", "Oof::Zab", "Idd", "Zoo"]
```

Oh, we have something to render! So, Graphivz to the rescue. Let's write class for relations graph rendering as before but in functional style without states and mutations:

```ruby
class GraphRenderer
  def initialize(relations, klasses)
    @relations = relations
    @klasses = klasses
  end

  def call
    [
      begin_header,
      klasses_info,
      relations_info,
      end_header
    ].flatten.join("\n")
  end

  private

  def begin_header
    [
      %(digraph graphname {),
      %(rankdir="LR")
    ]
  end

  def klasses_info
    @klasses.map do |klass|
      [wrap(klass), %Q{[label="#{klass}"]}].join(" ")
    end
  end

  def relations_info
    @relations.map do |klass, dependents|
      dependents.map do |child|
        [wrap(klass), wrap(child)].join(' -> ')
      end
    end
  end

  def end_header
    [
      %(})
    ]
  end

  def wrap(text)
    '"' + text + '"'
  end
end
```

![Relations graph](</assets/growing-trees-2/simple_render.png>)

Ok, nice for start, but... Ruby has namespaces resolving mechanism.

```ruby
module A
  module B
    class C
    end
  end
end

module A
  class D
    def call
      B::C.new
    end
  end
end

class E
  def call
    A::B::C.new
  end
end

# E.new.call # => #<A::B::C:0x007f9b0b835768>
# A::D.new.call # => #<A::B::C:0x007f9b0b875250>
```

But for our code D and E classes depend on different classes:

```ruby
processor.klass_dependencies # => {"D"=>["B::C"], "E"=>["A::B::C"]}
processor.klass_definitions # => ["C", "D", "E"]
```

This problem was left unsolved — a Part 3 was planned but never written. The code up to this point is self-contained and the namespace resolution is left as an exercise.
