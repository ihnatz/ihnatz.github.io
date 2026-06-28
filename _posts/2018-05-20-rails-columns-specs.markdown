---
layout: post
title:  "How to test Rails models structure"
date:   2018-05-20 18:05:10
tags: rails schema specs
published: false
---

When you are generating Rails migration you have something like this:

```
bin/rails g migration CreateProducts
```

```ruby
class CreateProducts < ActiveRecord::Migration[5.1]
  def change
    create_table :products do |t|
    end
  end
end
```

Rails is pretty smart to parse "create" word and decide that you want to create new table `products`. There is one moment -- by default this table doesn't have timestamps columns. It's an expected behavior because you may need simply a JOIN table for `has_and_belongs_to_many` association or anything non-Rails specific at all. But mostly you want to have these columns because they make developers' life much easier.

One way to solve this problem is to write a custom test which will enforce you to add these columns for the existing models. Rails provides a special method on class object to receive the class descendants, non-surprisingly it is `descendants`. So, by calling `descendants` on `ApplicationRecord` class you receive all classes which were inherited from `ApplicationRecord`. But there are some moments which could astonish you.

Let's run Rails console and try to exec this command:

```ruby
ApplicationRecord.descendants # => []
```

Probably your application will return some models but definitely not all application models. This behavior is standard for Rails application. By default Rails uses `config.eager_load = false` setting in your `config/environments/development.rb`. It means that nothing will be loaded until you reference it. So, if you have `Document` model and exec this:

```ruby
Document # we just enforce Rails to load this model
ApplicationRecord.descendants # => [Document]
```

There is a way how to enforce Rails load all your classes -- use `eager_load`. It could be done by changing default setting in environment settings or use special command `Rails.application.eager_load!` which will make work for you. Let's try again:

```ruby
Rails.application.eager_load!
ApplicationRecord.descendants.count # => 23
```

![Descendants](/assets/descendants.svg)

Much better now. So, we have all the application models in one place. Now we need to exclude anonymous classes and classes which aren't assume to have timestamps. I wrote a special class which makes this job:

```ruby
class InterfaceTester
  def initialize(blocklist = [])
    @blocklist = blocklist
    Rails.application.eager_load!
  end

  def models
    filtered_descendants(ApplicationRecord)
  end

  private

  def filtered_descendants(klass)
    klass.descendants.reject { |model| ignored?(model) }
  end

  def ignored?(model)
    [model.anonymous?, blocklisted?(model)].any?
  end

  def blocklisted?(model)
    return false unless @blocklist

    (model.ancestors & @blocklist).present?
  end
end
```

As you can see it's super simple. So, how to use it?

```ruby
describe InterfaceTester do
  subject { described_class.new(block_list) }

  let(:block_list) { [] }

  it "have timestamps columns" do
    subject.models.each do |model|
      expect(model.column_names).to include('created_at', 'updated_at')
    end
  end
end
```

Let's run it:


```
  1) InterfaceTester have timestamps columns
     Failure/Error: expect(model.column_names).to include('created_at', 'updated_at')
       expected ["id", "name", "transported"] to include "created_at" and "updated_at"
```

Hm, seems like not very useful, isn't it? RSpec allows you to add meta info for your specs, let's use this:

```ruby
        expect(model.column_names).to include('created_at', 'updated_at'),
  "#{model.name} model should have timestamps columns"
```

```
  1) InterfaceTester have timestamps columns
     Failure/Error:
             expect(model.column_names).to include('created_at', 'updated_at'),
       "#{model.name} model should contain timestamps columns"

       Product model should have timestamps columns
```


Much better, now we know which model is our problem. But we can do better, RSpec allows to define custom matchers for any kind of things you want.

```ruby
RSpec::Matchers.define :has_columns do |*columns|
  match do |model|
    columns.map(&:to_s).all? do |column|
      model.column_names.include?(column)
    end
  end
end
```

Let's use it!

```ruby
    it "have timestamps columns" do
      subject.models.each do |model|
        expect(model).to has_columns(:created_at, :updated_at)
      end
    end
```

```
  1) InterfaceTester have timestamps columns
     Failure/Error: expect(model).to has_columns(:created_at, :updated_at)
       expected Product(id: integer, name: text, transported: boolean) to has columns :created_at and :updated_at
```

Perfect, RSpec even defined clear failure message for us for free. So, final version can look like:

```ruby
describe InterfaceTester do
  subject { described_class.new(block_list) }

  let(:block_list) { [Product] }

  it "have timestamps columns" do
    subject.models.each do |model|
      expect(model).to has_columns(:created_at, :updated_at)
    end
  end
end
```

This technique is pretty expandable. For example here is a matcher which checks for defining instance methods:

```ruby
RSpec::Matchers.define :define_methods do |*methods_names|
  match do |model|
    methods_names.map(&:to_s).all? do |method_name|
      model.method_defined?(method_name)
    end
  end
end
```

For class methods you can use default [respond_to matcher](<https://relishapp.com/rspec/rspec-expectations/docs/built-in-matchers/respond-to-matcher>).

With a little code you can receive all `ApplicationControllers`:

```ruby
  def controllers
    filtered_descendants(ApplicationController)
  end
```

 or all classes under the namespace:

```ruby
  def classes_in_module(module_object)
    module_object.constants
                 .map    { |const_name| module_object.const_get(const_name) }
                 .select { |constant|   constant.is_a? Class }
                 .reject { |klass|      ignored?(klass) }
  end
```

and test them. Ruby doesn't have interfaces but nobody interfere to emulate it in specs for some special cases like this one.
