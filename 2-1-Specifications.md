---
layout: page
title: Need for Specifications
---

## Why specifications?

Many of the nastiest bugs in programs arise because of misunderstandings about behaviour at the interface between two pieces of code.
Although every programmer has specifications in mind, not all programmers write them down.
As a result, different programmers on a team have *different* specifications in mind.
When the program fails, it's hard to determine where the error is.
Precise specifications in the code let you apportion blame (to code fragments, not people!), and can spare you the agony of puzzling over where a fix should go.

Specifications are good for the client of a method because they spare the task of reading code.
If you're not convinced that reading a spec is easier than reading code, take a look at some of the standard Java specs and compare them to the source code that implements them.
[`ArrayList`](java:java/util/ArrayList), for example, in the package `java.util`, has a very simple spec but its code is not at all simple.

Specifications are good for the implementer of a method because they give the freedom to change the implementation without telling clients.
Specifications can make code faster, too.
Sometimes a weak specification makes it possible to do a much more efficient implementation.
In particular, a precondition may rule out certain states in which a method might have been invoked that would have incurred an expensive check that is no longer necessary.

The contract acts as a *firewall* between client and implementor.
It shields the client from the details of the *workings* of the unit --- you don't need to read the source code of the procedure if you have its specification.
And it shields the implementor from the details of the *usage* of the unit; he doesn't have to ask every client how she plans to use the unit.
This firewall results in *decoupling*, allowing the code of the unit and the code of a client to be changed independently, so long as the changes respect the specification --- each obeying its obligation.

## behavioural equivalence

Consider these two methods.
Are they the same or different?

{% highlight java %}
static int findA(int[] a, int val) {
    for (int i = 0; i &lt; a.length; i++) {
        if (a[i] == val) return i;
    }
    return a.length;
}

static int findB(int[] a, int val) {
    for (int i = a.length -1 ; i >= 0; i--) {
        if (a[i] == val) return i;
    }
    return -1;
}
{% endhighlight %}

Of course the code is different, so in that sense they are different.
Our question is whether we could substitute one implementation for the other.
Not only do these methods have different code, they actually have different behaviour:

+ when `val` is missing, `findA` returns the length of `a` and `findB` returns -1;
+ when `val` appears twice, `findA` returns the lower index and `findB` returns the higher.

But when `val` occurs at exactly one index of the array, the two methods behave the same.
It may be that clients never rely on the behaviour in the other cases.
So the notion of equivalence is in the eye of the beholder, that is, the client.
In order to make it possible to substitute one implementation for another, and to know when this is acceptable, we need a specification that states exactly what the client depends on.

In this case, our specification might be:

<pre>
*requires*: val occurs in a
*effects*:  returns index i such that a[i] = val
</pre>

## Specification structure

A specification of a method consists of several clauses:

+ a *precondition*, indicated by the keyword *requires*
+ a *postcondition*, indicated by the keyword *effects*

The precondition is an obligation on the client (i.e., the caller of the method).
It's a condition over the state in which the method is invoked.
If the precondition does not hold, the implementation of the method is free to do anything (including not terminating, throwing an exception, returning arbitrary results, making arbitrary modifications, etc.).

The postcondition is an obligation on the implementer of the method.
If the precondition holds for the invoking state, the method is obliged to obey the postcondition, by returning appropriate values, throwing specified exceptions, modifying or not modifying objects, and so on.

The overall structure is a logical implication: *if* the precondition holds when the method is called, *then* the postcondition must hold when the method completes.

### Specifications in Java

