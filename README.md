# The Joy' Type System, for Use in Clojure

**PAUSED INDEFINITELY -- instead see wisdom-kotlin**

This library provides the entire [Joy Prime](https://github.com/joy-prime/joy-prime) (Joy') runtime type system
for use in Clojure programs. Although the Joy' language provides additional linguistic support, 
Clojure is a first-class client of the Joy' type system. 

The Joy' type system is inspired by the alpha version of [Clojure spec](https://clojure.org/guides/spec).
As with all offspring, however, this inspiration is evident partly in where Joy' took the same path
as spec and partly in where it went bushwhacking in another direction. Joy' embraces spec's vision
of optionally, gradually expressing contracts in the runtime language. But unlike spec, 
Joy' treats interfaces and types as first-class notions that deserve at least as much support as 
"data shapes" like vectors and map keys. Joy' also departs from spec (at least from the alpha version of spec)
in reifying its types as values that themselves have natural Joy' types.

## Motivations 

Joy's type system design is driven by the following beliefs:

Clearly specified and mechanically enforced interfaces are essential.

Type systems are great for helping programmers think and communicate about their code.

Using type systems for compiler-enforced program invariants is appealing, but
leads to a painful and inescapable dilemma. Compile-time type systems form a complexity
spectrum that is painful from end to end -- it has no sweet spot:
  
* Very simple type systems (such as C) are not expressive enough to capture
  many of the invariants that programmers care about.
  
* Fairly simple type systems (such as Go, or Java before generics)
  artificially limit what programmers can express, such as by making it impractical to express
  generic containers or functional primitives like `map` and `reduce`.
    
* As compile-time type systems become more expressive (such as by adding generics to Java,
  or even more so in Haskell), they substantially increase language complexity and cognitive load. 
  They also get in the way of incremental development, because they require an entire program 
  to be provably consistent before it can be run.
    
* As programmers strive to capture more of their invariants, they push even fairly complex type systems 
  (such as Haskell) to steadily increase in complexity. The type system becomes a language of its own.
  (See, for example, the 
  [Glasgow Haskell Compiler's numerous language extensions](https://downloads.haskell.org/~ghc/8.2.1/docs/html/users_guide/glasgow_exts.html).)
  A program's behavior depends on the decisions of a complex type inference engine, which the programmer
  is expected to understand, anticipate, and nudge in desired directions.  
    
* Some of this complexity can be escaped through dependently typed languages such as 
  [Idris](https://www.idris-lang.org/), which dissolves much of the separation between types and values
  and allows the core language to operate on both. At this point on the spectrum, however, 
  the inference engine is even more complex and more central in program behavior. 
  The programmer is often forced to explicitly prove program invariants that are asserted
  through the type system.
  
## Overview
  
Joy' explores a new point in the programming language design space: an expressive and rigorous type system 
that fulfills a type system's traditional role for thinking and communicating about the code, but that is 
designed for runtime validation instead of compile-time validation. Joy's runtime validation includes both 
switchable runtime checks and automatic
[generative testing](https://nofluffjuststuff.com/conference/raleigh/2013/08/session?id=29335).

Here is how Joy' resolves the dilemma described above for compile-time type systems:

* Joy' types are runtime Clojure values. So there is no separate language for expressing types.

* Joy' types can be computed and manipulated using Clojure (or Joy') itself. So there is no
  separate language for manipulating types.

* Joy' types are verified at runtime (if at all). So although the programmer can freely compute
  types, this does not force them into proving program invariants that are asserted
  through the type system.
  
* Joy's runtime behavior is not implicitly changed by type inference. Because Joy'
  types can be computed and manipulated at runtime, it is is straightforward to write code
  whose behavior depends on type checking, but this behavior is explicitly coded in Clojure (or Joy'). 
  
Joy's type system is described in terms that are familiar from Java and other languages, such as 
"class", "type", and "interface". Although at an abstract level these terms have their conventional 
meanings, Joy's concrete meanings are unique. In particular, they each refer to a different construct 
than the same Clojure or Java term, so it is important to firmly anchor them in a mental Joy' namespace! 

A Joy' type describes a constrained set of Joy' values -- the "instances" of the type -- 
such as "vector of integers". A Joy' type does not intrinsically have a name nor an identifier, 
although types are commonly stored as the values of Clojure symbols. (Also, we will see later
that a Clojure keyword can be used as a special kind of type, called a "bare class type".)
A Joy' type is a Joy' value, and is itself described by a "metatype". (More on metatypes later.)

A Joy' class identifies a constrained set of Joy' types, such as "vectors". It is identified by
a Clojure keyword, which conventionally uses capitalized camel case, such as `::my/TaskManager`.
There is a root class `::joy/Any` that is a superclass of all other classes. There is a leaf class
`::joy/Void` that is a subclass of all other classes but has no values.

From the Joy' programmer's perspective, classes and types are created, manipulated, and applied at 
runtime rather than at compile time. A Joy' compiler or analysis tool may perform static
analysis for various reasons (such as to detect errors early or to optimize code) but this is neither 
the main purpose of Joy' classes and types nor how programmers are encouraged to think about them.

Every Joy' value has a class, which can be cheaply obtained from the value. (We might be tempted
to say "obtained at runtime", but from this point forward we will count on the reader to remember
that all of this, at least conceptually, happens "at runtime".)

Every Joy' class has a corresponding "bare class type". For example, the vector class has a corresponding
bare vector type. All values having that class, or any of its direct or indirect subclasses, are instances of
the bare class type. As a simpler way to say "instance of the bare class type", we also say
"instance of the class". Wherever a type can be used in Joy', a keyword can also be used to mean the 
bare class type of the class identified by that keyword. 

If the set of types identified by a class has additional instances beyond the bare class type, 
then these additional "class types" are subtypes of the bare class type. For example, all vector types
are subtypes of the bare vector type. A given type is both a superclass and a subclass of itself.

It is always possible to determine whether a given type is a subtype of another.

Joy' types are also Joy' values and can be freely manipulated.

Because types are values, types themselves have classes and types. Each Joy' class has an 
associated "metatype", which is the type of that class' types. A metatype's class is referred
to as a "metaclass" when that seems clarifying.

Every Clojure value is a Joy' value. (A Clojure programmer may prefer that we simply refer to 
these as "Clojure values" in this documentation, but here we show a slight bias toward Joy' programmers.)
Some Clojure values, such as functions, maps, and sequences, have rich class and type descriptions in 
the Joy' type system. When a Clojure value does not have a rich Joy' description, its Joy' class
name is mechanically derived from its Java class name: `org.foo.BarBaz` becomes `:org.foo/BarBaz`.

## Parameterized Types

A Joy' type may have parameters. These are declared by a class. A subclass inherits all type parameters
of its superclasses.

Type parameters are applied in three ways: 

* A type can be constructed with parameters. This is done through a "type parameter map", provided
  on type construction, where the keys are parameter names and the values are parameter values.
  The type may directly consume some of these parameters, while others may only be used by
  enclosed types as described below, or may be unused.
 
* A type can pick up parameters from its surrounding type. This is done by using 
  `::joy/for(param-name)` as a key in the type parameter map. The value for that key
  must be a function that takes the surrounding type parameter map as its sole parameter. Note
  that using functions other than keywords for this purpose will, for the foreseeable future,
  defeat subtyping that involves that parameter. It is a runtime error for a type parameter map
  to have both an explicit value and a computed value for the same parameter name.
  
* If a parameter is not specified in either of the above ways, then we call it a "blank" type parameter. 
  If a blank parameter's type implements the `::joy/HasDefault` interface, then the parameter is taken 
  to have that value. `::joy/Type` implements `::joy/HasDefault` with the default value of `::joy/Any`.

Subtype relationships are computed after applying parameters as above. Given a covariant parameter P
whose value is a type, for type S to be a subtype of a type T, the value of P for S must be a subtype
of the value of P for T. Given a covariant parameter P whose value is not a type, the values of P for 
S and T must be equal. All contravariant parameters are types; in that case the value of P for S must
be a *supertype* of its value for T.

Given a parameterized type and a value that is a instance of the type, the `::joy/type-params`
method returns a map with complete, specific type parameters for that value. "Specific" means that
type parameters which represent types are instantiated with the narrowest subtype that fits.

## Interfaces

A class can have one or more "interfaces", which are "interface implementations" of "interface types".
An interface type is a instance of the `::joy/Interface` class and specifies a set of Clojure symbols 
that refer to specially declared functions: the "methods" of the interface. Following the approach used by 
Clojure spec for specifying maps, a `::joy/Interface` does *not* directly specify the types of
its methods. Rather, a given (namespaced) method symbol always has the same function type, which
is available in the method's metadata. The same method symbol can be used in multiple interfaces.  

Each method takes an instance of the interface type as its first parameter, conventionally called `this`.
Methods are invoked directly by client code.

An interface implementation is a map from method symbols to method implementation functions. This map is
used when defining a class; it is *not* used by clients of the interface. Rather, each method is a 
Clojure function that the client directly invokes; the method dispatches on the class of its first argument 
to the appropriate implementation function.

## Class Definitions

A Joy' class definition can specify the following: 
    
* *metatype*: the type of the class' types. If this is unspecified, then as a default, the class 
  is provided with a metatype that has a single value representing the bare class type.

* *parent types*: supertypes of the bare class type. Parent types are listed in priority order for the 
  purpose of choosing the implementation of an interface if more than one is available. The list of parent 
  types cannot be changed after the class is defined (unless the class is entirely redefined). Every class
  has the bare class type of `::joy/Any` as an implicit supertype. It is fine to not explicitly list
  any parent types. Defining parent types has two effects: 
  
** The class of each parent type becomes a superclass of this one. As a result, this class and its
   descendants inherit the interfaces and type parameters (see below) of the superclass. Instances
   of this class are guaranteed to be instances of the superclass.
   
** Also, instances of this class are guaranteed to be instances of each parent type. Whenever type
   checking verifies that a value is an instance of this class, it also verifies that the value is
   an instance of the parent type. 

* *interfaces*: interface implementations that apply to values that are instances of the bare class type. 
  Additional interfaces can be added after the class is defined.
  
* *parameters*: the names (keywords) of type parameters. When these represent types, they are taken
  as covariant parameters for computing subtypes. These are inherited by subclasses.

* *contravariant parameters*: the name (keywords) of contravariant type parameters. These parameters must all
  represent types. They are inherited by subclasses.
  
## Function Types

All Clojure functions are instances of `::joy/Function`. Clojure functions can also be declared with 
metadata that declares them to have more specific function subtypes. 
This metadata is associated with the function object itself (*not* with the function's
symbol as would be more typical in Clojure). It is the value of the key `::joy/fn-type`, 
which must be a subtype of the bare function type.

`::joy/Function` has no parent types beyond `::joy/Any`. As with all Joy' classes, 
it has an open set of subclasses. These include a built-in subclass `::joy/TypedFunction` whose
metaclass is `::joy/TypedFunctionType`. Instances of that metaclass (thus subtypes of `::joy/TypedFunction`)
are constructed with a factory macro `Fun`. (So that's both fun with types and fun with a capital "F".
What else could we call the functions of Joy?)

Each parameter of the `Fun` macro represents a possible parameter arity of the function and is
expressed in a syntax that extends the Clojure parameter syntax with support for type constraints.
We will explain the semantics of `::joy/TypedFunction` by explaining this syntax.

If `[...]` is a legal Clojure function parameter syntax, then `(Fun [...])` is a legal invocation
of the `Fun` macro. For example, `(Fun [coll x])` is legal. So is `(Fun [call x & xs])`.

Multiple arities are represented as multiple arguments to `Fun`. For example,
`(Fun [coll x] [call x & xs])`.

Any top-level binding form in a parameter list can be surrounded a type declaration `(! `form` `type-expr`)`
to require that the argument passed to that form must have a particular type. type-expr is any expression that
evaluates to a type. For example, remembering that the keyword identifying a class can be used to
represent the bare class type, we can write `(Fun [(! coll ::joy/Collection) x])`.

Type declarations never change the binding behavior of a parameter list. In particular, if type declarations
are removed, the binding behavior is exactly as normal for Clojure.

The same `(! ...)` syntax can be used to specify a type for nested binding forms, including for
individual symbols that will be bound. For example, although it doesn't add much, we could
write `(Fun [(! [fst snd] ::joy/Vector) z])` to indicate that the argument supplied for `[fst snd]`
must be a vector.

The `(! ...)` syntax can also be wrapped around the entire parameter vector to supply a return type.
For example: `(Fun (! [coll x] ::joy/Collection))`.

We can freely combine the above usages. For example:
```
(Fun (! [(! coll ::joy/Collection) x] 
        ::joy/Collection))
```  

Binding is performed before type expressions are evaluated or types are checked. Also, type expressions
are evaluated in a context where the bound symbols are available as variables. So a type expression 
anywhere in the same parameter list (the same arity) can freely refer to symbols bound by the parameter
list.

Type expressions must be pure, in the sense of referentially transparent -- consuming neither external data
nor mutable data. Otherwise, the behavior of type checking is undefined. Implementations may make reasonable
behavior to detect this and indicate an error.

A `Fun` type S is a subtype of another, T, if the implementation is able to verify that all argument values
which are legal for T are also legal for S, and that all return values which are legal for S are also legal
for T. In other words, subtyping is contravariant in function parameters but covariant in the function
return value, but a subtype relationship exists only if the implementation can verify those relationships.
Over time, the Joy' type specification will be extended to make some guarantees about which function subtype
relationships can be verified.

## Metatypes

Every Joy' metaclass is a subclass of `::joy/Type`. A metaclass usually has an interface that defines 
one or more constructors, which are methods that return instances of the value class.

Although all metaclasses descend from a common superclass, 
the metaclass hierarchy does not mirror the class hierarchy of the corresponding value classes. This would
not make sense because an interface on a superclass cannot always be sensibly applied to a subclass. As
a simple example, consider a superclass Person with a subclass Employee. The subclass constructor may
require a Business parameter that has no sensible default value.

That said, sometimes we want to express a metatype whose instances are guaranteed to be subtypes
of a particular type. To support this, `::joy/Type` has a type parameter `::joy/for` whose value is
the type represented by that metatype.

`::joy/Type` has the following interface:

```
(definterface IType new-map-type new)

(defmethod from-map-type
  "Returns the `::joy/Map` subtype that must be passed to `new` to construct an instance of this type."
  [this] (type ::joy/Type {::joy/for ::joy/Map))

(defmethod new
  "Constructs a value of this type from the map `m`."
  [this (! m (from-map-type this))]
  Value)
```

There is a related method on `::joy/Any`:
```
(defmethod as-map
  "Creates a map that can be passed to `new` to exactly reproduce `this`."
  [this]
  joy/Map)
```
