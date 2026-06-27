---
layout: post
title:  "Rails: Fast test feedback loop"
date:   2021-08-18 13:26:31
categories: rails specs
published: false
---

Using TDD in pure Ruby is pretty simple. There is no need for any dependencies or frameworks. Let's imagine you want to create a class with the method which adds two values. Let's write a test:

```ruby
require 'minitest/autorun'

describe Calc do
  it "adds two values" do
    assert_equal 5, Calc.new.add(2, 3)
  end
end
```

It's a simple code. We can easily write an implementation (not the cleanest, but good enough) which passes this test:

```ruby
class Calc
  def add(a, b)
    a + b
  end
end
```

Let's run it:

```bash
❯ ruby calc.rb
Run options: --seed 35652

# Running:

.

Finished in 0.000873s, 1145.8179 runs/s, 1145.8179 assertions/s.
1 runs, 1 assertions, 0 failures, 0 errors, 0 skips
```

It's ridiculously fast. That's what you want to have during writing Rails applications. But the problem with Rails applications is that they need a lot of time to start.

```bash
❯ bundle exec rspec spec/
...................

Finished in 0.15816 seconds (files took 1.91 seconds to load)
19 examples, 0 failures
```

Almost two seconds to load for a really small Rails application. You can imagine how much it takes for really huge applications.

What makes it so long to run these tests? Usually, it's Rails itself and a lot of gems that are required by default in your application. Fortunately, we don't always need the whole Rails ecosystem for unit tests. And RSpec provides us a way how to do it.

It's a common practice for programmers to put a lot of code in `spec_helper.rb` and `rails_helper.rb` without understanding where to put one more require or config from some gem. Let's do it the right way.

Let's make a rule: if something is needed for all tests that it goes to `spec_helper`, otherwise `rails_helper`. With this you end with a really short and focused `spec_helper.rb` file:

```ruby
# Make requires easier
$LOAD_PATH.unshift File.join(File.dirname(__FILE__), '..', 'app', 'models')
$LOAD_PATH.unshift File.join(File.dirname(__FILE__), '..', 'app', 'serializers')
$LOAD_PATH.unshift File.join(File.dirname(__FILE__), '..', 'app', 'services')

# Allow gems loading
require 'bundler'
Bundler.require :default, :test

# Configure RSpec
RSpec.configure do |config|
  config.mock_with :rspec
  config.expect_with :rspec
  config.raise_errors_for_deprecations!
  config.after(:suite) do
    raise 'Do not require `rails` for unit tests' if defined?(Rails)
  end
end
```

With this, the test suite will not load any unused resources. Now you need to change all your spec files to use spec_helper by default:

```diff
-require 'rails_helper'
+require 'spec_helper'
```

As Rails doesn't require the whole application on start you need to manually resolve specs dependencies as well:

```diff
+require 'secret_presenter'
```

The next problem is the Rails parts. As rule programmers use Rails functions without knowing that they are part of the framework. Now, it's necessary to implicitly specify such dependencies.

For example, if code uses `delegate` method (which is a part of ActiveSupport) it's needed to be required manually: `require 'active_support/core_ext/module/delegation'`. It can look like a not very handy way but it doesn't add too many problems. There is some amount of ActiveSupport things that are used across the project, some examples from a middle-sized Rails project:

```ruby
require 'active_support/core_ext/hash'
require 'active_support/core_ext/hash/slice'
require 'active_support/core_ext/integer/time'
require 'active_support/core_ext/object'
require 'active_support/core_ext/object/blank'
require 'active_support/core_ext/object/deep_dup'
require 'active_support/core_ext/object/with_options'
require 'active_support/core_ext/string/inflections'
```

The next problem is ActiveRecord models. It's hard to imagine a Rails project without interacting with AR models. To deal with it we can use an RSpec feature that allows dynamic constants loading if they're available.

```diff
-let(:carrier) { instance_double(Carrier, name: 'CarrierName') }
+let(:carrier) { instance_double('Carrier', name: 'CarrierName') }
```

_A caveat #1_: in the current configuration, this code doesn't protect us from referring to non-existing models methods or fields, but we will fix it later.

_A caveat #2_: sometime it's handier  to use `instance_spy` if you don't care too much about some object's methods and you don't want to describe all the methods you want to stub. But don't forget to test such spies with `have_received` for important methods.

And the last one: now you need to write code treating Rails like an external dependency. The most typical example is Rails routes. No more `include Rails.application.routes.url_helpers`, now it's a dependency and you need to work with it accordingly:

```ruby
# was
include Rails.application.routes.url_helpers
root_path

# now
def initialize(routes: Rails.application.routes.url_helpers)
  @routes = routes
end

@routes.root_path
```

As you can see, routes now are an optional value and will not be loaded if any other value is provided.

```bash
❯ bundle exec rspec spec/
...................

Finished in 0.03626 seconds (files took 0.87177 seconds to load)
19 examples, 0 failures
```

Less than a second!

Now we have really fast tests, but almost all of our tests describe their dependencies (like `instance_double`, `class_double`, `instance_spy`, etc) as strings, which doesn't provide us with partial doubles checks. So, if I have `let(:carrier) { instance_double('Carrier', nameXY: 'CarrierName') }` in my code and `carriers` table doesn't have `nameXY` field I have no warnings from RSpec. That's exactly where we need `rails_helper.rb`. It's time to load Rails!

```bash
❯ bundle exec rspec --require rails_helper.rb spec
.......F...........

Failures:

  1) CreateApplication.call when carrier exists generates application based on a passed carrier
     Failure/Error: instance_double('Carrier', nameXY: 'CarrierName')
       the Carrier class does not implement the instance method: nameXY
     # ./spec/interactors/create_application_spec.rb:14:in `block (4 levels) in <main>'
     # ./spec/interactors/create_application_spec.rb:11:in `block (4 levels) in <main>'
     # ./spec/interactors/create_application_spec.rb:21:in `block (4 levels) in <main>'

Finished in 0.16322 seconds (files took 1.55 seconds to load)
19 examples, 1 failure
```

There is how much overhead is added by the Rails ecosystem, even on a small-size project!

It makes sense to mostly run unit tests without guarantees, but for CI and as a pre-commit routine run tests with full Rails loading. `rails_helper` is a good place to put controllers tests or tests that operate with DB as well.

And that's all. Is it easy to set up? Unfortunately not as easy as everybody would like. Many gems aren't ready out-of-box to support manual requiring, so it will require some workarounds. But as rule, it works as expected, and you have a fast feedback loop.