Some languages (notably [Eiffel](http://en.wikipedia.org/wiki/Eiffel_(programming_language))) incorporate preconditions and postconditions as a fundamental part of the language, as expressions that the runtime system (or even the compiler) can automatically check to enforce the contracts between clients and implementers.

Java does not go quite so far, but its static type declarations *are* effectively part of the precondition and postcondition of a method, a part that is automatically checked and enforced by the compiler.
The rest of the contract --- the parts that we can't write as types --- must be described in a comment preceding the method, and generally depends on human beings to check it and guarantee it.

Java has a convention for [documentation comments](http://en.wikipedia.org/wiki/Javadoc), in which parameters are described by `@param` clauses and results are described by `@return` and `@throws` clauses.
You should put the preconditions into `@param` where possible, and postconditions into `@return` and `@throws`.
So a specification like this:

<pre>
static int find(int[] a, int val)
  *requires*: val occurs exactly once in a
  *effects*:  returns index i such that a[i] = val
</pre>

... might be rendered in Java like this:

{% highlight java %}
/**
 * Find value in an array.
 * @param a array to search; requires that val occurs exactly once in a
 * @param val value to search for
 * @return index i such that a[i] = val
 */
static int find(int[] a, int val)
{% endhighlight %}

The [Java API documentation] is produced from Javadoc comments in the [Java standard library source code].
Documenting your specifications in Javadoc allows Eclipse to show you (and clients of your code) useful information, and allows you to [produce HTML documentation][Eclipse Javadoc] in the same format as the Java API docs.

[Java API documentation]: http://docs.oracle.com/javase/8/docs/api/
[Java standard library source code]: http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java
[Eclipse Javadoc]: http://help.eclipse.org/luna/index.jsp?topic=%2Forg.eclipse.jdt.doc.user%2Freference%2Fref-export-javadoc.htm

You can also refer to Oracle's **[How to Write Doc Comments]**.
[Javadoc Comments]: http://javaworkshop.sourceforge.net/chapter4.html
[How to Write Doc Comments]: http://www.oracle.com/technetwork/java/javase/documentation/index-137868.html
</div>

### Null references

In Java, references to objects and arrays can also take on the special value *null*, which means that the reference doesn't point to an object.
Null values are an unfortunate hole in Java's type system.
You can assign null to any non-primitive variable:

{% highlight java %}
String s = null;
int a[] = null;
{% endhighlight %}

and the compiler happily accepts this code at compile time.
But you'll get errors at runtime because you can't call any methods or use any fields with one of these references:

{% highlight java %}
s.length()    // throws NullPointerException  
a.length      // throws NullPointerException
{% endhighlight %}

Note, in particular, that `null` is not the same as an empty string `""` or an empty array.
On an empty string or empty array, you *can* call methods and access fields.
The length of an empty array or an empty string is 0.
The length of a string variable that points to `null` throws a `NullPointer&shy;Exception`.

Null values are troublesome and unsafe, so much so that you're well advised to remove them from your design vocabulary.
In 6.005 --- and in fact in most good Java programming --- **null values are implicitly disallowed as parameters and return values**.
So every method implicitly has a precondition on its object and array parameters that they be non-null.
Every method that returns an object or an array implicitly has a postcondition that its return value is non-null.
If a method allows null values for a parameter, it should explicitly state it, or if it might return a null value as a result, it should explicitly state it.
But these are in general not good ideas.
**Avoid null**.

There are extensions to Java that allow you to forbid `null` directly in the type declaration, e.g.:

{% highlight java %}
static boolean addAll(@NonNull List&lt;T> list1, @NonNull List&lt;T> list2)
{% endhighlight %}

where it can be [checked automatically](http://types.cs.washington.edu/checker-framework/) at compile time or runtime.

## What a specification may talk about

A specification of a method can talk about the parameters and return value of the method, but it should never talk about local variables of the method or private fields of the method's class.
You should consider the implementation invisible to the reader of the spec.

In Java, the source code of the method is often unavailable to the reader of your spec, because the Javadoc tool extracts the spec comments from your code and renders them as HTML.

## Testing and specifications

In testing, we talk about *black box tests* that are chosen with only the specification in mind, and *glass box tests* that are chosen with knowledge of the actual implementation.
But it's important to note that **even glass box tests must follow the specification**.
Your implementation may provide stronger guarantees than the specification calls for, or it may have specific behaviour where the specification is undefined.
But your test cases should not count on that behaviour.
Test cases must obey the contract, just like every other client.

For example, suppose you are testing this specification of `find`:

<pre>
static int find(int[] a, int val)
  *requires*: val occurs in a
  *effects*:  returns index i such that a[i] = val
</pre>

This spec has a strong precondition in the sense that `val` is required to be found; and it has a fairly weak postcondition in the sense that if `val` appears more than once in the array, this specification says nothing about which particular index of `val` is returned.
Even if you implemented `find` so that it always returns the lowest index, your test case can't assume that specific behaviour:

<pre class="no-markdown">
<code class="java">int[] array = new int[] { 7, 7, 7 };
<strike>assertEquals(0, find(array, 7));</strike>  // bad test case: violates the spec
assertEquals(7, array[find(array, 7)]);  // correct
</code></pre>

Similarly, even if you implemented `find` so that it (sensibly) throws an exception when `val` isn't found, instead of returning some arbitrary misleading index, your test case can't assume that behaviour, because it can't call `find()` in a way that violates the precondition.

So what does glass box testing mean, if it can't go beyond the spec?
It means you are trying to find new test cases that exercise different parts of the implementation, but still checking those test cases in an implementation-independent way.

## Specifications for mutating methods

We previously discussed mutable vs. immutable objects, but our specifications of `find` didn't give us the opportunity to illustrate how to describe side-effects --- changes to mutable data --- in the postcondition.

Here's a specification that describes a method that mutates an object:

<pre class="no-markdown">
static boolean addAll(List&lt;T> list1, List&lt;T> list2)
  <em>requires</em>: list1 != list2
  <em>effects</em>:  modifies list1 by adding the elements of list2 to the end of
              it, and returns true if list1 changed as a result of call</pre>

We've taken this, slightly simplified, from the Java [`List`](java:java/util/List) interface.
First, look at the postcondition.
It gives two constraints: the first telling us how `list1` is modified, and the second telling us how the return value is determined.

Second, look at the precondition.
It tells us that the behaviour of the method if you attempt to add the elements of a list to itself is undefined.
You can easily imagine why the implementor of the method would want to impose this constraint: it's not likely to rule out any useful applications of the method, and it makes it easier to implement.
The specification allows a simple implementation in which you take an element from `list2` and add it to `list1`, then go on to the next element of `list2` until you get to the end.
If `list1` and `list2` are the same list, this algorithm will not terminate --- an outcome permitted by the specification.

Remember also our implicit precondition that `list1` and `list2` must be valid objects, rather than `null`.
We'll usually omit saying this because it's virtually always required of object references.

Here is another example of a mutating method:

<pre class="no-markdown">
static void sort(List&lt;String> lst)
  <em>requires</em>: nothing
  <em>effects</em>:  puts lst in sorted order, i.e. lst[i] &lt;= lst[j]
              for all 0 &lt;= i &lt; j &lt; lst.size()</pre>

And an example of a method that does not mutate its argument:

<pre class="no-markdown">
static List&lt;String> toLowerCase(List&lt;String> lst)
  <em>requires</em>: nothing
  <em>effects</em>:  returns a new list t where t[i] = lst[i].toLowerCase()</pre>

Just as we've said that `null` is implicitly disallowed unless stated otherwise, we will also use the convention that **mutation is disallowed unless stated otherwise**.
The spec of `to&shy;Lower&shy;Case` could explicitly state as an *effect* that "lst is not modified", but in the absence of a postcondition describing mutation, we demand no mutation of the inputs.
