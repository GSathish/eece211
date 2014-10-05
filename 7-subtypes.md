---
layout: page
title: Subtypes
---

#### Objectives of this Reading

+ Subclassing &amp; inheritance: superclass inheritance is the source of several problems that we'll examine in this reading.
+ Composition &amp; delegation: a commonly-used pattern to avoid the problems of subclassing.
+ Subtyping: we'll look at the relationship between subtypes and subclasses.
  Careful thinking about the specifications of our types will guide our use of subtyping.

## Subtyping

Recall that a *type* is a set of values --- a (potentially unbounded) set of possible primitive values or objects.
If we think about all possible `List` values, some are `ArrayList`s and others are `LinkedList`s.
Or if we think about all possible `ImList` values, some are type `Empty` and the rest are type `Cons`.
So a *subtype* is simply a subset of the *supertype*.

**"B is a subtype of A" means "every B object is also an A object."**

In terms of specifications, "every B object satisfies the specification for A."

Subclassing, written with the *extends* keyword `class B `**`extends`**` A`, is one way to declare a subtyping relationship in Java. It is not the only way: other ways include implementing an `interface` (`class B implements A`) or one interface extending another (`interface B extends A`).

## Subclassing

Whenever we declare that B is a subtype of A, the Java type system allows us to use a B object whenever an A object is expected.

That is, when the [declared type][declared] of a variable or method parameter is A, the actual [runtime type][declared] can be B.

## Subclassing

In *Effective Java* by Joshua Bloch, *Item 16: Favor composition over inheritance*:

> [Subclassing] is a powerful way to achieve code reuse, but it is not always the best tool for the job.
> Used inappropriately, it leads to fragile software.

Within one module, where the subclass and superclass implementations are under the control of the same programmer and maintained and evolved in tandem, subclassing may be appropriate.
But subclassing in general is not safe, and here's why:

**Subclassing breaks encapsulation.**

Bloch again:

> In other words, a subclass depends on the implementation details of its superclass for its proper function.
> The superclass's implementation may change from release to release, and if it does, the subclass may break, even though its code has not been touched.
> As a consequence, the subclass must evolve in tandem with its superclass, unless the superclass's authors have designed and documented it specifically for the purpose of being extended.

Let's look at several examples to see what can go wrong with careless subclassing.

### Example from the Java library: java.util.Properties

```java
public class Properties extends Hashtable {

    // Hashtable is an old library class that implements
    // Map&lt;Object,Object>, so Properties inherits methods like:
    public Object get(Object key) { ... }
    public void put(Object key, Object value) { ... }

    // rep invariant of Properties:
    //   all keys and values are Strings
    // so provide new get/set methods to clients:
    public String getProperty(String key) {
        return (String) this.get(key);
    }
    public void setProperty(String key, String value) {
        this.put(key, value);
    }
}
```

The [`Properties`](java:java/util/Properties) class represents a collection of `String` key/value pairs.
It's a very old class in the Java library: it predates generics, which allow you to write `Map&lt;String,String>` and have the compiler check that all keys and values in the `Map` are `String`s.
It even predates `Map`.

But at the time, the implementor of `Properties` did have access to `Hashtable`, which in modern terms is a `Map&lt;Object,Object>`.

So `Properties` extends `Hashtable`, and provides the `getProperty(String)` and `setProperty(String, String)` methods shown above.
What could go wrong?

**Inherited superclass methods can break the subclass's rep invariant.**

### Example: CountingList

Let's suppose we have a program that uses an `ArrayList`.
To tune the performance of our program, we'd like to query the `ArrayList` as to how many elements have been added since it was created.
This is not to be confused with its current size, which goes down when an element is removed.

To provide this functionality, we write an `ArrayList` variant called `CountingList` that keeps a count of the number of element insertions and provides an observer method for this count.
The `ArrayList` class contains two methods capable of adding elements, `add` and `addAll`, so we override both of those methods:

```java
public class CountingList&lt;E> extends ArrayList&lt;E> {

    // total number of elements ever added
    private int elementsAdded = 0;
    
    @Override public boolean add(E elt) {
        elementsAdded++; 
        return super.add(elt);
    }
    
    @Override public boolean addAll(Collection c) {
        elementsAdded += c.size();
        return super.addAll(c);
    }
}
```

What if `ArrayList.addAll` works by calling `add` *n* times? 

