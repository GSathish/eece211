#### Objectives of Good Software Construction

<table class="table table-striped no-markdown">
<tr><th width="33%">Safe from bugs</th><th>Easy to understand</th><th>Ready for change</th></tr>
<tr><td>
  Correct today and correct in the unknown future.
</td><td>
  Communicating clearly with future programmers, including future you.
</td><td>
  Designed to accommodate change without rewriting.
</td></tr>
</table>

#### Objectives of This Reading

+ Mutable objects
+ Aliasing and the dangers of mutability
+ Immutability

#### Required reading

**[the `static` keyword](http://www.codeguru.com/java/tij/tij0037.shtml#Heading79)** and **[the `final` keyword](http://www.codeguru.com/java/tij/tij0071.shtml)** on CodeGuru.

## Risks of mutation

Recall from our discussion of instance diagrams that some objects are *immutable*: once created, they always represent the same value.
Other objects are *mutable*: they have methods that change the value of the object.

[`String`](java:java/lang/String) is an example of an immutable type.
A `String` object always represents the same string.
[`StringBuilder`](java:java/lang/StringBuilder) is an example of a mutable type.
It has methods to delete parts of the string, insert or replace characters, etc.

Mutable types seem much more powerful than immutable types.
If you were shopping in the Datatype Supermarket, and you had to choose between a boring immutable `String` and a super-powerful-do-anything mutable `StringBuilder`, why on earth would you choose the immutable one?
`StringBuilder` should be able to do everything that `String` can do, plus `set()` and `append()` and everything else.

The answer is that immutable types are **safer from bugs**, **easier to understand**, and **more ready for change**.
Here are two examples that illustrate why.

### Risky example #1: passing mutable values

Let's start with a simple method that sums the integers in a list:

```java
/** @return the sum of the numbers in the list */
public static int sum(List<Integer> list) {
    int sum = 0;
    for (int x : list)
        sum += x;
    return sum;
}
```

Suppose we also need a method that sums the absolute values.
Following good DRY practice ([Don't Repeat Yourself](http://en.wikipedia.org/wiki/Don't_repeat_yourself)), the implementer writes a method that uses `sum()`:

```java
/** @return the sum of the absolute values of the numbers in the list */
public static int sumAbsolute(List<Integer> list) {
    // let's reuse sum(), because DRY, so first we take absolute values
    for (int i = 0; i < list.size(); ++i)
        list.set(i, Math.abs(list.get(i)));
    return sum(list);
}
```

Notice that this method does its job by **mutating the list directly**.
It seemed sensible to the implementer, because it's more efficient to reuse the existing list.
If the list is millions of items long, then you're saving the time and memory of generating a new million-item list of absolute values.
So the implementer has two very good reasons for this design: DRY, and performance.

But the resulting behavior will be very surprising to anybody who uses it!
For example:

```java
// meanwhile, somewhere else in the code...
public static void main(String[] args) {
    // ...
    List<Integer> myData = Arrays.asList(-5, -3, -2);
    System.out.println(sumAbsolute(myData));
    System.out.println(sum(myData));
}
```

What will this code print?
Will it be 10 followed by -10?
Or something else?

Let's think about the key points here:

+ **Safe from bugs?**
  In this example, it's easy to blame the implementer of `sumAbsolute()` for going beyond what its spec allowed.
  But really, **passing mutable objects around is a latent bug**.
  It's just waiting for some programmer to inadvertently mutate that list, often with very good intentions like reuse or performance, but resulting in a bug that may be very hard to track down. 
+ **Easy to understand?**
  When reading `main()`, what would you assume about `sum()` and `sumAbsolute()`?
  Is it clearly visible to the reader that `myData` gets *changed* by one of them? 

### Risky example #2: returning mutable values

We just saw an example where passing a mutable object to a function caused problems.
What about returning a mutable object?

Let's consider [`Date`](java:java/util/Date), one of the built-in Java classes.
`Date` happens to be a mutable type.
Suppose we write a method that determines the first day of spring:

```java
/** @return the first day of spring this year */
public static Date startOfSpring() {
    return askGroundhog();
}
```

Here we're using the well-known Groundhog algorithm for calculating when spring starts (Harold Ramis, Bill Murray, et al. *Groundhog Day*, 1993).

Clients start using this method, for example to plan their big parties:

```java
// somewhere else in the code...
public static void partyPlanning() {
    Date partyDate = startOfSpring();
    // ...
}
```

All the code works and people are happy.
Now, independently, two things happen.
First, the implementer of `startOfSpring()` realizes that the groundhog is starting to get annoyed from being constantly asked when spring will start.
So the code is rewritten to ask the groundhog at most once, and then cache the groundhog's answer for future calls:

```java
/** @return the first day of spring this year */
public static Date startOfSpring() {
    if (groundhogAnswer == null) groundhogAnswer = askGroundhog();
    return groundhogAnswer;
}
private static Date groundhogAnswer = null;
```

(Aside: note the use of a private static variable for the cached answer.
Would you consider this a global variable, or not?)

Second, one of the clients of `startOfSpring()` decides that the actual first day of spring is too cold for the party, so the party will be exactly a month later instead:

```java
// somewhere else in the code...
public static void partyPlanning() {
    // let's have a party one month after spring starts!
    Date partyDate = startOfSpring();
    partyDate.setMonth(partyDate.getMonth() + 1);
    // ... uh-oh. what just happened?
}
```

(Aside: this code also has a latent bug in the way it adds a month.
Why?
What does it implicitly assume about when spring starts?)

What happens when these two decisions interact?
Even worse, think about who will first discover this bug --- will it be `startOfSpring()`?
Will it be `partyPlanning()`?
Or will it be some completely innocent third piece of code that also calls `startOfSpring()`?

Key points:

+ **Safe from bugs?**
  Again we had a latent bug that reared its ugly head.
+ **Ready for change?**
  Obviously the mutation of the date object is a change, but that's not the kind of change we're talking about when we say "ready for change."
  Instead, the question is whether the code of the program can be easily changed without rewriting a lot of it or introducing bugs.
  Here we had two apparently independent changes, by different programmers, that interacted to produce a bad bug.

In both of these examples --- the `List<Integer>` and the `Date` --- the problems would have been completely avoided if the list and the date had been immutable types.
The bugs would have been impossible by design.

In fact, you should never use `Date`!
Use one of the classes from [package `java.time`](java:java/time/package-summary.html): [`LocalDateTime`](java:java/time/LocalDateTime), [`Instant`](java:java/time/Instant), etc.
All guarantee in their specifications that they are *immutable*.

This example also illustrates why using mutable objects can actually be *bad* for performance.
The simplest solution to this bug, which avoids changing any of the specifications or method signatures, is for `startOfSpring()` to always return a *copy* of the groundhog's answer:

```java
    return new Date(groundhogAnswer.getTime());
```

This pattern is **defensive copying**, and we'll see much more of it when we talk about abstract data types.
The defensive copy means `partyPlanning()` can freely stomp all over the returned date without affecting `startOfSpring()`'s cached date.
But defensive copying forces `startOfSpring()` to do extra work and use extra space for *every client* --- even if 99% of the clients never mutate the date it returns.
We may end up with lots of copies of the first day of spring throughout memory.
If we used an immutable type instead, then different parts of the program could safely share the same values in memory, so less copying and less memory space is required.
Immutability can be more efficient than mutability, because immutable types never need to be defensively copied.

## Aliasing is what makes mutable types risky

Actually, using mutable objects is just fine if you are using them entirely locally within a method, and with only one reference to the object.
What led to the problem in the two examples we just looked at was having multiple references, also called **aliases**, for the same mutable object.

Walking through the examples with a instance diagram will make this clear, but here's the outline:

+ In the `List` example, the same list is pointed to by both `list` (in `sum` and `sumAbsolute`) and `myData` (in `main`).
  One programmer (`sumAbsolute`'s) thinks it's ok to modify the list; another programmer (`main`'s) wants the list to stay the same.
  Because of the aliases, `main`'s programmer loses. 
+ In the `Date` example, there are two variable names that point to the `Date` object, `groundhogAnswer` and `partyDate`.
  These aliases are in completely different parts of the code, under the control of different programmers who may have no idea what the other is doing.

Draw instance diagrams on paper first, but your real goal should be to develop the instance diagram in your head, so you can visualize what's happening in the code.

## Specifications for mutating methods

At this point it should be clear that when a method performs mutation, it is crucial to include that mutation in the method's spec, using [the structure we discussed in the previous reading][specs for mutating methods].

(Now we've seen that even when a particular method *doesn't* mutate an object, that object's mutability can still be a source of bugs.)

Here's an example of a mutating method:

```
static void sort(List<String> lst)
  <em>requires</em>: nothing
  <em>effects</em>:  puts lst in sorted order, i.e. lst[i] <= lst[j]
              for all 0 <= i < j < lst.size()
```

And an example of a method that does not mutate its argument:

```
static List<String> toLowerCase(List<String> lst)
  <em>requires</em>: nothing
  <em>effects</em>:  returns a new list t where t[i] = lst[i].toLowerCase()
```

If the _effects_ do not describe mutation, in EECE 210 we assume mutation of the inputs is implicitly disallowed.

## Iterating over arrays and lists

The next mutable object we're going to look at is an **iterator** --- an object that steps through a collection of elements and returns the elements one by one.
Iterators are used under the covers in Java when you're using a `for` loop to step through a `List` or array.
This code:

```java
List<String> lst = ...;
for (String str : lst) {
    System.out.println(str);
}
```

is rewritten by the compiler into something like this:

```java
List<String> lst = ...;
Iterator iter = lst.iterator();
while (lst.hasNext()) {
	String str = iter.next();
	System.out.println(str);
}
```

An iterator has two methods:

+ `next()` returns the next element in the collection
+ `hasNext()` tests whether the iterator has reached the end of the collection.

Note that the `next()` method is a **mutator** method, not only returning an element but also advancing the iterator so that the subsequent call to `next()` will return a different element.

You can also look at the [Java API definition of `Iterator`](java:java/util/Iterator).

### `MyIterator`

To better understand how an iterator works, here's a simple implementation of an iterator for `ArrayList<String>`:

<div class="panel panel-figure pull-right pull-margin">
```java
/**
 * A MyIterator is a mutable object that iterates over
 * the elements of an ArrayList<String>, from first to last.
 * This is just an example to show how an iterator works.
 * In practice, you should use the ArrayList's own iterator
 * object, returned by its iterator() method.
 */
public class MyIterator {
    
    private final ArrayList<String> list;
    private int index;
    // list[index] is the next element that will be returned
    //   by next()
    // index == list.size() means no more elements to return
    
    /**
     * Make an iterator.
     * @param list list to iterate over
     */
    public MyIterator(ArrayList<String> list) {
        this.list = list;
        this.index = 0;
    }
    
    /**
     * Test whether the iterator has more elements to return.
     * @return true if next() will return another element,
     *         false if all elements have been returned
     */
    public boolean hasNext() {
        return index < list.size();
    }
    
    /**
     * Get the next element of the list.
     * Requires: hasNext() returns true.
     * Modifies: this iterator to advance it to the element 
     *           following the returned element.
     * @return next element of the list
     */
    public String next() {
        final String str = list.get(index);
        ++index;
        return str;
    }
}
```
</div>

`MyIterator` makes use of a few Java language features that are different from the classes we've been writing up to this point.
Make sure you read the Java Tutorial sections required for this reading so that you understand them:

[**Instance variables**](http://docs.oracle.com/javase/tutorial/java/javaOO/variables.html), also called fields in Java.
Instance variables differ from method parameters and local variables; the instance variables are stored in the object instance and persist for longer than a method call.
What are the instance variables of `MyIterator`?

A [**constructor**](http://docs.oracle.com/javase/tutorial/java/javaOO/constructors.html), which makes a new object instance and initializes its instance variables.
Where is the constructor of `MyIterator`?

The `static` keyword is missing from `MyIterator`'s methods, which means they are [**instance methods**](http://docs.oracle.com/javase/tutorial/java/javaOO/methods.html) that must be called on an instance of the object, e.g. `iter.next()`.

The [**`this` keyword**](http://docs.oracle.com/javase/tutorial/java/javaOO/thiskey.html) is used at one point to refer to the **instance object**, in particular to refer to an instance variable (`this.list`).
This was done to disambiguate two different variables named `list` (an instance variable and a constructor parameter).
Most of `MyIterator`'s code refers to instance variables without an explicit `this`, but this is just a convenient shorthand that Java supports --- e.g., `index` actually means `this.index`.

**`private`** is used for the object's internal state and internal helper methods, while `public` indicates methods and constructors that are intended for clients of the class ([access control](http://docs.oracle.com/javase/tutorial/java/javaOO/accesscontrol.html)).

**`final`** is used to indicate which parts of the object's internal state can change and which can't.
`index` is allowed to change (`next()` updates it as it steps through the list), but `list` cannot (the iterator has to keep pointing at the same list for its entire life --- if you want to iterate through another list, you're expected to create another iterator object).

Here's a instance diagram showing a typical state for a MyIterator object in action:

<div class="panel panel-figure pull-right pull-margin">
<div class="panel-body">
<img src="figures/iterator.png" width="400"></img>
</div></div>

Note that we drew the arrow from `list` with a double line, to indicate that it's *final*.
That means the arrow can't change once it's drawn.
But the `ArrayList` object it points to is mutable --- elements can be changed within it --- and declaring `list` as final has no effect on that.

Why do iterators exist?
There are many kinds of collection data structures (linked lists, maps, hash tables) with different kinds of internal representations.
The iterator concept allows a single uniform way to access them all, so that client code is simpler and the collection implementation can change without changing the client code that iterates over it.
Most modern languages (including Python, C#, and Ruby) use the notion of an iterator.
It's an effective **design pattern** (a well-tested solution to a common design problem).
We'll see many other design patterns as we move through the course.

## Mutation undermines an iterator

Let's try using our iterator for a simple job.
Suppose we have a list of UBC courses represented as strings, like `["EECE 210", "EECE 259", "APSC 201"]`.
We want a method `dropEECECourses` that will delete the EECE courses from the list, leaving the other courses behind.
Following good practices, we first write the spec:

```java
/**
 * Drop all subjects that are from Course 6. 
 * Modifies subjects list by removing subjects that start with 6.
 * 
 * @param subjects list of UBC courses
 */
public static void dropCourseEECECourses(ArrayList<String> subjects)
```

Note that `dropEECECourses` has a frame condition (the *modifies* clause) in its contract, warning the client that its list argument will be mutated.

Next, following test-first programming, we devise a testing strategy that partitions the input space, and choose test cases to cover that partition:

    // Testing strategy:
    //   subjects.size: 0, 1, n
    //   contents: no EECE course, one EECE course, all EECE courses
    //   position: EECE course at start, EECE course in middle, EECE course at end

    // Test cases:
    //   [] => []
    //   [“APSC 201”] => [“APSC 201”]
    //   [“ENGL 112”, “MATH 253”, “ECON 102”] => [“ENGL 112”, “MATH 253”, “ECON 102”]
    //   [“PHIL 120”, “EECE 210”, “MATH 256”] => [“PHIL 120”, “MATH 256”]
    //   [“EECE 251”, “EECE 210”, “EECE 253”] => []

Finally, we implement it:

```java
public static void dropEECECourses(ArrayList<String> subjects) {
    MyIterator iter = new MyIterator(subjects);
    while (iter.hasNext()) {
        String subject = iter.next();
        if (subject.startsWith("6.")) {
            subjects.remove(subject);
        }
    }
}
```

Now we run our test cases, and they work! ... almost.
The last test case fails:

    // dropEECECourses([“EECE 210”, “EECE 259”, “EECE 281”])
    //   expected [], actual [“EECE 259”]

We got the wrong answer: `dropEECECourses` left a course behind in the list!
Why?
Trace through what happens.
It will help to use a instance diagram showing the `MyIterator` object and the `ArrayList` object and update it while you work through the code.

Note that this isn't just a bug in our `MyIterator`.
The built-in iterator in `ArrayList` suffers from the same problem, and so does the `for` loop that's syntactic sugar for it.
The problem just has a different symptom.
If you used this code instead:

```java
for (String subject : subjects) {
    if (subject.startsWith(“EECE”)) {
        subjects.remove(subject);
    }
}
```

then you'll get a [`ConcurrentModificationException`](java:java/util/ConcurrentModificationException).
The built-in iterator detects that you're changing the list under its feet, and cries foul.
(How do you think it does that?)

How can you fix this problem?
One way is to use the `remove()` method of `Iterator`, so that the iterator adjusts its index appropriately:

<pre class="no-markdown">
<code class="java">Iterator iter = subjects.iterator();
while (iter.hasNext()) {
    String subject = iter.next();
    if (subject.startsWith("6.")) {
        <strong class="text-danger">iter.remove(subject);</strong>
    }
}
</code></pre>
        
This is actually more efficient as well, it turns out, because `iter.remove()` already knows where the element it should remove is, while `subjects.remove()` had to search for it again.

But this doesn't fix the whole problem.
What if there are other `Iterator`s currently active over the same list?
They won't all be informed!

## Mutation and contracts

### Mutable objects can make simple contracts very complex

This is a fundamental issue with mutable data structures.
Multiple references to the same mutable object (also called **aliases** for the object) may mean that multiple places in your program --- possibly widely separated --- are relying on that object to remain consistent.

To put it in terms of specifications, contracts can't be enforced in just one place anymore, e.g. between the client of a class and the implementer of a class.
Contracts involving mutable objects now depend on the good behavior of everyone who has a reference to the mutable object.

As a symptom of this non-local contract phenomenon, consider the Java collections classes, which are normally documented with very clear contracts on the client and implementer of a class.
Try to find where it documents the crucial requirement on the client that we've just discovered - that you can't modify a collection while you're iterating over it.
Who takes responsibility for it?
[`Iterator`](java:java/util/Iterator)?
[`List`](java:java/util/List)?
[`Collection`](java:java/util/Collection)?
Can you find it?

The need to reason about global properties like this make it much harder to understand, and be confident in the correctness of, programs with mutable data structures.
We still have to do it --- for performance and convenience --- but we pay a big cost in bug safety for doing so. 

### Mutable objects reduce changeability

Mutable objects make the contracts between clients and implementers more complicated, and reduce the freedom of the client and implementer to change.
In other words, using *objects* that are allowed to change makes the *code* harder to change.
Here's an example to illustrate the point.

The crux of our example will be the specification for this method, which looks up a username in UBC’s database and returns the user's 9-digit identifier:

<div class="pull-margin">
```java
/**
 * @param username username of person to look up
 * @return the 9-digit UBC identifier for username.
 * @throws NoSuchUserException if nobody with username is in UBC’s database
 */
public static char[] getUBCId(String username) throws NoSuchUserException {        
    // ... look up username in UBC’s database and return the 9-digit ID
}
```
</div>

A reasonable specification.
Now suppose we have a client using this method to print out a user's identifier:

```java
char[] id = getUBCId("bitdiddle");
System.out.println(id);
```

**Now both the client and the implementor separately decide to make a change.**
The client is worried about the user's privacy, and decides to obscure the first 5 digits of the id:

```java
char[] id = getUBCId("bitdiddle");
for (int i = 0; i < 5; ++i) {
    id[i] = '*';
}
System.out.println(id);
```

The implementer is worried about the speed and load on the database, so the implementer introduces a cache that remembers usernames that have been looked up:

<div class="pull-margin">
```java
private static Map<String, char[]> cache = new HashMap<String, char[]>();

public static char[] getUBCId(String username) throws NoSuchUserException {        
    // see if it's in the cache already
    if (cache.containsKey(username)) {
        return cache.get(username);
    }
    
    // ... look up username in UBC’s database ...
    
    // store it in the cache for future lookups
    cache.put(username, id);
    return id;
}
```
</div>

These two changes have created a subtle bug.
When the client looks up `"bitdiddle"` and gets back a char array, now both the client and the implementer's cache are pointing to the *same* char array.
The array is aliased.
That means that the client's obscuring code is actually overwriting the identifier in the cache, so future calls to `getMidId("bitdiddle")` will not return the full 9-digit number, like "928432033", but instead the obscured version "*****2033".

**Sharing a mutable object complicates a contract**.
If this contract failure went to software engineering court, it would be contentious.
Who's to blame here?
Was the client obliged not to modify the object it got back?
Was the implementer obliged not to hold on to the object that it returned?

Here's one way we could have clarified the spec:

<pre>
public static char[] getUBCId(String username) throws NoSuchUserException 
  *requires*: nothing
  *effects*: returns an array containing the 9-digit UBC identifier of username,
             or throws NoSuchUserException if nobody with username is in UBC’s database. Caller may never modify the returned array.
</pre>

**This is a bad way to do it**.
The problem with this approach is that it means the contract has to be in force for the entire rest of the program.
It's a lifetime contract!
The other contracts we wrote were much narrower in scope; you could think about the precondition just before the call was made, and the postcondition just after, and you didn't have to reason about what would happen for the rest of time. 

Here's a spec with a similar problem:

<pre>
public static char[] getUBCId(String username) throws NoSuchUserException 
  *requires*: nothing
  *effects*: returns a new array containing the 9-digit UBC identifier of username,
             or throws NoSuchUserException if nobody with username is in UBC's
             database.
</pre>

**This doesn't entirely fix the problem either**.
This spec at least says that the array has to be fresh.
But does it keep the implementer from holding an alias to that new array?
Does it keep the implementer from changing that array or reusing it in the future for something else?

Here's a much better spec:

<pre class="pull-margin">
public static String getUBCId(String username) throws NoSuchUserException 
  *requires*: nothing
  *effects*: returns the 9-digit UBC identifier of username, or throws
             NoSuchUserException if nobody with username is in UBC's database.
</pre>

The immutable String return value provides a *guarantee* that the client and the implementer will never step on each other the way they could with char arrays.
It doesn't depend on a programmer reading the spec comment carefully.
String is *immutable*.
Not only that, but this approach (unlike the previous one) gives the implementer the freedom to introduce a cache --- a performance improvement.

## Useful immutable types

Since immutable types avoid so many pitfalls, let's enumerate some commonly-used immutable types in the Java API:

+ The primitive types and primitive wrappers are all immutable.
  If you need to compute with large numbers, [`BigInteger`](java:java/math/BigInteger) and [`BigDecimal`](java:java/math/BigDecimal) are immutable.

+ Don't use mutable `Date`s, use the appropriate immutable type from [`java.time`](java:java/time/package-summary.html) based on the granularity of timekeeping you need.

+ The usual implementations of Java's collections types --- `List`, `Set`, `Map` --- are all mutable: `ArrayList`, `HashMap`, etc.
  The [`Collections`](java:java/util/Collections) utility class has methods for obtaining *unmodifiable views* of these mutable collections:
  
  + `Collections.unmodifiableList`
  + `Collections.unmodifiableSet`
  + `Collections.unmodifiableMap`
  
  You can think of the unmodifiable view as a wrapper around the underlying list/set/map.
  A client who has a reference to the wrapper and tries to perform mutations --- `add`, `remove`, `put`, etc. --- will trigger an [`UnsupportedOperationException`](java:java/lang/UnsupportedOperationException).
  
  Before we pass a mutable collection to another part of our program, we can wrap it in an unmodifiable wrapper.
  We should be careful at that point to lose our reference to the mutable collection, lest we accidentally mutate it!
  Just as a mutable object behind a `final` reference can be mutated, the mutable collection inside an unmodifiable wrapper can still be modified, defeating the wrapper.

+ `Collections` also provides methods for obtaining immutable empty collections: `Collections.emptyList`, etc.
  Nothing's worse than discovering your *definitely very empty* list is suddenly *definitely not empty*.

## Summary

In this reading, we saw that mutability is useful for performance and convenience, but it also creates risks of bugs by requiring the code that uses the objects to be well-behaved on a global level, greatly complicating the reasoning and testing we have to do to be confident in its correctness.

Make sure you understand the difference between an immutable *object* (like a `String`) and an immutable *reference* (like a `final` variable).
Instance diagrams can help with this understanding.
Objects are values, represented by circles in a instance diagram, and an immutable one has a double border indicating that it never changes its value.
A reference is a pointer to an object, represented by an arrow in the instance diagram, and an immutable reference is an arrow with a double line, indicating that the arrow can't be moved to point to a different object.

The key design principle here is **immutability**: using immutable objects and immutable references as much as possible.
Let's review how immutability helps with the main goals of this course:

+ **Safe from bugs**.
  Immutable objects aren't susceptible to bugs caused by aliasing.
  Immutable references always point to the same object.
+ **Easy to understand**.
  Because an immutable object or reference always means the same thing, it's simpler for a reader of the code to reason about --- they don't have to trace through all the code to find all the places where the object or reference might be changed, because it can't be changed.
+ **Ready for change**.
  If an object or reference can't be changed at runtime, then code that depends on that object or reference won't have to be revised when the program changes.

