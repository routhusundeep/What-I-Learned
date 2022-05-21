[paper](http://scg.unibe.ch/archive/papers/Duca06bTOPLASTraits.pdf)

“Multiple inheritance is good, but there is no good way to do it.”  -- Cook 1987

A class is primarily a generator of instances. A class has a secondary role as a unit of reuse. Unfortunately, these two roles conflict.
1. It can be difficult or impossible to factor out wrapper methods as reusable classes
2. conflicting features inherited from different paths may be difficult to resolve
3. overridden features may be difficult to access or compose.

A **mixin** (or **mix-in**) is a class that contains methods for use by other classes without having to be the parent class of those other classes. How those other classes gain access to the mixin's methods depends on the language. Mixins are sometimes described as being "included" rather than "inherited".

**Advantages:**
1.  It provides a mechanism for multiple inheritance
2.  [Code reusability](https://en.wikipedia.org/wiki/Code_reuse "Code reuse"): Mixins are useful when a programmer wants to share functionality between different classes.
3.  Mixins allow inheritance and use of only the desired features from the parent class, not necessarily all of the features from the parent class.

**Problems**:
Because mixin composition is implemented using inheritance, mixins are composed linearly. This gives rise to several problems
1. Suitable total ordering of features may be difficult to find, or may not even exist
2. “glue code” that exploits or adapts the linear composition may be dispersed throughout the class hierarch
3. The resulting class hierarchies are often fragile with respect to change, so that conceptually simple changes may impact many parts of the hierarchy.

In a nutshell, a trait is a set of methods, divorced from any class hierarchy. Traits can be composed in arbitrary order. The composite entity has complete control over the composition and can resolve conflicts explicitly, without resorting to linearization.

- Two roles are clearly separated: traits are purely units of reuse, and classes are generators of instances.  
- Traits are simple software components that both provide and require methods (required  methods are those that are used by, but not implemented in, a trait).  
- Classes are composed from traits, in the process resolving any conflicts, and possibly  providing the required methods.  
- Traits specify no state, so the only conflict that can arise when combining traits is a  method conflict. Such a conflict can be resolved by overriding or by exclusion.  
- Traits can be in-lined, a process that we call “flattening”: the fact that a method originates in a trait rather than in a class does not affect the semantics of the class.  
- Difficulties experienced with multiple inheritance disappear with traits, because traits  
are divorced from the inheritance hierarchy.  
- Difficulties experienced with mixins also disappear, because traits impose no composition order.

**Decomposition Problems**
The way in which we decompose our domain concepts into classes is not necessarily the right way to decompose the implementations of these classes into sets of features
- Duplicated Features: single inheritance sometimes forces code to be duplicated.
- Inappropriate Hierarchies: A common way of avoiding such code duplication is to implement certain methods “too high” in the hierarchy. The idea is that instead of duplicating a method, it is moved to a super class until it is available in all the classes where it is actually required. The top most class will be polluted with unrelated features.
- Duplicated Wrappers: Instead of changing/providing super class, you can modify features using late binding, which is inherent in mixins

 **Composition Problems**
 - Conflicting Features: Multiple inheritance face the classic "Diamond problem", Linearization solves this for mixins.
 - Lack of Control and Dispersal of Glue Code: A suitable total order may not exist by default, so the user should be able control it, and be able to resolve conflicts. We need glue code to do it correctly for mixins.
 - Fragile Hierarchies


#### TRAITS — COMPOSABLE UNITS OF BEHAVIOR
Classes retain their primary role as generators of instances, while traits are purely units of reuse. Classes are organized in a single inheritance hierarchy, thus avoiding the key problems of multiple inheritance, but the incremental extensions that classes introduce to their superclasses are specified using one or more traits.

Traits bear a superficial resemblance to mixins, with several important differences. 
- Several traits can be applied to a class in a single operation, whereas mixins must be applied incrementally. 
- Trait composition is unordered, thus avoiding problems due to linearization of mixins.
- Traits contain only methods, so state conflicts are avoided, but method conflicts may exist. A class is specified by composing a superclass with a set of traits and some glue methods. Glue methods are defined in the class and they connect the traits together; i.e., they implement required trait methods (possibly by accessing state), they adapt provided trait methods, and they resolve method conflicts.

Trait composition respects the following three rules:  
- Methods defined in a class itself take precedence over methods provided by a trait. This allows glue methods defined in the class to override methods with the same name provided by the traits.  
- Flattening property. A non-overridden method in a trait has the same semantics as if it were implemented directly in the class.  
- Composition order is irrelevant. All the traits have the same precedence, and hence conflicting trait methods must be explicitly disambiguated

A conflict arises if we combine two or more traits that provide identically named methods that do not originate from the same trait.
In addition, traits allow method aliasing; this makes it possible for the programmer to introduce an additional name for a method provided by a trait.


Did not read through the formal Definitions and Semantics