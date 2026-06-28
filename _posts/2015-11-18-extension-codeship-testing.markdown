---
layout: post
title:  "Testing Chrome Extension with CodeShip"
date:   2015-11-18 01:22:09
tags: javascript codeship karma eslint
published: false
---

When you are writing your extension for Google Chrome (or Chromium) you want to be sure that your code works as you assume. The mosst effective and simple way to achive it is to use testing on your project.

We will use [Karma](<http://karma-runner.github.io/>) and [Jasmine](<http://jasmine.github.io/>) for testing our project. If your project don't use packet manager you need to install it. For exmaple like this:

{% highlight bash %}
npm init .
{% endhighlight %}

It will generate all curresponding stuff for your project. So, let's install karma and jasmine

{% highlight bash %}
npm install karma --save-dev
npm install jasmine-core --save-dev
npm install karma-jasmine --save-dev
{% endhighlight %}

It will install all that we need for testing our project. Now we need to generate karma config

{% highlight bash %}
./node_modules/karma/bin/karma init .
{% endhighlight %}

As we are testing Google Chrome extension only browser that we need is Chrome. After this you can run your tests.

{% highlight bash %}
./node_modules/karma/bin/karma start
{% endhighlight %}

Or, in case if you have installed karma global

{% highlight bash %}
karma start
{% endhighlight %}

But after command executing you will see something like it:

{% highlight bash %}
18 11 2015 01:42:44.818:WARN [karma]: No captured browser, open http://localhost:9876/
18 11 2015 01:42:44.825:INFO [karma]: Karma v0.13.15 server started at http://localhost:9876/
18 11 2015 01:42:44.826:WARN [launcher]: Can not load "Chrome", it is not registered!
  Perhaps you are missing some plugin?
{% endhighlight %}

And it's true. Karma can't start Google Chrome. You can do it manualy by passing karma server url into browser but it's not we want to do every time when we are starting our tests. To solve it we need to install `karma-chrome-launcher`

{% highlight bash %}
npm install karma-chrome-launcher --save-dev
{% endhighlight %}

and add it into `karma.conf.js` `plugins` section

{% highlight javascript %}
plugins: [
    'karma-jasmine',
    'karma-chrome-launcher'
]
{% endhighlight %}

After all this actions your `karma.conf.js` should seems something like it:

{% highlight javascript %}
module.exports = function(config) {
    var configuration = {
        basePath: '',
        frameworks: ['jasmine'],
        files: [
            'lib/*.js',
            'src/**/*.js',
            'test/*-spec.js'
        ],
        exclude: [
        ],
        preprocessors: {
        },
        reporters: ['dots'],
        port: 9876,
        colors: true,
        logLevel: config.LOG_INFO,
        autoWatch: true,
        browsers: ['Chrome'],
        singleRun: false,
        concurrency: Infinity,
        plugins: [
            'karma-jasmine',
            'karma-chrome-launcher'
        ]
    };
    config.set(configuration);
};
{% endhighlight %}

And let's write our first test:
`test/test-spec.js`

{% highlight javascript %}
describe('Test', function() {
    it('passes', function() {
        expect(true).toBe(true);
    });
});
{% endhighlight %}

Nice. We have testing. Now we want move it to CI. As we have Chrome extension we need a Chrome browser to test it. So we can't use `PhantomJS` for testing our extension. After lot of googling and attempts I found [CodeShip](<https://codeship.com/>) which only have almost actual version of Google Chrome that can be used for testing. So, what you need to do to use it with your extension?

Firstly you need to provide your test command. For it we need to add coomand in `package.json`

{% highlight javascript %}
"scripts": {
    "test": "./node_modules/karma/bin/karma start --single-run --browsers Chrome"
}
{% endhighlight %}

Then we need to provide command for Codeship to install extension dependencies

{% highlight bash %}
npm install
{% endhighlight %}

and command to run tests

{% highlight bash %}
npm run test
{% endhighlight %}

So, you have your first green build. But you want more. You want to have conventioned code in your extension. Let's use [JSLint](<http://eslint.org/>) for it.

{% highlight bash %}
npm install eslint --save-dev
{% endhighlight %}

Then you need to provide file with rules for `eslint`. For example this:

{% highlight javascript %}
{
    "ecmaFeatures" : {
        "blockBindings": true
    },
    "rules": {
        "indent": [
            2,
            4
        ],
        "quotes": [
            2,
            "single"
        ],
        "block-spacing": [
            2,
            "always"
        ],
        "linebreak-style": [
            2,
            "unix"
        ],
        "semi": [
            2,
            "always"
        ],
        "dot-notation": [
            2,
            { "allowKeywords": false }
        ],
        "no-unused-vars": [
            2,
            { "vars": "local", "args": "after-used", "argsIgnorePattern": "^_" }
        ],
        "no-param-reassign": [
            2,
            { "props": false }
        ],
        "no-useless-concat": [
            2
        ],
        "radix": [
            2
        ],
        "one-var": [
            2,
            "never"
        ],
        "yoda": [
            2,
            "never",
            { "exceptRange": true }
        ],
        "comma-spacing": [
            2,
            {"before": false, "after": true}
        ],
        "comma-style": [
            2,
            "last"
        ],
        "quote-props": [
            2,
            "as-needed",
            { "keywords": true }
        ],
        "object-curly-spacing": [
            2,
            "always"
        ],
        "space-before-blocks": [
            2,
            "always"
        ],
        "space-after-keywords": [
            2,
            "always"
        ]
    },
    "env": {
        "browser": true
    },
    "globals": {
        "$": true,
        "chrome": true,
        "module": true
    },
    "extends": "eslint:recommended"
}
{% endhighlight %}

You need to pay attention on `globals` section. If you are not using AMD or something like it you need to write each global variable here. Additionaly we are using 'block bindings' on our project, so we have string with  `"blockBindings": true`.

Almost done, to use this on CI we need to provide command for running eslint.

{% highlight javascript %}
"lint": "./node_modules/eslint/bin/eslint.js src/**/"
{% endhighlight %}

and add lint starting before `npm run test`

{% highlight bash %}
npm run lint
{% endhighlight %}

So, if our code dosn't follow conventions it will make build unsuccessful. And that's all.