What if `ArrayList.addAll` sometimes calls `add` *n* times, and sometimes does it a different way, depending on the type of the input collection `c`?

**When a subclass overrides superclass methods, it may depend on how the superclass uses its own methods.**

### Example: photo organizer

Here's version 1.0 of a class to store photo albums:

```java
public class Album {
    protected Set&lt;Photo> photos;
    public void addNewPhoto(Photo photo) { photos.add(photo); }  
}
```

The *protected* field `photos` is accessible to subclasses.

Let's create our own subclass of `Album` that allows photos to be removed:

```java
public class MyAlbum extends Album {
    public void removePhoto(Photo photo) { photos.remove(photo); }  
}
```

Now version 2.0 of the photo organizer comes out, with a new feature for keeping track of the people in our photos.
It has a new version of the `Album` class:

```java
public class Album { 
    protected Set&lt;Photo> photos;
    protected Map&lt;Person, Photo> photosContaining;
    // rep invariant: all Photos in the photosContaining map
    //                are also in the photos set
    // ...
}
```

The `MyAlbum` subclass breaks this new representation invariant.

**When a class is subclassed, either it must freeze its implementation forever, or all its subclasses must evolve with its implementation.**

## Subtyping vs. Subclassing

### Substitution principle

**Subtypes must be *substitutable* for their supertypes.**

In particular, a subtype must fulfill the same contract as its supertype, so that clients designed to work with the supertype can use the subtype objects safely.

The subtype must not surprise clients by failing to meet the guarantees made by the supertype specification (postconditions), and the subtype must not surprise clients by making stronger demands of them than the supertype does (preconditions).

**B is only a subtype of A if B's specification is at least as strong as A's specification.**

The Java compiler guarantees part of this requirement automatically: for example, it ensures that every method in A appears in B, with a compatible type signature.
Class B cannot implement interface A without implementing all of the methods in the interface.
And class B cannot extend class A and then override some method to return a different type or throw new checked exceptions.

But Java does not check every aspect of the specification: preconditions and postconditions we've written in the spec are not checked!

If you declare a subtype to Java --- e.g. by declaring class B to extend class A --- then you should make it substitutable.

### Violating the substitution principle: mutability

Here's an example of failing to provide substitutability:

```java
/** Represents an immutable rational number. */
public class Rational {
    public Rational plus(Rational that) { ... }
    public Rational minus(Rational that) { ... }
}
```

Now let's create a mutable version as a subclass:

```java
public class MutableRational extends Rational {
    public void addTo(Rational that) { ... }
    public void subtractFrom(Rational that) { ... }
}
```

By making it a subclass, we've declared to Java that MutableRational is a subtype of Rational... but is MutableRational truly a subtype of Rational?

**Clients that depend on the immutability of Rational may fail when given MutableRational values.**
For example, an immutable expression tree that contains Rational objects --- suddenly it's mutable.
A function that memoizes previously-computed values in a HashMap --- suddenly those values are wrong.
Multithreaded code that uses the same Rational values in different threads, as we'll see in a future class, is also in trouble.

**MutableRational fails to meet guarantees made by Rational.**
Specifically, the spec of Rational says that the value of objects will never change (immutability).
The spec of MutableRational is *not* at least as strong as that of Rational.

In general, **mutable counterparts of immutable classes should not be declared as subtypes.**
If you want a mutable rational class (perhaps for performance reasons), then it should not be a subtype of Rational.
[String](java:java/lang/String) and [StringBuilder](java:java/util/StringBuilder) (and [StringBuffer](java:java/util/StringBuffer), which is safe for multiple threads) offer an example of how to do it right.
The mutable types are not subtypes.
Instead, they provide operations to create a mutable StringBuilder/Buffer from an immutable String, mutate the text, and then retrieve a new String.

### Violating the substitution principle: adding values

Another example, starting with a class for positive natural numbers:

```java
/** Represents an immutable natural number &geq; 0. */
public class BigNat {
    public BigNat plus(BigNat that) { ... }
    public BigNat minus(BigNat that) { ... }
}
```

Now we need to write a program that deals with large integers, but both positive and negative:

```java
/** Represents an immutable integer. */
public class BigInt extends BigNat {
    private boolean isNegative;
}
```

BigInt just adds a sign big to BigNat.
Makes sense, right?
But is BigInt substitutable for BigNat?

