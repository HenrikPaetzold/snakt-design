# Translating Kotlin objects to Viper

Consider a Kotlin function such as `indexOf` below.  Translating such a function involves
choosing some representation for arrays of `T`, for elements of `T` itself, and for integers.

```kotlin
fun Array<T>.indexOf(element: T): Int {
    if (element == null) {
        for (index in indices) {
            if (this[index] == null) {
                return index
            }
        }
    } else {
        for (index in indices) {
            if (element == this[index]) {
                return index
            }
        }
    }
    return -1
}
```

There's a few things to note here:
- `T` may be a nullable type.
- `Array<T>` has an associated iterator type and corresponding functions.

There seem to be a number of levels of information that we can have about a reference:
1. The reference nominally belongs to a particular class, and hence has a certain place
   in a type hierarchy.  This is opaque: it doesn't say anything at the Viper level,
   but may be useful for e.g. validating smart casts.
2. The reference belongs to a particular class and hence methods that take a reference
   will have particular behaviour for it.  (Another option is for those methods to take
   the information as provided by points 1 or 3; this may be more compact, really, but it's
   not clear it can be used to express all properties (e.g. injectivity, relation between
   different methods)).
3. The reference belongs to a particular class and has associated fields that we can
   gain access to (read-only or read-write).

## Existing approaches

There are a number of prior compilers that map industry-level programming languages to Viper.
These use a variety of different approaches for representing objects.

### Gobra (Go → Viper)

- Gobra models structs as references and primitive as both as primitives and
  references. The reason primitives cannot always be modeled as primitives is
  because Go lets you create references to any values. In this case the
  primitive value is placed on the heap and modeled in Viper with a reference
  and single field containing the value. Unfortunately, Gobra doesn't decide
  this automatically. An `@` annotation has to be placed on shared variables
  that should be placed on the heap.
- Since access permissions have to be specified explicitly anyway, the type
  system is in some sense ignored by the verifier: you can't have type-incorrect
  programs because they will be rejected, but the types do not give much extra
  information about what you _can_ do.
- To the best of my knowledge Gobra does not support lambdas.


- (How) does it deal with constantness?
  - Constantness does not need to be handled explicitly as users provide
    permission annotations themselves.
  
- (How) does it deal with inheritance?
  - Go does not support classes, only structs. As such, there is also no concept
    of inheritance.
  - However, Go does support interfaces. Gobra models interfaces abstract
    predicate families (APFs, see section 4.1.4 of [this doctoral thesis][3] for
    more detail). In addition, an implementation proof is required that the
    methods of a subtype of the interface adhere to the specification of the
    interface. In simple cases Gobra can automatically infer these
    implementation proofs, in other cases the user has to provide them.
  - This is important for us as well, not just for interfaces but also for
    subtyping. As an illustration let A and B be two classes where A is a
    subtype of B. B now contains a method f with an accompanying specification.
    If A overrides this method we have to prove that the specification of the
    method in A entails the specification in B. In detail this means that the
    preconditions of A.f imply the precondition of B.f and that the
    postcondition of B.f imply the postcondition of A.f. This is to make sure
    that a parameter of type B can always be soundly be replaced by a value of
    type A.
  - It is not clear to me if we can simplify things a bit as Kotlin has nominal
    subtyping in comparison to Go's structural subtyping.

- (How) does it deal with purity?
  - Pure functions can either be defined just as comments (ignored by the
    compiler and solely for the purpose of verification) or a pure annotation
    can be attached to the actual Go function. In the latter case only a subset
    of pure Go expressions is allowed.
  - These pure functions are encoded as Viper functions. The reason for
    annotating a function as pure is that it can be used for the purpose of
    specification.

- Gobra provides first-class predicates.  How do they work?  What are they for?
  Do we need them?
  - Predicates are similar to pure function with the difference that they can
    define (potentially recursive) assertions. This is very useful when making
    assertions on (recursive) data structure.
  - To give users the power to make these powerful predicate definition
    themselves for use in their specification Gobra exposes them to the user (as
    do all other Viper frontends)

[3]: https://essay.utwente.nl/81338/1/Rubbens_MA_EEMCS.pdf

