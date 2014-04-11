---
layout: reference
title: "Classes and Inheritance"
category: reference
subcategory: syntax
related:
    - title: "Functions"
    - title: "Nested Classes"
    - title: "Traits"
---

## Classes

Classes are first-class citizens in Kotlin and are declared using the keyword *class*{: .keyword }

{% highlight kotlin %}
class Invoice {

}
{% endhighlight %}

Class bodies are optional. In Kotlin if a class has no body, it can omit the curly braces


{% highlight kotlin %}
class Invoice
{% endhighlight %}


### Constructors

Classes in Kotlin can only have a single constructor, which is declared in the header and any initialization code should be enclosed in an anonymous initializer

{% highlight kotlin %}
class Customer(name: String) {

    {
        logger.info("Customer initialized with value ${name}")
    }
}
{% endhighlight %}
```

If the initialization is merely assigning values to class properties, this can be done without having to use an anonymous initializer

{% highlight kotlin %}
class Customer(name: String) {
    val customerName = name
}
{% endhighlight %}

In fact, for declaring properties and initializing them from the constructor, Kotlin has a more concise syntax

{% highlight kotlin %}
class Customer(val customerName: String) {

}
{% endhighlight %}

which is equivalent to the previous code. Much the same way as when declaring properties, those explicitly
declared in the constructor can be mutable (*var*{: keyword }) or immutable (*val*{: keyword }).

### Creating instances of classes

To create an instance of a class, we call the constructor as if it were a regular function

{% highlight kotlin %}
val = Invoice()

val = Customer("Joe Smith")
{% endhighlight %}

Note that Kotlin does not have a *new*{: .keyword }


### Class Members

Classes can contain

* [Functions]({{ site.baseurl }}/docs/reference/functions.html)
* [Properties]({{ site.baseurl }}/docs/reference/properties.html)
* [Nested and Inner Classes]({{ site.baseurl }}/docs/reference/nested-classes.html)
* [Object Declarations]({{ site.baseurl }}/docs/reference/object-declarations.html)


## Inheritance

All classes in Kotlin have a common superclass *Any*{: .keyword }, that is a default super for a class with no supertypes declared:

{% highlight kotlin %}
class Example // Implicitly inherits from Any
{% endhighlight %}

Any is not java.lang.Object; in particular, it does not have any members, not even equals(), hashCode or toString(). This does not mean that you can not call, say, toString() on any object: you can, but it will be an extension function.
See [Java interoperability]({{ site.baseurl }}/docs/reference/java-interop.html#object-methods) for more details.

To declare an explicit supertype, one puts it after a colon in the class header:

{% highlight kotlin %}
open class Base(p : Int)

class Derived(p : Int) : Base(p)
{% endhighlight %}

As you can see, the base type can (and must) be initialized right there, using the parameters of the primary constructor.

The open annotation on a class is the opposite of Java's final: it allows others to inherit from this class. By default, all classes in Kotlin are final,
which corresponds to [Item 17 of Effective Java: Design and document for inheritance or else prohibit it](http://java.sun.com/docs/books/effective).

### Overriding Members

As we mentioned before, we stick to making things explicit in Kotlin. And unlike Java, Kotlin requires explicit annotations for overridable members that we call open and for overrides:

{% highlight kotlin %}
open class Base {
  open fun v() {}
  fun nv() {}
}
class Derived() : Base() {
  override fun v() {}
}
{% endhighlight %}

The override annotation is required for *Derived.v()*. If it were missing, the compiler would complain. If there is no open annotation on a function, like *Base.nv()*, declaring a method with the same signature in a subclass is illegal, either with override or without it. In a final class (e.g. a class with no open annotation), open members are prohibited.

A member marked override is itself open, i.e. it may be overridden in subclasses. If you want to prohibit re-overriding, use final:

{% highlight kotlin %}
open class AnotherDerived() : Base() {
  final override fun v() {}
}
{% endhighlight %}

#### Wait! How will I hack my libraries now?!

One issue with our approach to overriding (classes and members final by default) is that it would be difficult to subclass something inside the libraries you use to override some method that was not intended for overriding by the library designer, and introduce some nasty hack there.

We think that this is not a disadvantage, for the following reasons:

* Best practices say that you should not allow these hacks anyway
* People successfully use other languages (C++, C#) that have similar approach
* If people really want to hack, there still are ways: in some cases it will be possible to write your hack in Java, and Aspect frameworks always work for these purposes...


### Overriding Rules

In Kotlin, implementation inheritance is regulated by the following rule: if a class inherits many implementations of the same member from its immediate superclasses, it must override *this*{: .keyword } member and provide its own implementation (perhaps, using one of the inherited ones). To denote the supertype from which the inherited implementation is taken, we use *this*{: .keyword } qualified by the supertype name in angle brackets, e.g. this<Base>:

{% highlight kotlin %}
open class A {
  open fun f() { print("A") }
  fun a() { print("a") }
}

trait B {
  fun f() { print("B") } // trait members are 'open' by default
  fun b() { print("b") }
}

class C() : A(), B {
  // The compiler requires f() to be overridden:
  override fun f() {
    super<A>.f() // call to A.f()
    super<B>.f() // call to B.f()
  }
}
{% endhighlight %}

It's fine to inherit from both *A* and *B*, and we have no problems with *a()* and *b()* since *C* inherits only one implementation of each of these functions.
But for *f()* we have two implementations
inherited by *C*, and this we have to override *f()* in *C* and provide our own implementation that eliminates the ambiguity.

## Abstract Classes

A class and some of its members may be declared abstract. An abstract member does not have an implementation in its class. Thus, when some descendant inherits an abstract member, it does not count as an implementation:

{% highlight kotlin %}
abstract class A {
  abstract fun f()
}

trait B {
  open fun f() { print("B") }
}

class C() : A(), B {
  // We are not required to override f()
}
{% endhighlight %}

Note that we do not need to annotate an abstract class open – it goes without saying. Neither need we annotate an abstract function open.

We can override a non-abstract open member with an abstract one

{% highlight kotlin %}
open class Base {
  open fun f() {}
}

abstract class Derived : Base() {
  override abstract fun f()
}
{% endhighlight %}

## Class Objects

In Kotlin, unlike Java or C#, classes do not have static methods. In most cases, namespace-level functions form a good substitute for them, but there are a few cases when they don't. These cases involve access to class' internals (private members).

For example, to replace a constructor with a Factory method, one makes the constructor private and provides a function that calls the constructor. But if this function in located outside the class in question, it would not have any access to the constructor.

To address this issue (and to provide some other interesting features), Kotlin introduces a concept of a class object (the closest analog in other languages would be Companion objects in Scala). Roughly speaking, a class object for class C is an object (in the sense of Object declaration) that is associated to C. There may be not more than one class object for each class. A class object is declared inside its associated class, and thus it can access its private members. A class object for C itself is (usually) not and instance of C. For example:

{% highlight kotlin %}
class C() {
  class object {
    fun create() = C()
  }
}

fun main() {
  val c = C.create() // C denotes the class object here
}
{% endhighlight %}

At first you may think that this is just a way of grouping static members of a class together instead of mixing them with instance members: in Java we access static members of C by calling C.foo(), and the same happens with class object's members in Kotlin. But in fact there is an important difference: a class object can have supertypes, and C, as an expression denotes this object as a value, so one can pass it around, say, as an argument for a function. Let's modify our example to demonstrate this

{% highlight kotlin %}
abstract class Factory<out T> {
  abstract fun create() : T
}

open class C() {
  class object : Factory<C>() {
    override fun create() : C = C()
  }
}

fun main() {
  val factory = C // C denotes the class object
  val c = factory.create()
}
{% endhighlight %}

Note that class objects are never inherited:

{% highlight kotlin %}
class D : C()

val d = D.create() // Error: no class object for D
{% endhighlight %}

A description of some more interesting features related to class objects can be found in the Generic constraints section.
r
Note: if you think that class objects are a great way of implementing singletons in Kotlin, please see Object expressions and Declarations.