**Abstractly, it doesn't make any sense.**
We need to be able to say "every BigInt is a BigNat," but not every integer is a positive natural!
The abstract type of BigInt is not a subset of the abstract type of BigNat.
It's nonsense to declare BigInt a subtype of BigNat.

**Practically, it's risky.**
A function declared to take a BigNat parameter has an implicit precondition that the parameter is &geq; 0, since that's part of the spec of BigNat.
For example, we might declare

```java
    public double squareRoot(BigNat n);
```

but now it can be passed a BigInt that represents a negative number.
What will happen?

**BigInt fails to make guarantees made by BigNat.**
Specifically, that the value is not negative.
Its spec is *not* at least as strong.

### Violating the substitution principle: specifications

Here's a subclass that *is* a proper subtype: immutable square is a subtype of immutable rectangle:

```java
public class ImRectangle {
    public ImRectangle(int w, int h) { ... }
    public int getWidth() { ... }
    public int getHeight() { ... }
}

public class Square extends ImRectangle {
    public ImSquare(int side) { super(side, side); }
}
```

But what about mutable square and mutable rectangle?
Perhaps MutableRectangle has a method to set the size:

```java
public class MutableRectangle {
    // ...
    /** Sets this rectangle's dimensions to w x h. */
    public void setSize(int w, int h) { ... }
}

public class MutableSquare extends MutableRectangle {
    // ...
```

Let's consider our options for overriding `setSize` in MutableSquare:

<ul>
<li>
```java
/** Sets all edges to given size.
 *  Requires w = h. */
public void setSize(int w, int h) { ... }
```
**No.**
This stronger precondition violates the contract defined by Mutable&shy;Rectangle in the spec of `setSize`.
</li>
<li>
```java
/** Sets all edges to given size.
 *  Throws BadSizeException if w != h. */
void setSize(int w, int h) throws BadSizeException { ... }
```
**No.**
This weaker postcondition also violates the contract.
</li>
<li>
```java
/** Sets all edges to given size. */
void setSize(int side);
```
**No.**
This *overloads* `setSize`, it doesn't override it.
Clients can still break the rep invariant by calling the inherited 2-argument `setSize` method.
</li>
</ul>

### Declared subtypes must truly be subtypes

Design advice: when you declare to Java that "B is a subtype of A," ensure that B actually satisfies A's contract.

+ B should guarantee all the properties that A does, such as immutability.
+ B's methods should have the same or weaker preconditions and the same or stronger postconditions as those required by A.

This advice applies whether the declaration was made using subclassing (`class B extends A`) or interface implementation (`class B implements A`) or interface extension (`interface B extends A`).

Bloch's advice in *Item 16*:

> If you are tempted to have a class B extend a class A, ask yourself the question: **"is every B really an A?"**
> If you cannot answer yes to this question, B should not extend A.
> If the answer is no, it is often the case that B should contain a private instance of A and expose a smaller and simpler API: A is not an essential part of B, merely a detail of its implementation.

Even if the answer to this question is yes, you should carefully consider the use of `extends`, because --- as we saw in the example of `CountingList` --- the implementation of the subclass may not work due to unspecified behavior of the superclass.
In that example, the subclass's methods broke because the superclass's methods have an implicit dependence between them which is not in the superclass specification.
Before using `extends`, you should be able to convince yourself that dependences amongst the superclass methods will not impact subclass behavior.

## Use composition rather than subclassing

Here's Bloch's recommendation from *Item 16*:

> Instead of extending an existing class, give your new class a private field that references an instance of the existing class.
> This design is called **composition** because the existing class becomes a component of the new one.
> Each instance method in the new class invokes the corresponding method on the contained instance of the existing class and returns the results.
> This is known as **forwarding**, [(or delegation)].
> The resulting class will be rock solid, with no dependencies on the implementation details of the existing class.

The abstraction barrier between the two classes is preserved.
**Favor composition over subclassing.**

Let's apply this approach to the `Properties` class:

<pre class="no-markdown">
<code class="java"><strike>public class Properties extends Hashtable { ... }</strike>
public class Properties {
    private final Hashtable table;
    // ...
}
</code></pre>

And to `CountingList`:

<pre class="no-markdown">
<code class="java"><strike>public class CountingList&lt;E> extends ArrayList&lt;E> { ... }</strike>
public class CountingList&lt;E> implements List&lt;E> { 
    private List&lt;E> list;
    public CountingList&lt;E>(List&lt;E> list) { this.list = list; }
    // ...
}
</code></pre>