### Prusti (Rust → Viper), heap encoding

A more elaborate description can be found [here][0].

The heap-based encoding places all Rust values in the heap.  For Rust this makes sense,
since Rust allows the programmer to make a reference to any variable or member, while
Viper only allows references to the heap.

Primitive types are encoded as a heap value with a single field that represents the underlying
value (`val_int` for `i*` and `u*` types, `val_bool` for `bool`, `val_ref` for references, etc).
Composite types are represented as heap values with fields for their components.
Each type is additionally encoded using a predicate that specifies the permissions one
has to the fields, as well as the legal values these fields can take.
These predicates are built inductively from the predicates for the members.
Enumeration types are encoded using an explicit discriminator.

The primary problem with this encoding seems to be the limited possibilities of quantifying
over Rust values.  Any quantification has to happen over references and then be constrained
by predicates, which is inconvenient.  Viper functions also cannot contain postconditions
about the heap, and so it is not possible to return objects encoded this way from such a function.

**Question:** How does Prusti deal with immutability here?  A reference `p: &i32` can be
encoded as a fractionally-owned predicate, but what does Prusti do to inform the SMT solver
that calling `foo(p: &i32)` as `foo(&x)` does not change the value of `x`?

**Answer:** This is built into the permission logic of Viper. As long as the
caller keeps a non-zero fraction of the predicate, it can be automatically
inferred that the callee (and also no other running thread for that matter) can
possibly have write permission to the reference. Thus the value can be _framed_
around the method call, i.e. the value has not been changed. See the last
exercise of the [Fractional Permission Section][1] of the Viper tutorial for an
illustration of how this works.

[0]: https://viperproject.github.io/prusti-dev/dev-guide/encoding/types-heap.html
[1]: http://viper.ethz.ch/tutorial/#non-aliasing

### Prusti (Rust → Viper), snapshot encoding

A more elaborate description can be found [here][1].

The snapshot-based encoding uses domains to create an abstract representation of Rust values
(called "snapshots") that it then relates to heap values, allowing one to construct a snapshot
value from a heap value.
The snapshots are axiomatised to behave the same way as the Rust types do by imposing injectivity
and surjectivity constraints.

A significant advantage of this approach is that snapshot values can be compared using ==
equality in Viper.

Note that this encoding is specifically of _values_, not types.  There is no Viper representation
of the type system in Rust (in contrast to Python below).

[1]: https://viperproject.github.io/prusti-dev/dev-guide/encoding/types-snap.html

### Nagini (Python → Viper)

A more elaborate description can be found in [the PhD thesis of Marco Eilers][3], or in
[the paper that chapter is based on][2].

Nagini encodes the type hierarchy of Python in Viper via an axiomatisation.
Noteworthy here is that the type relations (e.g. `issubtype`) are encoded as functions,
rather than predicates, which means that they are not constrained by linearity.
These functions are opaque, with their behaviour defined using a set of axioms.

The encoding of fields is complicated and dependent on the runtime behaviour of the program.
Given Kotlin's statically typed nature, this part of the encoding does not seem to be
particularly useful to us.

[2]: https://link.springer.com/chapter/10.1007/978-3-319-96145-3_33
[3]: https://pm.inf.ethz.ch/publications/Eilers2022.pdf

### VerCors (Viper extension)

[The documentation][4] contains some further information, thuogh there doesn't seem to be
a single description of the translation.

The type system seems to be an extension of Viper's, with a number of tools for modelling
classes.  However, permissions still have to be specified explicitly.  In this sense there
is not that much new here compared to Viper itself.

[4]: https://github.com/utwente-fmt/vercors/wiki

## Feasible approaches for Kotlin

Let us outline the possibilities for translating the object-oriented part of Kotlin
into Viper.

### Assume accessibility, assume disjointness

The most straightforward translation maps primitive types to primitive types and class types
to references, with every value accompanied with a predicate that provides access permissions.

The above function signature is then translated as follows (ignoring polymorphism for the moment):

```
// Kotlin
fun Array<T>.indexOf(element: T): Int

// Viper
method Array_indexOf(this: Ref, element: Ref): Int
    requires Array_pred(this)
    ensures Array_pred(this)
```

