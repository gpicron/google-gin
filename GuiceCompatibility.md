# Guice Compatibility #

## API ##
We have alternative types for:
  * `Module`, `AbstractModule`
  * `Injector`

For `Module` and `AbtractModule`, use `GinModule` and `AbstractGinModule`. They work much like their counterparts, but lack some of the features. Note that there is a 100% one-way conversion from GWT to Guice using `GinModuleAdapter`, so consider putting shared code in a `GinModule`. Other than that, you could also copy paste bindings from existing modules (usually they'll work), but of course copy-pasting should be discouraged.

For `Injector`, use the `Ginjector` conbined with the `@GinModules` annotation. See the [GinTutorial](GinTutorial.md) to learn how you can get started and [GinFaq](GinFaq.md) for more helpful hints.

For everything else, use the Guice types. Note that we do not support everything Guice does (and we never will), so keep in mind when you are developing with GIN. See the feature comparison below for more information.


## Features ##
Here we compare the current GIN trunk with Guice 1.0 and 2.0. The feature list is not complete, but should give you a rough idea about what works and what doesn't.

**A** = Available

**N/A** = Not Available (will probably never happen)

**P** = Planned (possible)

### Configuration ###
| **Feature**                     | **Guice 1.0** | **Guice 2.0** | **Guice 3.0** | **GIN 1.0** | **GIN 1.5** | **GIN 2.0** | **GIN 2.1** |
|:--------------------------------|:--------------|:--------------|:--------------|:------------|:------------|:------------|:------------|
| Modules                         | A             | A             | A             | A (`GinModule`) | A           | A           | A           |
| Provider bindings               | A             | A             | A             | A           | A           | A           | A           |
| Provider methods                | N/A           | A             | A             | A           | A           | A           | A           |
| toInstance bindings             | A             | A             | A             | N/A         | N/A         | N/A         | N/A         |
| Constant bindings               | A             | A             | A             | A           | A           | A           | A           |
| Binder.install                  | A             | A             | A             | A           | A           | A           | A           |
| Binder.requestInjection         | N/A           | A             | A             | N/A         | N/A         | N/A         | N/A         |
| Binder.requestStaticInjection   | A             | A             | A             | A           | A           | A           | A           |
| Binder.getProvider              | N/A           | A             | A             | P           | P           | P           | P           |
| Binder.requireBinding           | N/A           | A             | A             | P           | P           | P           | P           |
| (Custom) binding annotations    | A             | A             | A             | A           | A           | A           | A           |
| `@Named`                        | A             | A             | A             | A           | A           | A           | A           |
| Scopes                          | A             | A             | A             | A           | A           | A           | A           |
| Custom Scopes                   | A             | A             | A             | N/A         | N/A         | N/A         | N/A         |
| `@Singleton class Blah{}`       | A             | A             | A             | A           | A           | A           | A           |
| `@ImplementedBy`                | A             | A             | A             | A           | A           | A           | A           |
| `@ProvidedBy`                   | A             | A             | A             | A           | A           | A           | A           |
| `TypeLiteral`                   | A             | A             | A             | A           | A           | A           | A           |
| Circular dependencies           | A             | A             | A             | N/A         | N/A         | A           | A           |
| Private modules                 | N/A           | A             | A             | N/A         | N/A         | A           | A           |
| `@Optional` injection           | N/A           | A             | A             | A           | A           | A           | A           |
| Assisted Inject                 | A             | A             | A             | N/A         | A           | A           | A           |
| JSR330 Support                  | N/A           | N/A           | A             | N/A         | A           | A           | A           |
| Binder.requireExplicitBindings   | N/A           | N/A           | A             | N/A         | N/A         | N/A         | N/A         |
| toConstructor bindings          | N/A           | N/A           | A             | N/A         | N/A         | N/A         | N/A         |
| Multi/Mapbinder                 | N/A           | A             | A             | N/A         | N/A         | N/A         | A           |
| Custom injections               | N/A           | A             | A             | N/A         | N/A         | N/A         | N/A         |
| Throwing/checked provider       | N/A           | N/A           | A             | N/A         | N/A         | N/A         | N/A         |
| Inject Package Private Types    | A             | A             | A             | N/A         | N/A         | A           | N/A         |

### Runtime ###
| **Feature**                     | **Guice 1.0** | **Guice 2.0** | **Guice 3.0** | **GIN 1.0** | **GIN 1.5** | **GIN 2.0** | **GIN 2.1**|
|:--------------------------------|:--------------|:--------------|:--------------|:------------|:------------|:------------|:|
| Constructor injection           | A             | A             | A             | A           | A           | A           | A |
| Field injection                 | A             | A             | A             | A           | A           | A           | A |
| Method injection                | A             | A             | A             | A           | A           | A           | A |
| Method interception (AOP)       | A             | A             | A             | N/A         | N/A         | N/A         | N/A |
| Injector.injectMembers          | A             | A             | A             | A           | A           | A           | A |
| Injection of private members    | A             | A             | A             | A           | A           | A           | A |