# The Joy Prime Programming Language (Vaporware)

Joy Prime is an idea for a new language based on Clojure.
The "Prime" suffix honors the original [Joy language](https://en.wikipedia.org/wiki/Joy_(programming_language))
created by Manfred von Thun of La Trobe University, Australia. (He died on October 23, 2011.) After spelling
out "Joy Prime" once to provide a search-engine anchor, the language's preferred spelling is Joy'. 
The preferred pronunciation is simply "joy".

Mostly, Joy' is Clojure. It embraces Clojure's syntax, most of its data types, and most of its implementation.
Joy' continues Clojure's approach of preferring and encouraging immutability without forcing it.
 
However, Joy' makes certain changes that result in a substantially different programming experience.
In particular, Joy' embraces Clojure's preference for runtime types instead of compile-time types,
but carries this considerably further. Joy' has a well-developed type system that has
much of the flavor of [dependent types](https://en.wikipedia.org/wiki/Dependent_types),
but that is designed for runtime verification instead of compile-time verification.

Joy's design is partly driven by a belief that Clojure awkwardly blends some great new ideas
with its Java roots. Joy's type system deeply embraces Clojure's use of heterogenous maps with 
namespaced keywords as keys, including Clojure's momentum toward giving the same semantics
to any given namespaced keyword wherever it is used. Joy' drops Clojure records and protocols; 
these are an awkward fit with even Clojure's use of heterogeneous maps, and certainly at odds 
with Joy's type system.

In fact, Joy' drops all direct Java interoperability. Instead, Joy' is fully interoperable with Clojure,
which gives it a natural indirect interoperability with Java. Whereas Clojure deliberately exposes all
Java semantics, Joy' attempts to build an airtight abstraction that hides all Java semantics. For example,
Clojure functions such as `type` that expose the underlying Java implementation are available through 
Joy's Clojure interoperability, but are treated at the Joy' level as foreign interfaces.

Joy' also drops Clojure multimethods. Instead of multimethods and protocols, Joy' tries to provide 
a clearer, better reified, more expressive, and more verifiable "interface" abstraction, with 
new approaches to type hierarchies and polymorphism.  

## Joy's Type System 

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

* With Java concepts out of the (direct) picture, Joy' reimagines some of them and reuses the names:

  * Joy' has its own notions of "type" and "class". Joy' classes are discretely identified categories of types. 
    Classes are organized into a hierarchy, which is a subtype relationship. All types within a class 
    are subtypes of the raw class, and some of them are subtypes of each other. Every Joy' value belongs to
    a single class, which is known at runtime. It is always possible to determine at runtime whether a given
    Joy' value belongs to a given Joy type. Classes and types define the structure of their values but
    do not directly define operations on those values.
    
  * Joy' specifies operations on values by allowing classes to be subtypes of "interfaces".
    This is also the basis for polymorphism in Joy'.
    Similarly to Java, a Joy' interface defines a set of "methods" that each take the same target value
    as a first parameter. A child class automatically supports all interfaces supported by its ancestor
    classes (else it would not be a subtype of its ancestor classes). The child may inherent its ancestors'
    implementations or provide its own. Unrelated classes can support the same interfaces, in which case they must
    each provide their own implementations. A class can have multiple parents, and inherits
    interface implementations through all of them parents, so parent classes can act as mixins. 
    A class' parents are listed in priority order for choosing an implementation when multiple ancestors provide
    implementations of the same interface.
    
  * Each primitive Clojure datatype has a Joy' class, and can be constructed through normal Clojure functions.
  
  * Each user-defined Joy' class has one or more constructors. Each constructor takes a type (the runtime 
    representation of a type) belonging to that class as a first argument. Constructors are invoked
    by client code to create instances of the class, and also are used for generative testing.


