# Design Notes on the OnlyInputTypes annotation

* **Type**: Design notes
* **Author**: Vitaly Bragilevsky
* **Contributors**: 
* **Status**: Under consideration
* **Discussion and feedback**: [KEEP-???](https://github.com/Kotlin/KEEP/issues/???)

`OnlyInputTypes` is an internal annotation that tweaks the behavior of Kotlin's type inference algorithm. 
It is used in the standard library implementation (e.g. for [`Map::contains`](https://github.com/JetBrains/kotlin/libraries/stdlib/src/kotlin/collections/Maps.kt#L238))
and several other projects to report type errors at earlier stages. 
`OnlyInputTypes` is applied at a generic type parameter declaration site and affects what types are allowed 
to be instantiated at a use site.

Over the years, we've got many requests to make `OnlyInputTypes` public (e.g. [this one](https://youtrack.jetbrains.com/issue/KT-13198) back from 2016).
The goal of this note is to describe the current behaviour of the `OnlyInputTypes` annotation and discuss potential use-cases and
caveats of making it public.

## Motivation

Let's look at the following example that doesn't use `OnlyInputTypes`:

```kotlin
class A1
class A2

private fun <T> assertEquals1(expected: T, actual: T): Boolean = ...

fun test1() {
    assertEquals1(A1(), A2())
    assertEquals1("answer", 42)
}
```

Both calls to `assertEquals1` are compiled without errors. In both cases, the type inference engine infers
some common type of the arguments (`Any` in the  first call, an intersection of `Comparable` and `Serializable`
in the second one) and instantiates the `T` type parameter to it.

In this particular example, we may consider such a behavior undesirable. Using values of different types in equality check
may be an evidence of a programmer's error. Clearly, arguments are not equal. In this situation, we could prefer to 
report a type error. Note that for a similarity check this behavior could 
be absolutely fine, so it's not necessarily an error from a type inference perspective.

To solve the problem, Kotlin provides the `kotlin.internal.OnlyInputTypes` annotation:
```kotlin
private fun <@kotlin.internal.OnlyInputTypes T> assertEquals1(expected: T, actual: T): Boolean = ...
```

Both calls result in a type error now because the common type of the arguments (which is the same as above)
*is not found among the input types* (`A1` and `A2` for the first call, `String` and `Int` for the second one).

Note that even in the presence of `OnlyInputTypes` we could make both calls correct by giving more context
about our intentions to the compiler:
```kotlin
    assertEquals1<Any>(A1(), A2())
    assertEquals1<Serializable>("answer", 42)
```

## Current behavior

Call sites in the presence of generic type parameters are type checked in the following order:

1. The compiler collects all the type constraints regarding type variables at the call site. Types mentioned in those constraints 
    form the *input types set*.
2. Type inference engine infers the most specific common type for all the occurrences of a type parameter (least upper bound).
3. If the type parameter is annotated with `OnlyInputTypes`, the compiler verifies that the type inferred 
    at stage 2 is one of the *input types set* and reports an error if it is not.

It's important to understand, that `@OnlyInputTypes` while putting additional constraints on the instantiated generic types 
doesn't affect type inference itself. Stage 2 in the algorithm above doesn't depend on `@OnlyInputTypes`.

## Usage in Kotlin libraries

### Assertions

```kotlin
fun <@OnlyInputTypes T> assertEquals(expected: T, actual: T, message: String? = null)
```
and variations: `assertSame`, `assertContains`

### Collections
```kotlin
fun <@OnlyInputTypes T> Iterable<T>.contains(element: T): Boolean
```
and variations: `indexOf`, `lastIndexOf`

```kotlin
fun <@OnlyInputTypes K, V> Map<out K, V>.get(key: K): V?
```
and variations: `containsKey`

```kotlin
fun <@OnlyInputTypes T> MutableCollection<out T>.remove(element: T): Boolean
```
and variations: `removeAll`, `retainAll`


## Potential use cases

TBD