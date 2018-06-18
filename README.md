# The Joy' Type System, for Use in Clojure

This library provides the entire [Joy Prime](https://github.com/joy-prime/joy-prime) (Joy') runtime type system
for use in Clojure programs. Although Joy' provides additional language support, Clojure is a first-class client 
of the Joy' type system. 

The Joy' type system is inspired by the alpha version of [Clojure spec](https://clojure.org/guides/spec).
As with all offspring, however, this inspiration is evident partly in where Joy' took the same path
as spec and partly in where it went bushwhacking in another direction. Joy' 
embraces spec's vision of optionally, gradually expressing contracts in the runtime language. But unlike
spec, Joy' treats interfaces and types as first-class notions that deserve at least as much support as 
data "shapes". Joy' also departs from spec (at least spec alpha) in reifying its types as values
that are naturally understood and verified in that same type system.

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
    
* As type systems become more expressive (such as by adding generics to Java, or even more so in Haskell),
  they substantially increase language complexity and cognitive load. They also get in the way of 
  incremental development, because they require an entire program to be verified consistent before
  it can be run.
    
* As programmers strive to capture more of their invariants, they push even fairly complex type systems 
  (such as Haskell) to steadily increase in complexity. The type system becomes a language of its own.
  (See, for example, the [Glasgow Haskell Compiler's numerous language extensions](https://downloads.haskell.org/~ghc/8.2.1/docs/html/users_guide/glasgow_exts.html).)
  A program's behavior depends on the decisions of a complex type inference engine, which the programmer
  is expected to understand, anticipate, and nudge in desired directions.  
    
* Some of this complexity can be escaped through dependently typed languages such as [Idris](https://www.idris-lang.org/),
  which dissolves much of the separation between types and values and allows the core language to operate on both.
  At this point on the spectrum, however, the inference engine is even more complex and more central in program
  behavior. The programmer is often forced to literally construct proofs of program invariants that are asserted
  through the type system.
  
## Overview
  
Joy' explores a new point in the design space: an expressive and rigorous type system that fulfills
a type system's traditional role for thinking and communicating about the code, but that is designed 
for runtime validation instead of compile-time validation. Runtime validation includes both 
switchable runtime checks and automatic [generative testing](https://nofluffjuststuff.com/conference/raleigh/2013/08/session?id=29335).

Joy's type system is described in terms that are familiar from Java or other languages,
such as "class", "type", and "interface". Although at an abstract level these terms have their
conventional meaning, Joy's specific usage is unique. In particular, none of these terms mean the
same as in Clojure or Java. It is important to firmly anchor these terms in a mental Joy' namespace! 

A Joy' type describes a constrained set of Joy' values, such as "a vector of integers". 

A Joy' class identifies a constrained set of Joy' types, such as "vectors".

From the Joy' programmer's perspective, classes and types are created, manipulated, and applied at 
runtime rather than at compile time. A Joy' compiler or analysis tool may perform static
analysis for various reasons (such as to detect errors or to optimize code) but this is neither 
the main purpose of Joy' classes and types nor how programmers are encouraged to think about them.

Every Joy' value has a class, which can be cheaply obtained from the value.

Every Joy' class has a corresponding "bare class type". For example, the vector class has a corresponding
bare vector type. All values having that class are members of that bare class type.

If the set of types identified by a class has additional members beyond the bare class type, 
then these additional "class types" are subtypes of the bare class type. For example, all vector types
are subtypes of the bare vector type.

It is always possible to determine at runtime whether a given type is a subtype of another.

Joy' types are also Joy' values and can be freely manipulated at runtime.

Because types are values, types themselves have classes and types. Each Joy' class has an 
associated "metatype", which is the type of that class' types.

A class can have one or more "interfaces", which are "interface implementations" of "interface types".
An interface type specifies a set of Clojure symbols that refer to specially declared functions:
the "methods" of the interface. Each method takes an instance of the interface type as its
first parameter and can be invoked directly by client code. An interface implementation is a map from 
method symbol to functions that implement the method for that class. 

## Class Definitions

A Joy' class definition can specify the following: 
    
* *metatype*: the type of the class' types. If this is unspecified, the class is provided with a metatype
  that has a single value representing the bare class type.

* *parent types*: supertypes of the bare class type. These are listed in priority order for the purpose
  of choosing the implementation of an interface if more than one is available. The list of parent types
  cannot be changed after the class is defined (unless the class is entirely redefined).

* *interfaces*: defined for all values that are members of the bare class type. Additional interfaces
  can be added after the class is defined. 
  
## Function Types

All Clojure functions are members of the bare function type `joy/bare-type(::joy/function)`. 
Clojure functions can also be declared with metadata that declares them to have more specific function subtypes. 
Note that this metadata is *not* associated with the function's symbol; rather, it is associated with
the function object itself. It is the value of the key `::joy/type-of-fn`, which must be a subtype of the 
bare function type.

As with all Joy' classes, `::joy/function` has an open set of subclasses and subtypes. Joy' provides 
a subclass `::joy/args-ret-fn` that simply provides a type for the function's arguments (as a single vector) 
and the function's return value. Types in this class have values `::joy/type-of-args` and `::joy/type-of-ret`.
Subtyping for this class is contravariant in `::joy-type-of-args`.

In the future, Joy' will likely provide a different subclass that supports [dependent type](https:/en.wikipedia.org/wiki/Dependent_type) 
relationships among arguments and parameters.

## Heterogeneous Sequence Types

Because Joy's main function subclass models a function's arguments as a single, heterogeneous vector,
It is important that Joy' has a class `::joy/type-regex-seq` that naturally expresses the patterns
that arise in argument vectors. This class follows Clojure spec alpha in modeling a heterogeneous seq
as a regular expression of element types. This is flexible, familiar to most programmers, and has a 
natural subtype structure.
  
## Metatypes

A metatype usually has an interface that defines one or more constructors, which are methods that return members
of the value class.

Every Joy' metatype is a subclass of `:joy.core/type`, which defines the following interface:
```
(new "Constructs a value of this type from the map `m`. If this type specifies a representation,
      then `m` must be an instance of that representation and `new` is a no-op." 
 (! [this (! m :joy.core/Map)] Value))

(old "Creates a map that can be passed to `new` to exactly reproduce `this`." 
 (! [this (! v Value)] :joy.core/Map))
```
