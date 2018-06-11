# The Joy Prime Type System, for Use in Clojure

This library provides the entire [Joy Prime](https://github.com/joy-prime/joy-prime) (Joy') runtime type system
for use in Clojure programs. Although Joy' provides additional language support, it deeply embraces
its Clojure roots, and Clojure is a first-class client of the Joy' type system. This type system does
reuse and modify concepts and terminology from Java, such as "class" and "interface", so the reader
must deeply embrace namespacing!

The Joy' type system is inspired by the alpha version of [Clojure spec](https://clojure.org/guides/spec).
As with all offspring, however, this inspiration is evident partly in where Joy' took the same path
as spec and partly in where it went bushwhacking in another direction. Joy' 
embraces spec's vision of optionally, gradually expressing contracts in the runtime language. But unlike
spec, Joy' treats interfaces and types as first-class notions that deserve at least as much support as 
data "shapes". Joy' also departs from spec (at least spec alpha) in reifying its types as values
that are naturally understood and verified in that same type system.

## The Joy' Type System 

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
  
Joy' explores a new point in the design space: an expressive and rigorous type system that fulfills
a type system's traditional role for thinking and communicating about the code, but that is designed 
for runtime validation instead of compile-time validation. Runtime validation includes both 
switchable runtime checks and automatic [generative testing](https://nofluffjuststuff.com/conference/raleigh/2013/08/session?id=29335).

With Java concepts out of the (direct) picture, Joy' reimagines some of them and reuses the names:

* Joy' has its own notions of "type" and "class". A Joy' type describes a constrained set of Joy' values,
  such as "a vector of integers". A Joy' class identifies a constrained set of Joy' types, such as "vectors".
 
* From the Joy' programmer's perspective, classes and types are created, manipulated, and applied at 
  runtime rather than at compile time. A Joy' compiler or analysis tool may perform static
  analysis for various reasons (such as to detect errors or to optimize code) but this is neither 
  the main purpose of Joy' classes and types nor how programmers are encouraged to think about them.

* Every Joy' value has a class, which can be cheaply obtained from the value.

* Every Joy' class has a corresponding "bare class type". For example, the vector class has a corresponding
  bare vector type. All values having that class are members of that bare class type.

* If the set of types identified by a class has additional members beyond the bare class type, 
  then these additional "class types" are subtypes of the bare class type. For example, all vector types
  are subtypes of the bare vector type.

* Joy' types are also Joy' values and can be freely manipulated at runtime.

* Because types are values, types themselves have classes and types. Each Joy' class has an 
  associated "metatype", which is the type of that class' types.

* A class can have one or more "interfaces", which are "interface implementations" of "interface types".
  An interface type specifies a set of Clojure symbols that refer to specially declared functions:
  the "methods" of the interface. Each of these methods takes an instance of the interface type as its
  first parameter and can be invoked directly by client code. An interface implementation is a map from 
  method to function. 
  
  
  Each of those keywords identifies a function type whose first parameter is declared as the interface type.
  An interface implementation is a map from each method keyword to a function of the appropriate type.
  
An interface is a set of function types called 
  "methods". Every interface has an associated "interface type", which is a supertype of all 
  types that support the interface. Every method on an interface takes a member of the interface type 
  as its first parameter.

A Joy' class definition can specify any or all of the following: 
    
* *parent types*: supertypes of the bare class type. These are listed in priority order for the purpose
  of choosing the implementation of an interface if more than one is available. Additional parent types
  can be appended to a previously defined class.

* *interfaces*: can be invoked on all values that are members of the bare class type

* *interface implementations*

 
    
    
  * A class can implement some or all 
  
  
    All types within a class are subtypes of the raw class and some of them are subtypes of each other. 
    Every Joy' value belongs directly to a single class, which is known at runtime. It is always possible to
    determine at runtime whether a given Joy' value belongs to a given Joy type.
    
    
  * Joy' classes are  
    
    A child class automatically supports all interfaces 
    supported by its ancestor classes (else it would not be a subtype of its ancestor classes). The child may inherent 
    its ancestors' implementations or provide its own. Unrelated classes can support the same interfaces, in which case 
    they must each provide their own implementations. A class can have multiple parents, and inherits
    interface implementations through all of them parents, so parent classes can act as mixins. 
    A class' parents are listed in priority order for choosing an implementation when multiple ancestors provide
    implementations of the same interface.
    
  * Each primitive Clojure datatype has a Joy' class, and can be constructed through normal Clojure functions.
  
  * Each user-defined Joy' class has one or more constructors. Each constructor takes a type (the runtime 
    representation of a type) belonging to that class as a first argument. Constructors are invoked
    by client code to create instances of the class, and also are used for generative testing.  
    
A Joy' class definition can specify any or all of the following:

* *interface*: the names and types of a set of pure functions that can be invoked on any member of this class

* *implementation*: bodies for the interface functions

* *supertypes*: an ordered list of supertypes of this class 
  
Each Joy' class definition also defines its *type class*: the class of its types. A Joy' type is
a value that specifies constraints on values of a particular class -- the type's *value class*.

A type class usually specifies an interface that defines one or more constructors, which are simply
methods on the type class that return members of the value class.

Each Joy' class has zero or more parent classes. The set of parent classes is open: new parents can be 
added to a class after it is defined. The set of parent classes is ordered: each interface method invocation
uses the first implementation in the ordered set of parent classes. When new parents are added after the child
class is defined, they can only be added to the end of the list. Newly added parents cannot directly or 
(through ancestors) indirectly specify representations.

Here are some common ways of using parent classes:

* A parent class may only specify an interface, leaving the implementation and representation up to the child.

* A parent class may specify an interface and an implementation, in which case it is effectively a mix-in.

* A parent class may specify a representation, which may be used as-is or extended by the client.

Every Joy' type class is a subclass of `:joy.core/type`, which defines the following interface:
```clojure
(new "Constructs a value of this type from the map `m`. If this type specifies a representation,
      then `m` must be an instance of that representation and `new` is a no-op." 
 (! [this (! m :joy.core/Map)] Value))

(old "Creates a map that can be passed to `new` to exactly reproduce `this`." 
 (! [this (! v Value)] :joy.core/Map))
```

When a class specifies a representation, this completely defines values of the class: it is fine
to create members of the class by creating a map with the specified keys. However, a parent class
representation is presumed to only partially define representations of subclasses. If a subclass also
defines a representation, its keys must be a superset of (including possibly the same as) the keys of 
the parent class. If a subclass does not define a representation, then a map containing a parent class'
represent keys cannot be presumed a valid instance of the subclass.

When neither a class nor any of its superclasses specify a representation, values of the class are 
opaque to Joy' code. All Joy' operations on values of the class must be implemented using methods of 
interfaces defined by the class or its superclasses. All Joy' construction of values of the class must
be done through methods of interfaces defined by the class' type class.

Some opaque Joy' values are implemented in Clojure or Java. This includes values defined in `joy.core`,
which are considered "primitive" Joy' values. Clojure interoperability provides additional constructors 
and operators for such values.

Every Joy' value can be represented in [EDN](https://github.com/edn-format/edn) (Clojure's Extensible Data Notation)
as a map with two keys:

* `:joy.core/class` the keyword that identifies the value's class
* `:joy.core/new-map` the map (possibly returned by `old`) to be passed to `new` to reproduce the value

Every built-in EDN element represents a primitive Joy' value.
