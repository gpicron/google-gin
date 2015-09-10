This tutorial will show how a simple GWT widget can be constructed with GIN. You should be already familiar with [Guice](http://code.google.com/p/google-guice/) - if not, then please read the [Guice User Guide](http://code.google.com/p/google-guice/wiki/Motivation?tm=6) for an introductory overview of Guice concepts.



# Introduction #

In regular Guice, you would use:

```
// Does NOT work in GWT code!
MyWidgetMainPanel mainPanel = injector.getInstance(MyWidgetMainPanel.class);
```

However, as the comment points out that won't work in GWT:
  * There is no actual class available in the resulting JavaScript.
  * Guice makes heavy use of reflection, most of which GWT does not emulate.

Fortunately in GWT there is a different idiom which accomplishes the same purpose.

# Using Gin #

## Step 1. Inheriting the GIN module ##
```
<module>
  ...
  <inherits name="com.google.gwt.inject.Inject"/>
  ...
</module>
```

## Step 2. Defining the Ginjector ##
Declare an interface with methods that return the desired types:

```
public interface MyWidgetGinjector extends Ginjector {
  MyWidgetMainPanel getMainPanel();
}
```

Experienced GWT users will notice the similarity to the way GWT image bundles and messages are done: You simply create a method for each object type you want to create, and the an implementation of the interface gets generated for you **at compile time**.

Note that you only need to create injector methods for classes that you would directly access in your top-level initialization code, such as the UI classes to install in your `RootPanel`. You don't need to create injector methods for lower-level classes that will be automatically injected. So for example, if Class `A` uses class `B` which uses class `C`, you only need to create an injector method for `A`, as the other classes `B` and `C` will automatically be injected into `A`.

In other words, injector methods provide a bridge between the Guice and non-Guice world.

## Step 3. Declare bindings ##

The next step is to bind the various classes and providers using a Guice module. The module class looks almost exactly like it would in regular Guice (We use the `GinModule` and `AbstractGinModule` instead of `Module` and `AbstractModule`.) Here's an example module:

```
public class MyWidgetClientModule extends AbstractGinModule {
  protected void configure() {
    bind(MyWidgetMainPanel.class).in(Singleton.class);
    bind(MyRemoteService.class).toProvider(MyRemoteServiceProvider.class);
  }
}
```

Note that if GIN can't find a binding for a class, it falls back to calling `GWT.create()` on that class. What this means that image bundles and translated messages will just magically work.

We emulate most of Guice's types but not modules, because:
  * We need to be able to develop GIN at our own pace.
  * We will probably never support `toInstance(...)` and the like because we do our work with modules at compile time, not runtime.

For compatibility with regular Guice we provide a `GinModuleAdapter` class, which makes your `GinModule` available as a `Module`.

## Step 4. Associating the module with the injector ##

Add the `GinModules` annotation to your Ginjector, specifying the module(s) needed to configure the application.

```
@GinModules(MyWidgetClientModule.class)
public interface MyWidgetGinjector extends Ginjector {
  MyWidgetMainPanel getMainPanel();
}
```

Current project layout:
```
MyWidgetProject/
    client/
        MyWidget.java
        MyWidgetGinjector.java
        MyWidgetMainPanel.java
        MyWidgetClientModule.java
    public/
    server/
```

## Step 5. Creating the Ginjector ##

To create the injector instance, use the standard `GWT.create()` call. This can be done during static initialization:

```
public class MyWidget implements EntryPoint {
  private final MyWidgetGinjector injector = GWT.create(MyWidgetGinjector.class);

  public void onModuleLoad() {
    MyWidgetMainPanel mainPanel = injector.getMainPanel();
    RootPanel.get().add(mainPanel);
  }

}
```

# Compilation #

Your project should now be ready to be compiled. Since Gin uses Guice at compile-time and Guice operates on compiled java classes, you'll have to provide the compiled java classes to the gwt compiler. Example from an ant build file:

```
<target name="javac" description="Compiles Java types needed during GWT compilation">
  <mkdir dir="war/WEB-INF/classes"/>
  <javac srcdir="src" destdir="war/WEB-INF/classes">
    <classpath refid="project.libs"/>
  </javac>
</target>

<target name="gwtc" depends="javac" description="GWT Web Compilation">
  <java classname="com.google.gwt.dev.Compiler">
    <classpath>
      <pathelement location="src"/>

       <!-- Note the reference to the compiled java classes -->
      <pathelement location="war/WEB-INF/classes"/>
      <path refid="project.libs"/>
    </classpath>
     <arg value="com.example.myapp.MyApp"/>
  </java>
</target>
```

# Full Example #

You can find an example for Gin project layout and compilation in our [samples collection](http://code.google.com/p/google-gin/source/browse/trunk/samples/).

# Gin "Magic" #

Gin tries to make injection painless and remove as much boilerplate from your code as possible. To do that the generated code includes some magic behind the scenes which is explained here.

## Deferred Binding ##

One way Gin optimizes code is by automating GWT [deferred binding](http://code.google.com/webtoolkit/doc/latest/DevGuideCodingBasics.html#DevGuideDeferredBinding). So if you inject an interface or class<sup>1</sup> bound through deferred binding (but not through a Guice/Gin binding), Gin will internally call `GWT.create` on it and inject the result. One example are GWT [messages](http://google-web-toolkit.googlecode.com/svn/javadoc/2.0/com/google/gwt/i18n/client/Messages.html) and [constants](http://google-web-toolkit.googlecode.com/svn/javadoc/2.0/com/google/gwt/i18n/client/Constants.html) (used for i18n purposes):

```
public interface MyConstants extends Constants {
  String myWords();
}

public class MyWidget {

  @Inject
  public MyWidget(MyConstants myconstants) {
    // The injected constants object will be fully initialized - 
    // GWT.create has been called on it and no further work is necessary.
  }
}
```

_Note:_ Gin will not bind the instances created through `GWT.create` in singleton scope. That should not cause unnecessary overhead though, since deferred binding generators usually implement singleton patterns in their generated code.

<sup>1</sup>: Gin creates all instances through `GWT.create` (instead of `new`) for which it is legal: interfaces and classes with default constructors (whether they have an `@Inject` annotation or not).

## Remote Services ##

The `RemoteService` magic Gin performs is an extension of the deferred binding optimization explained above. Every time Gin is asked to inject an [asynchronous remote service](http://code.google.com/webtoolkit/doc/latest/DevGuideServerCommunication.html#DevGuideMakingACall), it will inject an instance retrieved through calling `GWT.create` on its regular remote service:

```
public interface MyRemoteService extends RemoteService { ... }

public interface MyRemoteServiceAsync { ... }

public class MyWidget {
  
  @Inject
  public MyWidget(MyRemoteServiceAsync service) {
    // The 'service' will be created by calling GWT.create(MyRemoteService.class)
    // prior to injection. 
  }
}
```