---
layout: post
title:  "How hey.com uses native bridge"
date:   2021-10-16 01:22:35
tags: strada hotwired
published: false
---

_or how Strada will likely work_.

In this article, I would like to describe how hey.com uses a native bridge to interact between WebView and native code. I haven't seen the hey.com source code, but at the same time it's not a pure black box, some things can help us to figure out how it works. It's a very elegant way and I like to share some info about it with you!

In these examples, I will use the Android version, mostly because my Kotlin is slightly better than Swift and because I read `turbo-android` many-many times trying to fix one pretty annoying bug.

## Introduction

You can skip this section if you already know how `JavascriptInterface` works on Android.

---------------------------------------------------
There are some basic examples of interaction between WebView and native code on the [Android Developers Page](<https://developer.android.com/guide/webapps/webview>). What's important there it's that there are two sides (JavaScript + Android) and they can interact through registered adapters.

Let me bring an example from the original article to illustrate the main idea here:

```kotlin
/** This class will be called from JavaScript **/
class WebAppInterface(private val mContext: Context) {

    /** Show a toast from the web page  */
    @JavascriptInterface
    fun showToast(toast: String) {
        Toast.makeText(mContext, toast, Toast.LENGTH_SHORT).show()
    }
}
```

```kotlin
/** Register WebAppInterface as an object which can be accessed with Android alias **/
val webView: WebView = findViewById(R.id.webview)
webView.addJavascriptInterface(WebAppInterface(this), "Android")
```

```html
<!-- index.html code -->
<input type="button" value="Say hello" onClick="showAndroidToast('Hello Android!')" />

<script type="text/javascript">
    function showAndroidToast(toast) {
        Android.showToast(toast);
    }
</script>
```

---------------------------------------------------

It was a really basic example that demonstrates how the sides can interact with each other. But in the real world, there are many nuances you need to keep in mind. It's about Android, what about iOS? How to manage increasing code size? And there are many more.

Let's see how hey.com solves these problems. To demonstrate how it works let's show the messages from Rails `session` in the native view. This how it looks on iOS, Android and in a browser. You can see that there is no indication on iOS, there is toast element on Android and simple div with animation on browser.

![iOS](/assets/strada/ios.gif)
![Android](/assets/strada/android.gif)
![Browser](/assets/strada/browser.gif)
{: .text-center}

Meet the sides:
- **Strada** (aka `nativeBridge`). This is JavaScript code responsible for sending messages to the native code. Can be **Android Strada** or **iOS Strada**.
- **Strata** (aka `webBridge`). This is JavaScript code which delegates messages from web to **Strada** adapter.
- **Stimulus**. This is a JavaScript framework which helps to interact with JavaScript code by HTML `data-` annotations.
- **BridgeKt** (or **BridgeSwift**). This is Kotlin/Swift code which will perform native actions.

## Setting the bridges

Android application makes the first request to the webserver. As soon as the server responds the magic starts to happen:

- Application initializes **BridgeKt**, which registers itself as an interface called **Strada**

```kotlin
this.bridge = Bridge(webView)

class Bridge(val webView: WebView) {
    init {
      webView.addJavascriptInterface(this, "Strada");
    }
}
```

- Application executes **Strada** code on the WebView to be able to call the registered interface

```kotlin
internal fun installBridge(onBridgeInstalled: () -> Unit) {
    val script = "window.nativeBridge == null"
    val bridge = context.contentFromAsset("js/strada.js")

    runJavascript(script) { s ->
        if (s?.toBoolean() == true) {
            runJavascript(bridge) {
                onBridgeInstalled()
            }
        }
    }
}
```

- The executed code from `installBridge` registers **Strada** on the page

```javascript
window.nativeBridge = new NativeBridge()
```

- Browser with a page from the webserver runs the JavaScript code and registers **Strata**

```javascript
const bridge = new Strata();
window.webBridge = bridge;
```

- Browser registers **Stimulus** and all its' controllers

```javascript
const application = Application.start()
const context = require.context("controllers", true, /_controller\.js$/)
application.load(definitionsFromContext(context))
```

After this stage we have
- Initialized **Strata** registered as `webBridge`.
- Initialized **Strada** registered as `nativeBridge`.
- Initialized **Stimulus**.
- Initialized **BridgeKt** which can be referred as **Strada** from JavaScript.

## Meeting each other

Now, with all this zoo of different technologies and bridges, we need to create a working system.

- As soon as **Strada** is initialized, it sends "ready" signal to the **BridgeKt** through the **Strada** interface:

```javascript
function initializeBridge() {
  window.nativeBridge = new NativeBridge()
  window.nativeBridge.ready()
}

ready() {
  Strada.bridgeDidInitialize()
}
```

**BridgeKt** registers all the components that can be called from the WebView, here is `FlashMessagesComponent` which will be able to show messages from Rails `session` in native view:

```kotlin
this.registeredComponents = listOf(
  FlashMessagesComponent(this)
)

@JavascriptInterface
fun bridgeDidInitialize() {
    this.registeredComponents.forEach {
        register(it.name)
    }
}

fun register(name) {
  runJavascript("window.nativeBridge.register($name)")
}
```

As soon as the first component is registered, **Strada** registers itself as an adapter for **Strata**

```kotlin
register(component) {
  this.supportedComponents.push(component)

  if (!this.adapterIsRegistered) {
    this.registerAdapter()
  }
}

registerAdapter() {
  this.adapterIsRegistered = true

  if (window.webBridge) {
    window.webBridge.setAdapter(this)
  } else {
    document.addEventListener("web-bridge:ready", () => window.webBridge.setAdapter(this))
  }
}
```

And everything is ready for work! I omitted some information about how hey.com manages possible race conditions and possible missing messages to make it simple.

After this stage, we can call any functions on the `webBridge`. It will pass them to `nativeBridge`. After that the bridge will be able to call the native code.

## Cooperation

With everything ready to work let's finally use it! As soon as we have any flash message the webserver renders it:

```html
<div id="flash">
  <div
    class="flash-notice"
    role="alert"
    data-controller="element-removal bridge--flash-message"
    data-action="animationend->element-removal#remove"
  >
    <div class="flash-notice__content">This is a message from Rails!</div>
  </div>
</div>
```

As you can see there is `bridge--flash-message` controller here. Let's look inside!

```javascript
export default class extends BaseComponentController {
  static component = "flash-message"

  connect() {
    super.connect()

    if (this.enabled) {
      this.notifyBridgeOfConnect()
      this.removeFlashNotice()
    }
  }

  notifyBridgeOfConnect() {
    const title = this.bridgeElement.title
    this.send("connect", { title })
  }

  removeFlashNotice() {
    this.element.remove()
  }
}
```

As we can see it `extends BaseComponentController`. On `connect` it calls the `this.send` method. Let's look at it.

```javascript
send(event, data = {}, callback) {
  /* */
  const message = { component: this.component, event, data, callback }
  /* */
  window.webBridge.send(message)
}
```

Here our old friend `window.webBridge` and it sends message like `{ component: "flash-message", event: "connect", { title: "This is a message from Rails!" } }`.

`webBridge` inside checks does **Strada** support this component, adds more info, and passes it to the adapter (which is `nativeBridge`)

```javascript
var id = this.generateMessageId();
var message = {
  id: id,
  component: component,
  event: event,
  data: data || {}
};
this.adapter.receive(message);
```

The adapter calls **BridgeKt** through the **Strada** interface:

```javascript
receive(message) {
  this.postMessage(JSON.stringify(message))
}

postMessage(message) {
  Strada.bridgeDidReceiveMessage(message)
}
```

**BridgeKt** finds the corresponding component and passes info there:


```kotlin
@JavascriptInterface
fun bridgeDidReceiveMessage(rawDescription: String) {
    val message = Message.fromJSON(rawDescription)
    val component = this.registeredComponents.find { it.name == message.component }
    component.call(message)
}
```

And, finally, in the component we do the native code:

```kotlin
fun call(message: Message) {
  Toast.makeText(bridge.context, message.data, Toast.LENGTH_SHORT).show()
}
```

That's it!

It was a long way, and it's a pretty complicated system. But what it gives us? We can use different implementations of **Strada** (for iOS and Android) and keep the web code the same. As well, we have two easily extendable lists of components (on the native and on the web sides) with an ability to check their compatibility. Easy to extend and reuse. Isn't it what we like to see?