`CountingList` is an instance of the **wrapper pattern**:

+ A wrapper modifies the behavior of another class without subclassing.
+ It also decouples the wrapper from the specific class it wraps --- `CountingList` could wrap an `ArrayList`, `LinkedList`, even another `CountingList`.

A wrapper works by taking an existing instance of the type whose behavior we wish to modify, then implementing the contract of that type by forwarding method calls to the provided instance, delegating the work.

So in `CountingList` we might see:

```java
public class CountingList&lt;E> implements List&lt;E> { 
    private List&lt;E> list;
    private int elementsAdded = 0;
    
    public CountingList&lt;E>(List&lt;E> list) { this.list = list; }
    
    public boolean add(E elt) {
        elementsAdded++;
        return list.add(elt);
    }
    public boolean addAll(Collection c) {
        elementsAdded += c.size();
        return list.addAll(c);
    }
    // ...
}
```

**When subclassing is necessary, design for it.**

+ Define a *protected* API for your subclasses, in the same way you define a *public* API for clients.
+ Document for subclass maintainers how you use your own methods (e.g. does `addAll()` call `add()`?).
+ Don't expose your rep to your subclasses, or depend on subclass implementations to maintain your rep invariant.
  Keep the rep *private*.

You can find more discussion of how to design for subclassing in *Effective Java* under *Item 17: Design and document for inheritance or else prohibit it*.

## Subclassing and equality

We've already seen several ways that subclassing breaks encapsulation:

+ Inherited superclass methods can break the subclass's rep invariant.
+ When a subclass overrides superclass methods, it may depend on how the superclass uses its own methods.
+ When a class is subclassed, either it must freeze its implementation forever, or all its subclasses must evolve with its implementation.

Subclassing also causes problem for the definition of equality.

Let's revisit [the `Duration` example][duration].
Here's the code with equality and the Object contract implemented:

```java
public class Duration {
    private final int mins;
    private final int secs;
    // rep invariant:
    //    mins >= 0, secs >= 0
    // abstraction function:
    //    represents a span of time of mins minutes and secs seconds

    /** Make a duration lasting for m minutes and s seconds.
     *  Requires m >= 0, s >= 0. */
    public Duration(int m, int s) {
        mins = m; secs = s;
    }
    /** @return length of this duration in seconds */
    public long getLength() {
        return mins*60 + secs;
    }

    @Override
    public boolean equals(Object that) {
        if ( ! (that instanceof Duration)) { return false; }
        Duration thatDuration = (Duration)that;
        return this.getLength() == thatDuration.getLength();
    }

    @Override
    public int hashCode() {
        return (int)getLength();
    }
}
```

Suppose we subclass `Duration` in order to represent durations with millisecond precision:

```java
public class PreciseDuration extends Duration {
    private final int millisecs;
    // rep invariant:
    //    millisecs >= 0
    
    /** Make a duration lasting for m min, s sec, and ms millesec.
     *  Requires m, s, ms >= 0. */
    public PreciseDuration(int m, int s, int ms) {
        super(m, s); 
        millisecs = ms;
    }
    
    /** @return length of this duration in milliseconds */
    public long getMillisecs() {
        return super.getLength() * 1000 + millisecs;
    }
}
```

First, let's review the problems we've already encountered and see if they apply:

+ `PreciseDuration` cannot break `Duration`'s rep invariant, since the rep is private and immutable.
  (Although notice that `Duration` does not defend itself with a call to `checkRep` in the constructor!)
+ We are not overriding any superclass methods, which might cause our subclass to depend on the superclass method implementations.
+ We may have a problem in the future if `Duration` evolves, so this code is less ready for change.
  But for now, it seems to work.

Our new problem: how should we define equality for `PreciseDuration`?

**Can we simply use the `equals()` inherited from our superclass?**

**No**, because it ignores the additional information in our subclass: milliseconds.
A `PreciseDuration` of 0:30.0000 is equal to 0:30.8123.

**Can we simply override `equals()` in the usual way?**
For example:

```java
public class PreciseDuration {
    // ...
    @Override public boolean equals(Object that) {
        if ( ! (that instanceof PreciseDuration)) { return false; }
        PreciseDuration thatDuration = (PreciseDuration)that;
        return this.getMillisecs() == thatDuration.getMillisecs();
    }
}
```

