---
layout: sip
title: SIP-xx - Export keyword

vote-status: Unsubmitted
permalink: /sips/:title.html
---

**Author: David Barri (japgolly) and Ruslan Shevchenko (rssh)**

**Supervisor and advisor: -**

## History

| Date          | Version            |
|---------------|--------------------|
| 31st Mar 2018 | Draft Commencement of export Pre-SIP |
|  7   Apr 2018 |  Merged with exported import Pre-SIP (09 September 2012	- 08 Mar 2013) |

## Introduction

This SIP proposes an `export` keyword (or `@export` annotation ) that would propagated exported value from the local template scope
to the scope of the public template members.

```scala
object $2 {
  export $1.{a => b, x => _, _}
}
import $2._
```

is mostly equivalent to

```scala
import $1.{a => b, x => _, _}
```

Note, that the  idea of adding ‘something like export’  to scala is not original, first request for export feature was published in 2008 by Jamie Webb: https://wiki.scala-lang.org/display/SYGN/Export.

## Motivation

### Make imports composable:

  An ability to organize a set of import specifications into reusable structure is useful in large-scale software development  for preventing code duplication and decreasing probability possibility  of program errors in places,  where sequence of  imports is used to specify implicit context  of  some application and this context  must be the same in all compilation units inside application.

Currently such type of errors are hard to debug and impossible to prevent in compile-time.

Example:

File A
```scala
   import com.mongodb.casbah.Imports._
   import com.novus.salat._
   import  com.mycompany.salatcontext._
   .......
```

File B
```
   import com.mongodb.casbah.Imports._
   import com.novus.salat._
   import  com.novus.salat.global._
   ….................
```

Here  in A we use custom salat context, and in B -- default context which is intended for usage in the simple cases.   This is compiled fine,  but   we receive runtime exception during the first usage of salat with totally unrelated message (about missing fields in persistent object)

The proposed feature, introduce keyword`export` (or annotation `@exported` for  import specifications TODO: chooses)  which  export content of specification to all contexts which import this scope.

I.e.  for our previous example, we can wrap usage of mongo inside our context:

File `com.mycompany.mongo`
```scala
    package com.mycompany 
    
    package object  mongo {
      export com.mongodb.casbah.Imports._
      export com.novus.salat._
      export com.mycompany.salatcontext._
    }
```
  
Then  use this in our compilation units:

 File A: 
```scala
   import com.mycompany.mongo._
   …..............
```   

File B:
```scala
   import com.mycompany.mongo._
   ….................
```


### Delegation:

* Satisfying an interface mostly by delegating.

  In this example there is a before and after.
  Using `export` results in a drop from 337 to 104 LoC, a 72% reduction.

  https://gist.github.com/japgolly/cb50c4773ce37ddaa190222bd008dbca

TODO:  describe with all examples inline.


### Other Use Cases and Examples

