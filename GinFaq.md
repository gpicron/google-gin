# GIN frequently asked questions #

## How does GIN work? ##

GIN uses [Guice](http://code.google.com/p/google-guice) at compile-time via a [GWT Generator](http://google-web-toolkit.googlecode.com/svn/javadoc/1.5/com/google/gwt/core/ext/Generator.html). The generator creates an implementation of your `Ginjector` interfaces.

## Does GIN have runtime overhead? ##

No, it shouldn't -- it essentially generates the exact code you'd use if you did the dependency injection and object construction by hand. You can look for yourself by compiling your GWT application with `PRETTY` mode and looking at the JavaScript.

## What bindings don't work? ##

Because your Module class is actually executed at _compile_ time, you have to make sure its `configure(...)` method does not execute any GWT client-side code directly. And thus, you also can't carry along any object instances from the Module in to the eventual client code. So, these don't work:

  * `.toInstance(obj)`, because `obj` exists at compile-time, not run-time
  * `.toProvider(new Provider<T>() { ... }`, for the same reason but now because the provider does not exist at run-time. You can fix this case by making the provider a public concrete class in `.client`.

While this does:
```
class MyModule extends AbstractGinModule {
  @Override
  protected void configure() {
    bind(Something.class).toProvider(SomethingProvider.class);
  }

  static class SomethingProvider implements Provider<Something> {
    public Something get() {
      return new Something(getDataFromJavaScript());
    }

    // Using JSNI in a provider
    private native String getDataFromJavaScript() /*-{
      ...
    }-*/;
  }
}
```

Note that you will not be able to reuse your `GinModule` on the server side (using `GinModuleAdapter`) if your bindings directly or indirectly make use of GWT specific infrastructure like `GWT.create(...)`, widgets or JSNI.

See the GuiceCompatibility page for a feature comparison with Guice.

## How do I use GIN with GWT RPC? ##

There are two options:
  * Write a custom `Provider<MyRemoteServiceAsync>`.
  * Use GIN's automatic bindings for remote services

The automatic binding works by replacing an injection of `MyRemoteServiceAsync` with `GWT.create(MyRemoteService.class)`. When using this mechanism, to set the URI used for the remote service, put `@RemoteServiceRelativePath` on interface `MyRemoteService`.

## Why do I get a `[ERROR] Deferred binding result type 'A' should not be abstract` error during compile? ##

This can happen when Gin tries to instantiate an interface by calling GWT.create on it - a method Gin uses to allow easy injection of interfaces configured with GWT's deferred binding mechanism. Unfortunately, if no deferred binding or replacement is registered for the interface, you will get the above error. To fix the problem, bind the offending interface to an injectable implementation in your GinModule.

_Note:_ This can also happen when using `@Optional` injection - if the injected parameter is of an interface type and the interface is not bound, Gin will **not** use optional injection but call GWT.create on the interface, potentially causing this error.


---


**Note:** Please don't ask questions in the comments below but join and ask the [GIN discussion group](http://groups.google.com/group/google-gin) instead. We won't answer questions in the comments!