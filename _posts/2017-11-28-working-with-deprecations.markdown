---
layout: post
title:  "Working with deprecations"
date:   2017-11-28 21:11:13
categories: rails honeybadger
published: false
---

Deprecations are the normal part of lifecycle of any large application. A big amount of code makes changes of public API really difficult. Especially if we need to do it for all the code. When you meet this problem for the first time, the first reaction is to simply add some `puts` with info and you think it will be ok, but there are better options, especially if your project is on RoR.

Let’s try to make our deprecation warning messages more standard. So, let’s imagine that you have method which performs some calculations (of course it will be named `calculate`), and you want to deprecate it.
 
```ruby
class Calculator
  def calculate
    # a lot of very useful calculations
    :result
  end
end
```


The most simple way to do it is to use `ActiveSupport::Deprecation`. So, you simply need to add one line to your method body:

```ruby
def calculate
  ActiveSupport::Deprecation.new.warn
  :result
end
```

 
Let’s run our code:

```
DEPRECATION WARNING: You are using deprecated behavior which will be removed
from the next major or minor release. (called from <main> at logic.rb:9)
```

There are many options you can use to customize your deprecation messages. You can add some details, may be pass some callbacks, etc. But if you are going this way, you are changing method body and polluting git history. In this case if something change you will need to change your messages across whole application. Yes, you can introduce some singletons, add inheritance, move it to different classes, may be introduce a factory...

Wait a minute, I’m simply wanted to deprecate some methods.

Actually, I do want to use the trick from the new Ruby version (can’t remember exact number), which enforced method definitions to return their names, and allowed something like: `deprecated def calculate` to exist. And it’s really easy, we need to declare anonymous module which we will prepend for the class which contains methods we want to deprecate.


```ruby
module Deprecatable
  def deprecate(method_name)
    mod = Module.new do
      define_method(method_name) do |*args, &block|
        ActiveSupport::Deprecation.new.warn
        super(*args, &block)
      end
    end

    self.prepend(mod)
  end
end
```

```ruby
class Calculator
  extend Deprecatable

  deprecate def calculate
    :result
  end
end
```

We don’t change any business logic, just add some wrapper for it, and it looks really awesome (at least for me) but, actually, it doesn’t solve our problems (but it is still cool). From the other side, it is very similar to Java annotations, but who cares). Ok, we still have a problem.

Reading through Rails source code, you could find a module `Deprecation::MethodWrapper`, which is included in `ActiveSupport::Deprecation`. It allows to deprecate methods without making direct changes in our code. According to related docs we can use it in the next manner:

```ruby
ActiveSupport::Deprecation.deprecate_methods(Calculator, :calculate)
```

And the result is:

```
DEPRECATION WARNING: calculate is deprecated and will be removed from Rails 5.2
(called from <main> at logic.rb:11)
```

Ok, it’s very nice, no code was changed, message is a little weird, but it can be easily changed to something more readable. Rails provides us such an ability:
 
```ruby
ActiveSupport::Deprecation.deprecate_methods(Calculator,
  calculate: 'please, do not use this method because of the reason',
  deprecator: ActiveSupport::Deprecation.new('next-release', 'Calculator'))
```

New result:

```
DEPRECATION WARNING: calculate is deprecated and will be removed from Calculator
next-release (please, do not use this method because of the reason) (called from
<main> at logic.rb:15)
```

