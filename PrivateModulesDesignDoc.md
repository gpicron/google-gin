# Introduction #

Supporting private modules required reworking a large portion of the Gin internals in order to support a hierarchical collection of bindings.  Specifically, installing a new private module creates a new collection of bindings that is a child of the current collection.

In addition to the changes necessary to support the hierarchical binding collections, the process for creating all of the implicit bindings had to be changed significantly in order to determine in which binding collection each implicit binding should be placed.

This document summarizes the changes that had to be made to Gin, and attempts to explain the algorithm for creating and positioning the implicit bindings (resolution).

# Changes throughout Gin #

  * Split out the code for storing bindings from `BindingProcessor` into `GinjectorBindings`
  * Modify `GinjectorBindings` to be hierarchical
    * `GinjectorGenerator` had to be updated to produce code for all `GinjectorBindings` in the hierarchy.
    * `ParentBinding` and `ChildBinding` were added to the types of bindings, to support inheriting a binding from a parent and receiving an exposed binding from a child, respectively.
    * Many files were changed to respect the new hierarchical bindings.
  * Create a `BindingResolver` that performs the resolution steps.  Initial version used a tree-based algorithm that failed horribly when cycles were present.  The algorithm documented here is version 2, which deals much more gracefully with cycles.
  * Create a class `Dependency` for representing dependencies between types (eg Foo -> Bar)
    * Modify the codebase to return `Set<Dependency>` instead of `Set<Key<?>>` any places dependencies were managed, and correctly set the optional/lazy bits.

## Dependency Graph ##

The dependency graph has `Key`s as nodes and dependencies between `Key`s as edges.  Each edge is labeled with some additional information:
  * Eager:  An eager edge means that the target key is injected when the source key gets injected.  This is the case of most dependencies.  An example of a non-Eager dependency would be the dependency from `Provider<Foo>` to `Foo`.  The injection of `Foo` only happens when the `get()` method in the provider is called.
  * Optional: An optional edge indicates that the target of the dependency is @Inject(optional=true) in the source.  If we're unable to resolve a binding for the target it does not represent an error.

## Injector Hierarchy ##

Installing a private module creates a new child injector that contains the bindings from the private module.  As in Guice, child injectors can inherit bindings from parent injectors.  Bindings from a child injector are only available to the parent injector if they are explicitly exposed.

It is worth noting that these injectors do not directly correspond to the Ginjector interface in user code.  Methods in the user Ginjector expose bindings available to the root injector in the hierarchy to user code, and as such any keys that the Ginjector must expose are also dependencies from the root injector.

# Resolution #

The resolution process is roughly responsible for making Gin injectors behave similarly to Guice injectors.  The process by which Guice resolves each binding is documented [here](http://code.google.com/p/google-guice/wiki/BindingResolution).  Unfortunately, since Gin has to produce the code for the Ginjector at compile time it is not possible to use an identical process to Guice.

To accomplish similar behavior, we satisfy dependencies for each injector according to the following:
  1. If a child injector needs a binding that a parent has available, the child will use the parent's binding.
  1. If an injector needs a binding that isn't already available to it, it will cause the binding to be created, and positioned as high as possible subject to the following constraints:
    * All of the bindings that it depends on are available (either at or above the injector)
    * There is not already a binding for that key at any of the child injectors.

There are several observations about the above process that are important to make.  The most important is that the order of the visiting the injectors for resolution does not matter.  Since we always place things as high as possible, and the only thing that constrains positions initially is the pre-existing bindings, it is impossible for any decisions we make to affect future positions.

Furthermore, resolving the bindings in a given injector will only add bindings to that injector and higher (closer to the root of the hierarchy).

## Details ##

The resolution algorithm occurs in several stages, split up into a variety of classes.  It is called once for each injector in the hierarchy.  In the following description, the "root injector" refers to the injector that is being resolved.

### DependencyExplorer ###
Builds the Dependency Graph, starting with all the dependencies of the root injector.  Attempts to ensure that all dependencies are satisfied using either an available binding or a generated implicit binding.

The output from this stage includes the dependencies induced from the root, the implicit bindings that were created, and any keys that were unavailable and couldn't be implicitly created.

This performs a traversal over dependency graph, starting with the unresolved dependencies in the injector.  For each key, it checks to see if it is already available to the root injector.  If it is already available, it has no outgoing edges, since we can just re-use the binding that already exists.  If it isn't available, it attepmts to create an implicit binding using `ImplicitBindingCreator`.  If an implicit binding is created, the outgoing dependencies are added to the graph and visited recursively.  If creating the implicit binding failed, then the key is recorded as unavailable, and an appropriate error is stored for later reporting (by `UnresolvedBindingValidator`).

### UnresolvedBindingValidator ###
Ensures that a solution exists by detecting error conditions.  Reports errors if there are any eager cycles or unresolvable required bindings.  Removes unresolvable optional bindings.

If there any eager cycles (cycles that pass through only eager edges) reported by `EagerCycleFinder` this will log an error and cause resolution to fail.  If there are any bindings that couldn't be created or bindings that cannot be correctly positioned (because they key is already bound in a child of the root injector, and so creating the implicit binding would cause a double binding) it will report an error (if the key is required) or remove the dependency (if it was optional).

If an error is detected, it will use `PathFinder` to compute the shortest path explaining why that key was deemed necessary, and report that in addition to the error.  The later passes (`BindingPositioner` and `BindingInstaller`) are only run if there were no errors while resolving required keys.

### BindingPositioner ###
At this point we know a solution exists, so this step determines the final position of each implicit binding, placing each as high as possible subject to the constraints mentioned.

It creates an initial "guess" that has each implicit binding positioned as high as possible without creating a double binding (satisfying the 2nd constraint above) and each pre-existing binding placed in the appropriate injector.  It then iterates to fix-point, updating the position of each key, K, with the following equation:
```
   Position(K) = Min({Position(X) | X \in DependenciesFrom(K)} U {Position(K)}
```
Note that updating the position can only move it down, because the old position is included in the set of injectors that we're taking the minimum over.  After updating the key, if we've caused it to change, we queue up all keys that depend on K for revisiting.

### BindingInstaller ###
This pass is responsible for installing all of the implicit bindings into the positions specified by BindingPositioner.

Installing a binding into a given injector requires adding the binding to the appropriate `GinjectorBindings` set, and also ensuring that all of the keys that the binding depends on are bound in that `GinjectorBindings` set.  If they aren't already available, this step adds a `ParentBinding` to the `GinjectorBindings`, that will instruct the `GinjectorGenerator` to output the code necessary to allow the child to access the parents binding.