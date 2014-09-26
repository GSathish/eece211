---
layout: page
title: Designing Specifications
---

Now, we will look at different specs for similar behaviours, and talk about the tradeoffs between them.

We will look at three dimensions for comparing specs:

+ How **deterministic** it is.
  Does the spec define only a single possible output for a given input, or allow the implementor to choose from a set of legal outputs?
+ How **declarative** it is.
  Does the spec just characterize *what* the output should be, or does it explicitly say *how* to compute the output?
+ How **strong** it is.
  Does the spec have a small set of legal implementations, or a large set?

## Deterministic vs. underdetermined specs

Recall the two example implementations of `find` we began with in the previous part:

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

Here is one possible specification of find:

<pre>
static int find(int[] a, int val)
  <em>requires</em>: val occurs exactly once in a
  <em>effects</em>:  returns index i such that a[i] = val
</pre>

This specification is deterministic: when presented with a state satisfying the precondition, the outcome is determined.
Both `findA` and `findB` satisfy the specification, so if this is the specification on which the clients relied, the two implementations are equivalent and substitutable for one another.
(Of course a procedure must have the *name* demanded by the specification; here we are using different names to allow us to talk about the two versions.
To use either, you'd have to change its name to `find`.)

Here is a slightly different specification:

<pre>
static int find(int[] a, int val)
  <em>requires</em>: val occurs in a
  <em>effects</em>:  returns index i such that a[i] = val
</pre>

This specification is not deterministic.
Such a specification is often said to be non-deterministic, but this is a bit misleading.
Non-deterministic code is code that you expect to sometimes behave one way and sometimes another.
This can happen, for example, with concurrency: the scheduler chooses to run threads in different orders depending on conditions outside the program.

But a 'non-deterministic' specification doesn't call for such non-determinism in the code.
The behaviour specified is not non-deterministic but *under-determined*.
In this case, the specification doesn't say which index is returned if `val` occurs more than once; it simply says that if you look up the entry at the index given by the returned value, you'll find val.

This specification is again satisfied by both `findA` and `findB`, each 'resolving' the under-determinedness in its own way.
A client of find can't predict which index will be returned, but should not expect the behaviour to be truly non-deterministic.
Of course, the specification is satisfied by a non-deterministic procedure too --- for example, one that rather improbably tosses a coin to decide whether to start searching from the top or the bottom of the array.
But in almost all cases we'll encounter, non-determinism in specifications offers a choice that is made by the implementor at implementation time, and not at runtime.

So for this specification, too, the two versions of find are equivalent.

Finally, here's a specification that distinguishes the two:

<pre>
static int find(int[] a, int val)
  <em>effects</em>: returns largest index i such that
             a[i] = val, or -1 if no such i
</pre>


## Declarative vs. operational specs

Roughly speaking, there are two kinds of specifications.
*Operational* specifications give a series of steps that the method performs; pseudocode descriptions are operational.
*Declarative* specifications don't give details of intermediate steps.
Instead, they just give properties of the final outcome, and how it's related to the initial state.

Almost always, declarative specifications are preferable.
They're usually shorter, easier to understand, and most importantly, they don't expose implementation details inadvertently that a client may rely on (and then find no longer hold when the implementation is changed).
For example, if we want to allow either implementation of `find`, we would *not* want to say in the spec that the method "goes down the array until it finds `val`," since aside from being rather vague, this spec suggests that the search proceeds from lower to higher indices and that the lowest will be returned, which perhaps the specifier did not intend.

One reason programmers sometimes lapse into operational specifications is because they're using the spec comment to explain the implementation for a maintainer.
Don't.
Do that using comments within the body of the method, not in the spec comment.

## Stronger vs. weaker specs

Suppose you want to substitute one method for another. How do you compare the specifications?

A specification **A** is stronger than or equal to a specification **B** if

 + **A**'s precondition is weaker than or equal to **B**'s
 + **A**'s postcondition is stronger than or equal to **B**'s, for the states that satisfy **B**'s precondition.

If this is the case, then an implementation that satisfies **A** can be used to satisfy **B** as well.

These two rules embody several ideas.
They tell you that you can always weaken the precondition; placing fewer demands on a client will never upset them.
And you can always strengthen the post-condition, which means making more promises.

For example, this spec for `find`:
<pre>
static int find1(int[] a, int val)
  <em>requires</em>: val occurs exactly once in a
  <em>effects</em>:  returns index i such that a[i] = val
</pre>
can be replaced in any context by:
<pre>
static int findStronger2(int[] a, int val)
  <em>requires</em>: val occurs at least once in a
  <em>effects</em>:  returns index i such that a[i] = val
</pre>
which has a weaker precondition.
This in turn can be replaced by:
<pre>
static int findStronger3(int[] a, int val)
  <em>requires</em>: val occurs at least once in a
  <em>effects</em>:  returns lowest index i such that a[i] = val
</pre>
which has a stronger postcondition.  

What about this specification:
<pre>
static int find4(int[] a, int val)
  <em>requires</em>: nothing
  <em>effects</em>:  returns index i such that a[i] = val,
              or -1 if no such i
</pre>

## Diagramming specifications

Imagine (very abstractly) the space of all possible Java methods.

Each point in this space represents a method implementation.

Here we'll diagram `findA` and `findB` defined [above](#deterministic_vs_underdetermined_specs).

A specification defines a *region* in the space of all possible implementations.
A given implementation either behaves according to the spec, satisfying the precondition-implies-postcondition contract (it is inside the region), or it does not (outside the region). <img src="https://dl.dropboxusercontent.com/u/567187/EECE%20210/Images/Designing%20Specifications/fig1.png"></img>

Both `findA` and `findB` satisfy *findStronger2*, so they are inside the region defined by that spec. 

<img src="https://dl.dropboxusercontent.com/u/567187/EECE%20210/Images/Designing%20Specifications/fig2.png"></img>

We can imagine clients looking in on this space: the specification acts as a firewall.
Implementors have the freedom to move around inside the spec, changing their code without fear of upsetting a client.
Clients don't know which implementation they will get.
They must respect the spec, but also have the freedom to change how they're using the implementation without fear that it will suddenly break.

How will similar specifications relate to one another?
Suppose we start with specification **S1** and use it to create a new specification **S2**.

If **S2** is stronger than **S1**, how will these specs appear in our diagram?

+ Let's start by **strengthening the postcondition**.
  If **S2**'s postcondition is now stronger than **S1**'s, **S2** is the stronger specification.

  Think about what strengthening the postcondition means for implementors: it means they have less freedom, the requirements on their output are stronger.
  Perhaps they previously satisfied *findStronger2* by returning any index `i`, but now the spec demands the *lowest* index `i`.
  So there are now implementations *inside findStronger2* but *outside findStronger3*.

  Could there be implementations *inside findStronger3* but *outside findStronger2*?
  No.
  All of those implementations satisfy a stronger postcondition than what *findStronger2* demands.

+ Think through what happens if we **weaken the precondition**, which will again make **S2** a stronger specification.
  Implementations will have to handle new inputs that were previously excluded by the spec.
  If they behaved badly on those inputs before, we wouldn't have noticed, but now their bad behaviour is exposed.

<img src="https://dl.dropboxusercontent.com/u/567187/EECE%20210/Images/Designing%20Specifications/fig3.png"></img>

We see that when **S2** is stronger than **S1**, it defines a *smaller* region in this diagram; a weaker specification defines a larger region.

In our figure, since `findB` iterates from the end of the array `a`, it does not satisfy *findStronger3* and is outside that region.

A specification **S2** that is neither stronger nor weaker than **S1** might overlap (such that there exist implementations that satisfy only **S1**, only **S2**, and both **S1** and **S2**) or might be disjoint.

## Designing good specifications

What makes a good method?
Designing a method means primarily writing a specification.

About the form of the specification: it should obviously be succinct, clear, and well-structured, so that it's easy to read.

The content of the specification, however, is harder to prescribe.
There are no infallible rules, but there are some useful guidelines.

**The specification should be coherent**: it shouldn't have lots of different cases.
Long argument lists, deeply nested if-statements, and boolean flags are a sign of trouble.
Consider this specification:

<pre>
static int minFind(int[] a, int[] b, int val)
  <em>effects</em>: returns smallest index in arrays a and b at which
             val appears
</pre>

Is this a well-designed procedure?
Probably not: it's incoherent, since it does two things (finding and minimizing) that are not really related.
It would be better to use two separate procedures.

**The results of a call should be informative**.
Consider the specification of a method that puts a value in a map:

<pre>
static V put (Map&lt;K,V> map, K key, V val)
  <em>requires</em>: val may be null, and map may contain null values
  <em>effects</em>:  inserts (key, val) into the mapping,
              overriding any existing mapping for key, and
              returns old value for key, unless none,
              in which case it returns null
</pre>

Note that the precondition does not rule out `null` values so the map can store `null`s.
But the postcondition uses `null` as a special return value for a missing key.
This means that if `null` is returned, you can't tell whether the key was not bound previously, or whether it was in fact bound to `null`.
This is not a very good design, because the return value is useless unless you know for sure that you didn't insert `nulls`.

**The specification should be strong enough**.
There's no point throwing a checked exception for a bad argument but allowing arbitrary mutations, because a client won't be able to determine what mutations have actually been made.
Here's a specification illustrating this flaw (and also written in an inappropriately operational style):

<pre>
static void addAll(List&lt;T> list1, List&lt;T> list2)
  <em>effects</em>: adds the elements of list2 to list1,
             unless it encounters a null element,
             at which point it throws a NullPointerException
</pre>

**The specification should also be weak enough**. Consider this specification for a method that opens a file:

<pre>
static File open(String filename)
  <em>effects</em>: opens a file named filename
</pre>

This is a bad specification.
It lacks important details: is the file opened for reading or writing?
Does it already exist or is it created?
And it's too strong, since there's no way it can guarantee to open a file.
The process in which it runs may lack permission to open a file, or there might be some problem with the file system beyond the control of the program.
Instead, the specification should say something much weaker: that it attempts to open a file, and if it succeeds, the file has certain properties.

**The specification should use *abstract types* where possible**, giving more freedom to both the client and the implementor.
In Java, this often means using an interface type, like `Map` or `Reader`, instead of specific implementation types like `HashMap` or `FileReader`.
Consider this specification:

<pre>
static ArrayList&lt;T> reverse(ArrayList&lt;T> list)
  <em>effects</em>: returns a new list which is the reversal of list, i.e.
             newList[i] == list[n-i-1]
             for all 0 &lt;= i &lt; n, where n = list.size()
</pre>

This forces the client to pass in an `ArrayList`, and forces the implementor to return an `ArrayList`, even if there might be alternative `List` implementations that they would rather use.
Since the behaviour of the specification doesn't depend on anything specific about *`ArrayList`*, it would be better to write this spec in terms of the more abstract `List&lt;T>`.

## Precondition or postcondition?

Another design issue is whether to use a precondition, and if so, whether the method code should attempt to make sure the precondition has been met before proceeding.
In fact, the most common use of preconditions is to demand a property precisely because it would be hard or expensive for the method to check it.

As mentioned above, a non-trivial precondition inconveniences clients, because they have to ensure that they don't call the method in a bad state (that violates the precondition); if they do, there is no predictable way to recover from the error.
So users of methods don't like preconditions.
That's why the Java API classes, for example, invariably specify (as a postcondition) that they throw unchecked exceptions when arguments are inappropriate.
This approach makes it easier to find the bug or incorrect assumption in the caller code that led to passing bad arguments.
In general, it's better to **fail fast**, as close as possible to the site of the bug, rather than let bad values propagate through a program far from their original cause.

Sometimes, it's not feasible to check a condition without making a method unacceptably slow, and a precondition is often necessary in this case.
If we wanted to implement the `find()` method using binary search, we would have to require that the array be sorted.
Forcing the method to actually *check* that the array is sorted would defeat the entire purpose of the binary search: to obtain a result in logarithmic and not linear time.

The decision of whether to use a precondition is an engineering judgment.
The key factors are the cost of the check (in writing and executing code), and the scope of the method.
If it's only called locally in a class, the precondition can be discharged by carefully checking all the sites that call the method.
But if the method is public, and used by other developers, it would be less wise to use a precondition.
Instead, like the Java API classes, you should throw an exception.

## About access control

We have been using *public* for almost all of our methods, without really thinking about it. The decision to make a method `public` or `private` is actually a decision about the contract of the class.

<p class="graybox" style="text-align: right;">
  <strong>Readings</strong><br />
  <a href="http://docs.oracle.com/javase/tutorial/java/package/index.html">Packages</a> in the Java Tutorials.<br />
  <a href="http://docs.oracle.com/javase/tutorial/java/javaOO/accesscontrol.html">Controlling Access</a> in the Java Tutorials.
</p>


Public methods are freely accessible to other parts of the program. Making a method public advertises it as a service that your class is willing to provide. If you make all your methods public --- including helper methods that are really meant only for local use within the class --- then other parts of the program may come to depend on them, which will make it harder for you to change the internal implementation of the class in the future.

Your code won't be as **ready for change**.  

Making internal helper methods public will also add clutter to the visible interface your class offers.
Keeping internal things *private* makes your class's public interface smaller and more coherent (meaning that it does one thing and does it well).
Your code will be **easier to understand**.

We will see even stronger reasons to use *private* when we start to write classes with persistent internal state.
Protecting this state will help keep the program **safe from bugs**.