This approach assumes that functions are never passed aliasing data; see the `aliasing_problem`
method in `Examples/Viper/Kotlin/aliasing.vpr` for an example of when Viper can validate a
postcondition that does not hold if this assumption is violated.
Since Kotlin does not impose any restrictions on aliasing, this approach is fundamentally unsound.
However, given we are making a tool to provide warnings to the user it may be that we can accept this.

The class predicates can be built inductively much the same way they are in the heap-based encoding
of Prusti.

A problem with this approach is that calling a polymorphic function with a primitive as a type parameter
requires some kind of trick, since the implementation is specified as taking a `Ref`.

More broadly, this overapproximates what kind of things we need to model as references.  Immutable types
like `Result` can probably be treated as values, and this may make our life easier.

### Inhale accessibility on read

An alternative approach that permits for aliasing, but can verify much less, is to inhale and exhale
permissions of possibly aliasing locations on an as-needed basis.  This is demonstrated in method
`aliasing_inhale_exhale` in `Examples/Viper/Kotlin/aliasing.vpr`, where by exhaling the permissions
to `x.f` and `y.f` whenever we are not using them we allow for the possibility for these to alias.
However, note that we then also allow for `x.f` and `y.f` to be modified by any other thread:
`aliasing_inhale_exhale_problem` shows that we cannot show even quite simple properties in this context.

### Abstract representation for value types

The above two approaches translate all Kotlin classes into Viper references (see `class-hierarchies.md`
for a more detailed description).  This encoding is not the easiest to work with: we need access
permissions for all members that we want to read or write, and we cannot permit aliasing.

Instead, we can choose to model classes that behave like values using Viper domains.
This involves a number of conditions:
1. All fields of the class must be immutable.
2. The class must be final.
3. The class must not extend any other class.

**Note**: There's still work to be done to determine that these conditions are sufficient.
Some other things that may be worth asking:
1. May the class implement interfaces?
2. Are there language features (pointer comparison?) that can cause us problems?

For example, consider the following Kotlin code.

```kotlin
class A(val x: Int, val y: Int)

fun sum(a: A): Int = a.x + a.y
```

We can translate this into Viper as follows.

```
domain A {
    function A_new(x: Int, y: Int): A
    function A_get_x(a: A): Int
    function A_get_y(a: A): Int

    axiom ax_A_get_x {
        forall x: Int, y: Int :: A_get_x(A_new(x, y)) == x
    }
    axiom ax_A_get_y {
        forall x: Int, y: Int :: A_get_y(A_new(x, y)) == y
    }
}

function sum(a: A): Int
{
    A_get_x(a) + A_get_y(a)
}
```

It may be necessary to add some kind of axiom to be able to conclude that `get_x` and `get_y` together
determine the instance they are applied to.

Note that objects of such classes can still be cast to something of type `Any`.  It isn't quite clear
how we should model that.  A similar issue arises when passing such values to polymorphic functions.

This approach is similar to Prusti's snapshot encoding, but we do not build a heap structure that corresponds
to the snapshot.  We expect this not to be necessary, since such classes are by construction immutable,
and so we can always treat them as values.  (The classes may contain references, but these are still
immutable, though they may refer to mutable data structures.)

## Corner cases

Let's suppose that we take a mixed approach, where some Kotlin types are represented by `Ref` and others
by a primitive or a domain.

### Representing `Any?`

Any Kotlin value can be cast to the `Any?` type, which we can represent as `Ref` in Viper.
For types that are already represented by a reference, this is straightforward: however, for primitives
and types represented by domains, it is not as clear how this translation should be done.

One promising possibility is to wrap the value as a reference with a field containing the wrapped value.  

Kotlin allows runtime type checks, so after a cast to `Any?` the original type is still present at
runtime to validate casts and resolve type comparisons.  Any wrapper must thus also include a tag with
the runtime representation of this type; likely the same approaches as in `class-hierarchies.md` should
be reasonable (note: still TODO).

When casting from `Any?` to a different type, the nullness and tag are checked to ensure the conversion
is permitted.  If the object represents a wrapped value, that value is extracted.  For a class type
represented using a reference, the extra permissions are inhaled (possibly via a helper method).