**No**, because it ignores the information from our superclass: minutes and seconds.
A very short `PreciseDuration` of 0:00.8123 is equal to the half-hour-long 0:29.8123.

The implementation is even more deeply flawed, which we'll see if we try to fix the bug...

**Can we override `equals()` and also delegate to our superclass?**
For example:

```java
public class PreciseDuration {
    // ...
    @Override public boolean equals(Object that) {
        if ( ! (that instanceof PreciseDuration)) { return false; }
        if ( ! super.equals(that)) { return false; }
        PreciseDuration thatDuration = (PreciseDuration)that;
        return this.getMillisecs() == thatDuration.getMillisecs();
    }
}
```

**No**, because it is not [symmetric]!
Consider this example:

```java
Duration approx = new Duration(5, 30);
Duration precise = new PreciseDuration(5, 30, 2123);

/* 1 */ approx.equals(precise) &rarr; ???
/* 2 */ precise.equals(approx) &rarr; ???
```

In (1) the `approx` instance has runtime type `Duration`, so we run `Duration.equals`, which *does not consider milliseconds*.

But in (2), the `precise` instance has runtime type `PreciseDuration`, so we run `PreciseDuration.equals`, which *does consider milliseconds*.

**Can we use the superclass definition of `equals` except for when we compare two `PreciseDuration` objects?**
Here's the code:

```java
public class PreciseDuration extends Duration {
    // ...
    @Override public boolean equals(Object that) {
        if ( ! (that instanceof PreciseDuration)) {
            return super.equals(that);
        }
        PreciseDuration thatDuration = (PreciseDuration)that;
        return this.getMillisecs() == thatDuration.getMillisecs();
    }
}
```

**No**, because this implementation is not transitive!
Two different `PreciseDuration` objects can both be equal to the same `Duration` object, but not be equal to one another.

**There is no really satisfactory solution.**

The standard approach is that superclass equality should reject **all** subclass objects.
So in `Duration`, instead of:

```java
if ( ! (that instanceof Duration)) { return false; }
```

we should use:

```java
if ( ! that.getClass().equals(this.getClass())) { return false; }
```

But this solution is inflexible --- for example, it doesn't permit a subclass that *doesn't* add any new abstract values.

The better solution to this problem: **avoid subclassing, and use composition instead**.

## Interfaces and abstract classes

Java has two mechanisms for defining a type that can have multiple different implementations: **interfaces** and **abstract classes**.
An abstract class is a class that can only be subclassed, it cannot be instantiated.
There are two differences between the two mechanisms:

