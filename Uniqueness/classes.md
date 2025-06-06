# Uniqueness and classes in Viper

This document describes how uniqueness annotations on classes
described [here](https://github.com/francescoo22/kt-uniqueness-system/blob/main/Kt-Uniqueness-System.pdf) are reflected
in the [encoding of classes into Viper predicates](../Design/classes-as-predicates.md).

## Immutable predicates

These predicates provide read access to properties, recognized as immutable due to the Kotlin type system.

```kt
class A(val x: Int, var y: Int)

class B(val a1: A, var a2: A)
```

```
class B
|
|--- val a1: A
|   |
|   |--- val x: Int
|   |
|   |--- var y: Int
|
|--- var a2: A
    |
    |--- val x: Int
    |
    |--- var y: Int
```

By looking at the tree structure of a class, the **immutable** predicate of that class grants read (`wildcard`)
permissions to all the paths that include only immutable (`val`) properties.

```
field x: Int
field y: Int
field a1: Ref
field a2: Ref

predicate A(this: Ref){
    acc(this.x, wildcard)
}

predicate B(this: Ref){
    acc(this.a1, wildcard) && acc(A(this.a1), wildcard)
}
```

Note that immutable predicates don't care if the properties are `unique` or `shared`.

## Mutable predicates

These predicates grant access to property that can be safely accessed thanks to the information provided by the
uniqueness annotation system.

```kt
class A(val x: Int, var y: Int)

class B(
    @Unique val a1: A,
    @Unique var a2: A,
    @Shared val a3: A,
    @Shared var a4: A
)
```

```
class B
|
|--- @Unique val a1: A
|     |
|     |--- val x: Int
|     |
|     |--- var y: Int
|
|--- @Unique var a2: A
|     |
|     |--- val x: Int
|     | 
|     |--- var y: Int
|
|--- @Shared val a3: A
|     |
|     |--- val x: Int
|     |
|     |--- var y: Int
|
|--- @Shared var a4: A
      |
      |--- val x: Int
      | 
      |--- var y: Int
```

By looking at the tree structure of a class, the **mutable** predicate of that class grants write or read permissions
(depending on whether they are defined as `val` or `var`) to all the paths that include only `unique` or primitive
properties.

```
field x: Int
field y: Int
field a1: Ref
field a2: Ref
field a3: Ref
field a4: Ref

predicate UniqueA(this: Ref){
    acc(this.y) && acc(this.x, wildcard)
}

predicate UniqueB(this: Ref){
    acc(this.a1, wildcard) && UniqueA(this.a1) &&
    acc(this.a2) && UniqueA(this.a2)
}
```