Ok, we have a nice and clean solution, which is doing what we want, but… How often do you ignore such messages? I will fix it later, aha. It will be nice to track such deprecation in some place to fix it later. Hm, very similar to errors tracking systems. So, [Honeybadger](https://www.honeybadger.io/) for example. As you can see above, Rails provides us an ability to pass custom deprecators. Further investigation reveals that Rails is actually calling only one method: `deprecation_warning` (if we are talking about methods deprecations). So, we can introduce our own deprecator which will notify Honeybadger.

 As I said before, we simply need to implement `deprecation_warning` method to introduce our deprecator.


```ruby
class HoneybadgerDeprecator
  DEPRECATION = '⛔ Method `%<method_name>s` is deprecated. Please, refer to %<refer>s.'

  def initialize(debug: false, refer: 'your team lead')
    @debug = debug
    @refer = refer
  end

  def deprecation_warning(depricated_method_name, message = nil, caller_backtrace = nil)
    caller_backtrace ||= caller_locations(2) if @debug
    message ||= format(DEPRECATION,
                       method_name: depricated_method_name,
                       refer: @refer)
    Honeybadger.notify(ActiveSupport::DeprecationException.new(message),
                       message: message,
                       trace: caller_backtrace,
                       depricated_method_name: depricated_method_name)
  end
end
```

Also it will be nice to add some tests to check that code actually works
 
```ruby
RSpec.describe HoneybadgerDeprecator do
  let(:fred)     { Class.new { def call; end } }
  let(:message)  { /`call` is deprecated/ }

  before do
    ActiveSupport::Deprecation.deprecate_methods(fred, :call, deprecator: subject)
    Honeybadger.configure do |config|
      config.api_key = 'temp' # we need to set any API key
      config.backend = 'test' # special backend for Honeybadger testing
      config.logger  = NullLogger.new # we do not want to hit our console
    end
  end

  it 'sends notification to honeybadger' do
    expect do
      fred.new.call
      Honeybadger.flush
    end.to change(Honeybadger::Backend::Test.notifications[:notices], :size).by(1)

    expect(Honeybadger::Backend::Test.notifications[:notices].first.error_message)
      .to match(message)
  end
end
```

Now, let’s try to apply it to our `Calculator` code:

```ruby
ActiveSupport::Deprecation.deprecate_methods(Calculator,
  calculate: 'please, do not use this method because of the reason',
  deprecator: HoneybadgerDeprecator.new)
```

and it works! Now we can track all deprecations, analyze some statistics and so on. But usually Honeybadger is configured to work only in the production environment, which makes sense. So, we should always remember about developers who use it in the development environment as well. Seems like there are no better way to implement it, then simple logging. 

So, the final result is:


```ruby
class HoneybadgerDeprecator
  DEPRECATION = '⛔ Method `%<method_name>s` is deprecated. Please, refer to %<refer>s.'
  LOG_TEXT = '%<message>s Called from: %<trace>s'

  def initialize(debug: false, refer: 'your team lead', logger: Logger.new(STDOUT))
    @debug = debug
    @refer = refer
    @logger = logger
  end

  def deprecation_warning(depricated_method_name, message = nil, caller_backtrace = nil)
    caller_backtrace ||= caller_locations(2) if @debug
    message ||= format(DEPRECATION,
                       method_name: depricated_method_name,
                       refer: @refer)
    if Rails.env.development?
      @logger.warn(format(LOG_TEXT,
                          message: message,
                          trace: caller_backtrace.to_a.join("\n")))
    else
      Honeybadger.notify(ActiveSupport::DeprecationException.new(message),
                         message: message,
                         trace: caller_backtrace,
                         depricated_method_name: depricated_method_name)
    end
  end
end
```

And don’t forget the tests:
 
```ruby
RSpec.describe HoneybadgerDeprecator do
  let(:fred)     { Class.new { def call; end } }
  let(:rails)    { double('Rails').as_null_object }
  let(:logger)   { instance_double(Logger) }
  let(:message)  { /`call` is deprecated/ }

  subject { HoneybadgerDeprecator.new(logger: logger) }

  before do
    stub_const('Rails', rails)
    ActiveSupport::Deprecation.deprecate_methods(fred, :call, deprecator: subject)
  end

  context 'in development mode' do
    before do
      allow(rails).to receive(:development?).and_return(true)
    end

    it 'notifies developer to console' do
      expect(logger).to receive(:warn).with(be =~ message)
      fred.new.call
    end
  end

  context 'in production mode' do
    before do
      allow(rails).to receive(:development?).and_return(false)
      Honeybadger.configure do |config|
        config.api_key = 'temp'
        config.backend = 'test'
        config.logger  = NullLogger.new
      end
    end

    it 'sends notification to honeybadger' do
      expect do
        fred.new.call
        Honeybadger.flush
      end.to change(Honeybadger::Backend::Test.notifications[:notices], :size).by(1)
      expect(Honeybadger::Backend::Test.notifications[:notices].first.error_message)
        .to match(message)
    end
  end
end
```

_UPD:_ Another approach is to use Rails-provided DI instead of direct methods calling based on current Rails environment. There is a nice example how to use it in [official Rails guide](<http://guides.rubyonrails.org/configuring.html#custom-configuration>). New version of code could look something like:

```ruby
class ApplicationDeprecator
  DEPRECATION = '⛔ Method `%<method_name>s` is deprecated. Please, refer to %<refer>s.'

  def initialize(debug: false, refer: 'your team lead')
    @debug = debug
    @refer = refer
  end

  def deprecation_warning(depricated_method_name, message = nil, caller_backtrace = nil)
    caller_backtrace ||= caller_locations(2) if @debug
    message ||= format(DEPRECATION,
                       method_name: depricated_method_name,
                       refer: @refer)
    reporter.notify(message, depricated_method_name, caller_backtrace)
  end

  private

  def reporter
    Rails.configuration.deprecation_reporter.tap do |instance|
      unless instance.respond_to?(:notify)
        raise ArgumentError, 'Deprecation reporter should respond to :notify'
      end
    end
  end
end

class HoneybadgerReporter
  def notify(message, depricated_method_name, caller_backtrace)
    Honeybadger.notify(ActiveSupport::DeprecationException.new(message),
                       message: message,
                       trace: caller_backtrace,
                       depricated_method_name: depricated_method_name)
  end
end

class LogReporter
  LOG_TEXT = '%<message>s Called from: %<trace>s'

  def initialize(logger: Logger.new(STDOUT))
    @logger = logger
  end

  def notify(message, depricated_method_name, caller_backtrace)
    @logger.warn(format(LOG_TEXT,
                        message: message,
                        trace: caller_backtrace.to_a.join("\n")))
  end
end
```

And now, to configure what exactly should happen:

```ruby
# config/environments/production.rb
config.deprecation_reporter = HoneybadgerReporter.new
```

```ruby
# config/environments/test.rb
# config/environments/development.rb
config.deprecation_reporter = LogReporter.new
```
