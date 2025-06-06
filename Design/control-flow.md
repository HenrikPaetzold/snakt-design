# Translating Kotlin control flow

We expect the translation to mostly be straightforward, but it's worth
considering the full list to ensure we're not missing anything.
One potentially difficult point is exceptions. 

In Kotlin, most control flow structures can be used both as statements
and as expressions.
In Viper this is not the case, so we will need to normalise the expression.

## `if`

As a statement, an `if` can be compiled straightforwardly.
As an expression, the simplest approach is likely to normalise this into
an assignment to a fresh variable.

One thing to watch out for is the following:

```kotlin
val y = f() + if (g()) { x } else { 0 }
```

It is not enough to extract the `if` expression; the following code is **not** equivalent,
as it reorders the calls to `f` and `g`.  It may thus be best to normalise *all* expressions,
so that a function or operator is only ever applied to variables.

```kotlin
val z;
if (g()) {
  z = x
} else {
  z = 0
}
val y = f() + z
```

## `when`

TODO: likely can be compiled to a sequence of ifs.

## `while`

While loops can be compiled directly.  The difficulty is in establishing invariants for
the loop; this needs a discussion of its own, so we skip it here.

Do-while loops are more difficult.  The simplest approach is to turn the loop into a while
loop, duplicate the loop body, and prepend it to the loop.  In other words, we can translate

```kotlin
do {
  f()
} while (g())
```

into 

```kotlin
f()
while (g()) {
  f()
}
```

(Of course, nothing is ever quite so easy: we still need to support `break` and `continue` in
these loops.)

Another difficulty is side effects in the condition of the loop.  Method calls are not permitted
in the condition: instead, we can compile

```kotlin
while (g()) {
  f()
}
```

into 

```kotlin
var b = g()
while (b) {
  f()
  b = g()
}
```

Of course, some care is required to ensure that this translation works well with `break` and `continue`.

## `for`

For loops can be converted to while loops by making the iterator explicit.
Specifically,

```kotlin
for (x in xs) {
  f(x)
}
```

can be converted into

```kotlin
var xs_it = xs.iterator()
while (xs_it.hasNext()) {
  val x = xs_it.next()
  f(x)
}
```

## `break` and `continue`

Both `break` and `continue` can be implemented in terms of `goto`.
One thing to note is that the loop invariant does not seem to be checked
when `goto` is used out of the loop.
It's not quite clear what information is available at the label.

## `return`

Similarly to `break` and `continue`, the control flow of `return` can be
implemented using a goto.  The returning of the value has to happen via
a variable.

## Exceptions

Exceptions add additional invisible control flow paths.  When an exception is
thrown, we cannot expect the postcondition of the function to hold, so unless
we are in a `try` block there is nothing for us to do.
In `try` blocks the situation is harder: pessimistically, we should require
the postcondition of the `try` block to hold between any two function calls (as
any such calls could throw).

For the first prototype, not supporting `try` blocks seems like a reasonable option.
We can model `throw` statements non-termination (or as an error).

## Note on `inline` functions

A lot of Kotlin control flow depends on `inline` functions actually being inlined.
It seems reasonable to do this when translating to Viper: this also permits a much
larger portion of the language to be handled even without lambda support.