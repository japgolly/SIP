---
layout: sip
title: SIP-xx - Export keyword

vote-status: Unsubmitted
permalink: /sips/:title.html
---

**Authors: David Barri (japgolly)**

**Supervisor and advisor: -**

## History

| Date          | Version            |
|---------------|--------------------|
| 31st Mar 2018 | Draft Commencement |


## Introduction

This SIP proposes an `export` keyword that would look identical to usage of the `import` keyword,
with the difference that instead of the local scope being modified, the enclosing code block would
be modified such that:

```scala
object $2 { export $1.{a => b, x => _, _ } }
import $2._
```

is mostly equivalent to

```scala
import $1.{a => b, x => _, _ }
```

"Mostly equivalent" instead of identical because:

* packages are not exported
* types of values might be weakened from their unique reference to their value type


## Implementation

Here is a demonstration of what can be exported, and how.
Let's say you have the following object of goodies that you'd like to `export` somewhere:

```scala
object Exportable {
  final val V = 0
  val v = 1
  lazy val lv = 2
  var x = "yuck"
  def d[A >: Null](i: Int): A = null
  object O { val x = 2 }
  class C[A <: AnyRef](a: A)
  case class CC[A <: AnyRef](a: A)
  trait T[A <: AnyRef] { val x = 3 }
  type TA[A <: AnyRef] = List[A]
  implicit class IC[A <: AnyRef](val a: A) extends AnyVal
}
```

Firstly, you can use the `export` keyword in only three scopes:
* an immediate `object` body
* an immediate `class` body
* an immediate `trait` body

```scala
object O { export Exportable._ } // OK
class C { export Exportable._ } // OK
trait T { export Exportable._ } // OK

object Otherwise {
  def d = {
    export Exportable._ // compilation error: use import instead
    ???
  }
}
```

Upon using the `export` keyword, Scala will add types and terms to its object/class/trait.
In some cases, the finality of the parent will result in different code.

If the scope is final (i.e. `object` or `final class`), then the result of `export Exportable._ ` is:

```scala
final val    Vi : Int                 = 0              // constant-fold value already (?)
final val    Vs : Exportable.Vs.type  = Exportable.Vs  // constant-folding(?) + type equality
val          vs : Exportable.vs.type  = Exportable.vs  // def prevents this.vs.type
lazy val     lzs: Exportable.lvs.type = Exportable.lvs // def prevents this.lvs.type
implicit val Ivs: Exportable.Ivs.type = Exportable.Ivs // def prevents this.vs.type
// plus the unconditional results (see below)
```

If the scope is non-final (i.e. `trait` or non-final `class`), then the result of `export Exportable._ ` is:

```scala
val          Vi : Int    = 0
val          Vs : String = Exportable.Vs
def          vs : String = Exportable.vs
def          lzs: String = Exportable.lvs
implicit def Ivs: String = Exportable.Ivs
// plus the unconditional results (see below)
```

The unconditional part of `export Exportable._ ` is:

```scala
def          vi                   : Int                = Exportable.vi // .type not available on primitives
def          lzi                  : Int                = Exportable.lvi // .type not available on primitives
def          x                    : String             = Exportable.x
def          x_=(a: String)       : Unit               = Exportable.x = a
def          d[A >: Null](i: Int) : A                  = Exportable.d[A](i)
lazy val     O                    : Exportable.O.type  = Exportable.O // def prevents v.type
type         C[A <: AnyRef]                            = Exportable.C[A]
type         CC[A <: AnyRef]                           = Exportable.CC[A]
lazy val     CC                   : Exportable.CC.type = Exportable.CC // def prevents v.type
type         T[A <: AnyRef]                            = Exportable.T[A]
type         TA[A <: AnyRef]                           = Exportable.TA[A]
type         IC[A <: AnyRef]                           = Exportable.IC[A]
def          IC[A <: AnyRef](a: A): IC[A]              = Exportable.IC[A](a)
implicit def Id[A >: Null](i: Int): A                  = Exportable.Id[A](i)
implicit def Ivi                  : Int                = Exportable.Ivi // .type not available on primitives
}
```

Additionally in non-final scopes, certain modifiers
(currently: `private`, `protected`, `final`, `override`)
can be propagated through.

Eg. using `protected`:
```scala
trait Example {
  export protected Exportable.{vs, TA}
  // results in
  protected def vs: String = Exportable.vs
  protected type TA[A <: AnyRef] = Exportable.TA[A]
}
```

Eg. using `final` uses the final-scoping rules in addition to adding `final`
```scala
trait Example {
  export final Exportable.{vs, TA}
  // results in
  final val vs: Exportable.vs.type = Exportable.vs
  final type TA[A <: AnyRef] = Exportable.TA[A]
}
```

## Formal definition
TODO


## More examples
TODO


## Implementation notes
TODO


## Questions

* Make `def`s `@inline`? Or allow `@inline export`?

* Is it important to maintain `.type` on exported vals and objects?
  Not doing so would result in more performant bytecode
  (by allowing `def`s to `val`s, `lazy val`s and objects).
  Maybe default to off and have a means of opt-in like, something like `export` vs `export val`


## Limitations / Not-in-scope / Unsupported

* Exporting packages. `export java.util` is not supported. Types and stable typed terms only.


## Conclusion
TODO


## References
TODO