+ Abstract classes can provide implementations for some instance methods, while interfaces cannot.
  (New in Java 8 is a mechanism for providing "default" implementations of instance methods in interfaces, but this introduces a knot of tricky questions we'll leave aside.)

+ To implement the type defined by abstract class `A`, class `B` *must* be a subclass of abstract class `A` (declared with `extends`).
  But *any* class that defines all of the required methods and follows the specification of interface `I` can be a subtype of `I` (declared with `implements`).

In *Effective Java*, Bloch discusses the advantages of interfaces in *Item 17: Prefer interfaces to abstract classes*:

> Existing classes can be easily retrofitted to implement a new interface.
> All you have to do is add the required methods if they don't yet exist and add an `implements` clause to the class declaration.

&nbsp;

> Existing classes cannot, in general, be retrofitted to extend a new abstract class.
> If you want to have two classes extend the same abstract class, you have to place the abstract class high up in the type hierarchy where it subclasses an ancestor of both classes.

... but such a change can wreak havoc with the hierarchy.

With interfaces, we can build type structures that are not strictly hierarchical.
Bloch provides an excellent example:

> For example, suppose we have an interface representing a singer and another representing a songwriter:

> ```java
> public interface Singer {
>      AudioClip sing(Song s);
> }
> public interface Songwriter {
>      Song compose(boolean hit);
> }
> ```

> In real life, some singers are also songwriters.
> Because we used interfaces rather than abstract classes to define these types, it is perfectly permissible for a single class to implement both `Singer` and `Songwriter`.
> In fact, we can define a third interface that extends both `Singer` and `Songwriter` and adds new methods that are appropriate to the combination:

> ```java
> public interface SingerSongwriter extends Singer, Songwriter {
>      void actSensitive();
> }
> ```

A class `Troubadour` that `implements SingerSongwriter` must provide implementations of `sing`, `compose`, and `actSensitive`, and `Troubadour` instances can be used anywhere that code requires a `Singer` or a `Songwriter`.

If we are favoring interfaces over abstract classes, what should we do if we are defining a type that others will implement, and we want to provide code they can reuse?

A good strategy is to define an abstract **skeletal implementation** that goes along with the interface.
The type is still defined by the interface, but the skeletal implementation makes the type easier to implement.
For example, the Java library includes a skeletal implementation for each of the major interfaces in the collections framework: [`Abstract&shy;List`](java:java/util/AbstractList) implements [`List`](java:java/util/List), [`Abstract&shy;Set`](java:java/util/AbstractSet) for [`Set`](java:java/util/Set), etc.
If you are implementing a new kind of `List` or `Set`, you may be able to subclass the appropriate skeletal implementation, saving yourself work and relying on well-tested library code instead.

The skeletal implementation can also be combined with the [wrapper class pattern](#use_composition_rather_than_subclassing) described above.
If a class cannot extend the skeleton itself (perhaps it already has a superclass), then it can implement the interface and delegate method calls to an instance of a helper class (which could be a private [inner class](http://docs.oracle.com/javase/tutorial/java/javaOO/innerclasses.html)) which does extend the skeletal implementation.

Writing a skeletal implementation requires you to break down the interface and decide which operations can serve as *primitives* in terms of which the other operations can be defined.
These primitive operations will be left unimplemented in the skeleton (they will be abstract methods), because the author of the concrete implementation must provide them.
The rest of the operations are implemented in the skeleton, written in terms of the primitives.

For example, the Java [`Map`](java:java/util/Map) interface defines a number of different observers:

```java
public interface Map&lt;K,V> {
    public boolean containsKey(Object key);
    public boolean containsValue(Object value);
    public Set&lt;K> keySet();
    public Collection&lt;V> values();
    // ...
}
```

But all of these can be defined in terms of

```java
    public Set&lt;Map.Entry&lt;K,V>> entrySet();
```

which returns a set of [`Map.Entry`](java:java/util/Map.Entry) objects representing key/value pairs.

To make it easier to implement a new kind of `Map`, Java provides [`AbstractMap`](java:java/util/AbstractMap).
The documentation says:

> To implement an unmodifiable map, the programmer needs only to extend this class and provide an implementation for the `entrySet` method, which returns a set-view of the map's mappings.

So:

```java
public class MyMap&lt;K,V> extends AbstractMap&lt;K,V> {
    public Set&lt;Map.Entry&lt;K,V>> entrySet() {
        // my code here to return a set of key/value pairs in the map
    }
    // ...
}
```

And the skeletal implementation, written for us by the authors of `Map`, implements the other methods in terms of `entrySet`:

```java
public class AbstractMap&lt;K,V> implements Map&lt;K,V> {
    public boolean containsKey(Object key) {
        // simplified version, actual code must handle null :(
        for (Map.Entry&lt;K,V> entry : this.entrySet()) {
            if (entry.getKey().equals(key)) { return true; }
        }
        return false;
    }
    public boolean containsValue(Object value) {
        // simplified version, actual code must handle null :(
        for (Map.Entry&lt;K,V> entry : this.entrySet()) {
            if (entry.getValue().equals(value)) { return true; }
        }
        return false;
    }
    public Set&lt;K> keySet() {
        // return a set of just the keys from this.entrySet()
    }
    public Collection&lt;V> values() {
        // return a collection of just the values from this.entrySet()
    }
    // ...
}
```

## Summary

In this reading we've seen some **risks of subclassing**: subclassing breaks encapsulation, and we must use subclassing only for true subtyping.

The **substitution principle** says that B is a true subtype of A if and only if B objects are substitutable for A objects anywhere that A is used.
Equivalently, the specification of B must imply the specification of A.
Preconditions of the subtype must be the same or weaker, and postconditions the same or stronger.
Violating the substitution principle will yield code that doesn't make semantic sense and contains lurking bugs.

Favor **composition** over inheritance from a superclass: the wrapper pattern is an alternative to subclassing that preserves encapsulation and avoids the problems we've seen in several examples.
Since subclassing creates tightly-coupled code that lacks strong barriers of encapsulation to separate super- from subclasses, composition is useful for writing code that is safer from bugs, easier to understand, and more ready for change.