* Organising packages nicely while still providing nice UX for end-users
  [(example)](https://github.com/japgolly/scalajs-react/blob/v1.2.0/core/src/main/scala/japgolly/scalajs/react/package.scala#L26-L41)

* Creating traits for "everything that one would import" that can be used to create different flavours.
  [(example)](https://github.com/japgolly/scalacss/blob/master/core/shared/src/main/scala/scalacss/defaults/Exports.scala)

* Reducing boilerplate when avoiding the no-value-classes-in-traits limitation.
  [(example 1)](https://github.com/japgolly/scalajs-react/blob/v1.2.0/scalaz-7.2/src/main/scala/japgolly/scalajs/react/internal/ScalazReactExt.scala#L59-L63)
  [(example 2)](https://github.com/japgolly/scalajs-react/blob/v1.2.0/scalaz-7.2/src/main/scala/japgolly/scalajs/react/internal/ScalazReactState.scala#L36-L39)


## Informal Description

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
  trait T[-A] { val x = 3 }
  type TA[+A <: AnyRef] = List[A]
  implicit class IC[A <: AnyRef](val a: A) extends AnyVal
}
```

Firstly, maenwile usage of the `export` clause is possible only in the template entity body (i.e. inside immediate scope of `object`,`class` or `trait`).

Upon using the `export` clause, Scala will made types and terms in clause be visible as its object/class/trait.

Additionally, scope modifiers (`private`, `protected`) can be propagated through.

Eg. using `protected`:
```scala
trait Example {
  export protected Exportable.{vs, TA}
}
```
 will result that definition from Exportable will be visible as protected.

Note, that if same name is defined in template body, than definition in object shadows exported.  This allows implementation of proxies which add custom logic to some methods.

```
trait MyProxy(export p:Proxied){
 
  def methodFromP():String = {
    p.method
    System.out.println("methodFromP called")
  }
}
```

* Generation of bridge methods  (TBD) (see Questions).




## Formal definition

Syntax:  
-----------------------------------------------
(Variant 1):
```
Export ::= ‘export’ ImportExpr {‘,’ ImportExpr}
```
---------------------------------------------
(Variant2:) - annotation as in [1]
 Annotations  must be allowed  inside template declarations before import clauses.  I. e, grammar rules must be changed from:
```
TemplateStat ::=   Import
                             | {Annotation} {Modifier} Def
                             | {Annotation} {Modifier} Dcl
                             | Expr
``` 
to
```
TemplateStat ::=    {Annotation}  Import
                             | {Annotation} {Modifier} Def
                             | {Annotation} {Modifier} Dcl
                             | Expr
```

and introduce static annotation:  scala.annotation.exported

---------------------------------------------End Variant 2

Also 
`export val identifier=Expr`  is a shortcut for

```
 val identifier=Expr
 export identifier._
```

Semantics:

   When some class or object has export in scope A and one is imported by wildcard to scope B, than names, exported in A become  visible in  B

I.e. resolving of name or search for  must include next steps:
* search in  local scope
* search in names, imported by import declaration.
** if  import declaration contains import by wildcard, and object or package object contains exported import declaration, then search continues in scope imported by this  declaration (if  one was not previously imported).
** Cycles in exported import declarations are allowed
* search in exports declared in  parents of current object. 
* search in enclosing scope

   This must be applicable for the names, implicit conversions and  specifying used language features via language object as specified in SIP-18.  

   Search for object member must include next step: 

* search in scope of this object.
* search in the scope of the object exports.

Q:  export vs inheritance priority (?)


------------
For variant with the bridge methods for delegated enabled: TBD.


## Implementation notes

Public API

   Exported  imports must be the part of scala type definition and serialized in scala pickled signature and be available in reflection.   This is necessary for tools support, 


Scaladoc

   Existence of   exported symbols  must be reflected in documentation, generated by  scaladoc:  if  some template contains exported imports, than generated documentation must contains section ‘exported imports’  with list of all exported imports  (including indirect)  and links to documentation of appropriative objects.

  Example of  generated scaladoc for simple example  can be found at https://github.com/rssh/scala-annotated-import-example




TODO


## Questions

* Introduce new keyword `export` or add anotation `@export` to imports and val-s ? 

* Are we want to generate bridge methods for implementation.

I.e. let we have next structure:
```
trait MyInterface {
  def myMethod
}

class MyImpl extends MyInterface

class WrappedMyImpl(v: MyImpl) extends MyInterface {
  export v._
}

```

Question - if myMethod is not defined in `b`, should bridge method in WrappedMyImpl be generated ?

* export vs inheritance: (conflict). 

## Limitations / Not-in-scope / Unsupported



## Conclusion
TODO


## References

TODO

[1] annotated import proposal [ https://docs.google.com/document/d/1dlT6NgB9610jqLscCJW2LRB7TapDh3q4d2S3YA_q5zs/edit?usp=sharing ]
