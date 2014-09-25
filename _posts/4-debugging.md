---
layout: page
title: Debugging
---

### Main Course Objectives
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


#### Objectives

The topic of _this reading_ is debugging:

0. how to avoid it,
0. how to keep it easy,
0. how to do it systematically when you have to.

<!-- Scope minimization section should talk about parameter passing rather than using global variables -->
<!-- have a section about what static means, for methods and variables -->


## Instance Diagrams

It will be useful for us to draw pictures of what's happening at runtime, in order to understand subtle questions.  Instance diagrams represent the internal state of an object or even a program at runtime -- its stack (methods in progress and their local variables) and its heap (objects that currently exist).

Why should we use instance diagrams?

+ To talk to each other through pictures (in class and in team meetings)
+ To illustrate concepts like primitive types vs. object types, immutable values vs. immutable references, pointer aliasing, stack vs. heap, abstractions vs. concrete representations.
+ To help explain your design for your team project (with each other and with your TA)
+ To pave the way for richer design notations in subsequent courses. 

Although the diagrams in this course use examples from Java, the notation can be applied to any modern programming language, e.g., Python, Javascript, C++, Ruby.

### Primitive values

<div class="panel panel-figure pull-right pull-margin">
<img src="https://dl.dropboxusercontent.com/u/567187/EECE%20210/Images/Debugging/primitives.png" alt="primitive values in an instance diagram" width="300"></img>
</div>

Primitive values are represented by bare constants.  The incoming arrow is a reference to the value from a variable or an object field.

### Object values

<div class="panel panel-figure pull-right pull-margin">
<img src="https://dl.dropboxusercontent.com/u/567187/EECE%20210/Images/Debugging/objects.png" alt="object values in an instance diagram" width="400"></img>
</div>

An object value is a circle labeled by its type.  When we want to show more detail, we write field names inside it, with arrows pointing out to their values. For still more detail, the fields can include their declared types. Some people prefer to write `x:int` instead of `int x`, but both are fine.  

### Mutating Values vs. Reassigning Variables

Instance diagrams give us a way to visualize the distinction between changing a variable and changing a value.  When you assign to a variable or a field, you're changing where the variable's arrow points.  You can point it to a different value.

When you assign to the contents of a mutable value -- such as an array or list -- you're changing references inside that value.

Immutability (immunity from change) is a major design principle in this course. Immutable types are types whose values can never change once they have been created.

Java also gives us immutable references: variables that are assigned once and never reassigned.To make a reference immutable, declare it with the keyword `final`:
```java
final int n = 5;
```

<div class="panel panel-figure pull-right pull-margin">
<img src="https://dl.dropboxusercontent.com/u/567187/EECE%20210/Images/Debugging/final-reference.png" alt="final reference is a double arrow" width="200"></img>
</div>

If the Java compiler isn't convinced that your `final` variable will only be assigned once at runtime, then it will produce a compiler error.  So `final` gives you static checking for immutable references.

In an instance diagram, an immutable reference (`final`) is denoted by a double arrow.  Here's an object whose `id` never changes (it can't be reassigned to a different number), but whose `age` can change.

### Immutable Objects vs. Mutable Objects

<div class="panel panel-figure pull-right pull-margin">
<img src="https://dl.dropboxusercontent.com/u/567187/EECE%20210/Images/Debugging/reassignment.png" alt="reassigning a variable" width="200"></img>
</div>

String is *immutable*: once created, a String object always has the same value.  To add something to the end of a String, you have to create a new String object:
```java
String s = "a";
s = s.concat("b");    /// s+="b" and s=s+"b" mean the same thing as this call
```

<div class="clearfix"></div>

<div class="panel panel-figure pull-right pull-margin">
<img src="https://dl.dropboxusercontent.com/u/567187/EECE%20210/Images/Debugging/integer.png" alt="Integer is immutable" width="200"></img>
</div>
Immutable objects (intended by their designer to always represent the same value) are denoted by a double border.  For example, here's an Integer object, the result of `new Integer(7)`.  By design, this Integer object can never change value during its lifetime.  There is no method of Integer that will change it to a different integer value.


<div class="clearfix"></div>

<div class="panel panel-figure pull-right pull-margin">
<img src="https://dl.dropboxusercontent.com/u/567187/EECE%20210/Images/Debugging/mutation.png" alt="mutating an object" width="200"></img>
</div>
By contrast, StringBuilder (another built-in Java class) is a *mutable* object that represents a string of characters.  It has methods that change the value of the object, rather than just returning new values:
```java
StringBuilder sb = new StringBuilder("a");
sb.append("b");
```

StringBuilder has other methods as well, for deleting parts of the string, inserting in the middle, or changing individual characters.

So what?  In both cases, you end up with `s` and `sb` referring to the string of characters "abcdef".  The difference between mutability and immutability doesn't matter much when there's only one reference to the object. But there are big differences in how they behave when there are *other* references to the object.  For example, when another variable `t` points to the same String object as `s`, and another variable `tb` points to the same StringBuilder as `sb`, then the differences between the immutable and mutable objects become more evident:
<div class="panel panel-figure pull-right pull-margin">
<img src="https://dl.dropboxusercontent.com/u/567187/EECE%20210/Images/Debugging/string-vs-stringbuilder.png" alt="different behavior of String and StringBuilder" width="500"></img>
</div>
```java
  String t = s;
  t = t + "c";

  StringBuilder tb = sb;
  tb.append("c");
```

Why do we need the mutable StringBuilder in programming?  A common use for it is to concatenate a large number of strings together, like this:
```java
String s = "";
for (int i = 0; i &lt; n; ++i) {
    s = s + n;
}
```

Using immutable Strings, this makes a lot of temporary copies -- the first number of the string ("0") is actually copied *n* times in the course of building up the final string, the second number is copied *n-1* times, and so on.  It actually costs *O(n^2)* time just to do all that copying, even though we only concatenated *n* elements.

StringBuilder is designed to minimize this copying.  It uses a simple but clever internal data structure to avoid doing any copying at all until the very end, when you ask for the final String with a toString() call:
```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i &lt; n; ++i) {
  sb.append(String.valueOf(n));
}
String s = sb.toString();
```

Getting good performance is one reason why we use mutable objects.  Another is convenient sharing: two parts of your program can communicate more conveniently by sharing a common mutable data structure.

But the convenience of mutable data comes with big risks.  Mutability makes it harder to understand what your program is doing, and much harder to enforce contracts.  We'll see an example of that later in the lecture.

### Arrays and Lists

<div class="panel panel-figure pull-right pull-margin">
<img src="https://dl.dropboxusercontent.com/u/567187/EECE%20210/Images/Debugging/arrays-and-lists.png" alt="Arrays and Lists" width="500"></img>
</div>


Like other object values, arrays and lists are labeled with their type. In lieu of field names, we label the outgoing arrows with indexes 0, 1, 2, ...  When the sequence of elements is obvious, we may omit the index labels.

Both the array and List objects are mutable, as indicated by the single-line border and the single-line arrows that can be reassigned.

## First Defense: Make Bugs Impossible

Now we turn to the main topic of this reading: debugging.

The best defense against bugs is to make them impossible by design.

One way that we've already talked about is **static checking**.
Static checking eliminates many bugs by catching them at compile time.

We also saw some examples of **dynamic checking** in earlier class meetings.
For example, Java makes array overflow bugs impossible by catching them dynamically.
If you try to use an index outside the bounds of an array or a List, then Java automatically produces an error.
Older languages like C and C++ silently allow the bad access, which leads to bugs and [security vulnerabilities](http://en.wikipedia.org/wiki/Buffer_overflow).

**Immutability** (immunity from change) is another design principle that prevents bugs. 
An *immutable type* is a type whose values can never change once they have been created. 
String is an immutable type.
There are no methods that you can call on a String that will change the sequence of characters that it represents.
Strings can be passed around and shared without fear that they will be modified.  How is that different from a character array, `char[]`?

As we saw above, Java also gives us *immutable references*: variables declared with the keyword `final`, which can be assigned once but never reassigned.  It's good practice to use `final` for declaring the parameters of a method and as many local variables as possible.  Like the type of the variable, these declarations are important documentation, useful to the reader of the code and statically checked by the compiler. 

Consider this example:
```java
final char[] vowels = new char[] { 'a', 'e', 'i', 'o', 'u' };
```
The `vowels` variable is declared final, but is it really unchanging?  Which of the following statements will be illegal (caught statically by the compiler), and which will be allowed?
```java
vowels = new char[] { 'x', 'y', 'z' }; 
vowels[0] = 'z';
```

You'll find the answers in the exercise.  Be careful about what `final` means!  It only makes the *reference* immutable, not necessarily the *object* that the reference points to.  

## Second Defense: Localize Bugs

If we can't prevent bugs, we can try to localize them to a small part of the program, so that we don't have to look too hard to find the cause of a bug.
When localized to a single method or small module, bugs may be found simply by studying the program text.

We already talked about **fail fast**: the earlier a problem is observed (the closer to its cause), the easier it is to fix.

Let's begin with a simple example:
```java
/**
 * @param x  requires x >= 0
 * @return approximation to square root of x
 */
public double sqrt(double x) { ... }
```

Now suppose somebody calls `sqrt` with a negative argument.
What's the best behavior for `sqrt`?
Since the caller has failed to satisfy the requirement that `x` should be nonnegative, `sqrt` is no longer bound by the terms of its contract, so it is technically free to do whatever it wants: return an arbitrary value, or enter an infinite loop, or melt down the CPU.
Since the bad call indicates a bug in the caller, however, the most useful behavior would point out the bug as early as possible.
We do this by inserting a runtime assertion that tests the precondition.
Here is one way we might write the assertion:
```java
/**
 * @param x  requires x >= 0
 * @return approximation to square root of x
 */
public double sqrt(double x) { 
    if (! (x >= 0)) throw new AssertionError();
    ...
}
```

When the precondition is not satisfied, this code terminates the program by throwing an AssertionError exception.
The effects of the caller's bug are prevented from propagating.

Checking preconditions is an example of **defensive programming**.
Real programs are rarely bug-free.
Defensive programming offers a way to mitigate the effects of bugs even if you don't know where they are.

### Assertions

It is common practice to define a procedure for these kinds of defensive checks, usually called `assert`:
```java
assert (x >= 0);
```
This approach abstracts away from what exactly happens when the assertion fails.
The failed assert might exit; it might record an event in a log file; it might email a report to a maintainer.

Assertions have the added benefit of documenting an assumption about the state of the program at that point.
To somebody reading your code, `assert(x>=0)` says "at this point, it should always be true that x >= 0."
Unlike a comment, however, an assertion is executable code that enforces the assumption at runtime.

In Java, runtime assertions are a built-in feature of the language.
The simplest form of the assert statement takes a boolean expression, exactly as shown above, and throws AssertionError if the boolean expression evaluates to false:
```java
assert x >= 0;
```

An assert statement may also include a description expression, which is usually a string, but may also be a primitive type or a reference to an object.
The description is printed in an error message when the assertion fails, so it can be used to provide additional details to the programmer about the cause of the failure.
The description follows the asserted expression, separated by a colon. For example:
```java
assert (x >= 0) : "x is " + x;
```

If x == -1, then this assertion fails with the error message

> x is -1

along with a stack trace that tells you where the assert statement was found in your code and the sequence of calls that brought the program to that point.
This information is often enough to get started in finding the bug.

<div style="color:darkred">A serious problem with Java assertions is that assertions are *off by default*.</div>

If you just run your program as usual, none of your assertions will be checked!
Java's designers did this because checking assertions can sometimes be costly to performance. 
So you have to enable assertions explicity by passing -ea (which stands for *enable assertions*) to the Java virtual machine:

> java -ea MyClass

In Eclipse, you enable assertions by going to Run >> Run Configurations >> Arguments, and putting -ea in the VM arguments box.

It's a good idea to have assertions turned on when you're running JUnit tests.
You can test that assertions are enabled using the following test case: 
```java
@Test(expected=AssertionError.class)
public void testAssertionsEnabled() {
    assert false;
}
```

For most applications, however, assertions are *not* expensive compared to the rest of the code, and the benefit they provide in bug-checking is worth that small cost in peformance.
So to make assertions that are *always* turned on, you can just use the `assertTrue` method in JUnit.
Since it's a static method, you don't have to be writing a unit test to use `assertTrue`.
Just call `Assert.assertTrue()`:
```java
Assert.assertTrue(x >= 0);
```


### What to Assert

Here are some things you should assert:

**Method argument requirements**, like we saw for `sqrt`.

**Method return value requirements.**  This kind of assertion is sometimes called a *self check*. For example, the sqrt method might square its result to check whether it is reasonably close to x:
```java
public double sqrt(double x) {
    assert x >= 0;
    double r;
    ... // compute result r
    assert Math.abs(r*r - x) &lt; .0001;
    return r;
}
```

**Covering all cases.** If a conditional statement or switch does not cover all the possible cases, it is good practice to use an assertion to block the illegal cases:
```java
switch (vowel) {
  case 'a':
  case 'e':
  case 'i':
  case 'o':
  case 'u': return "A";
  default: Assert.fail();
}
```
The assertion in the default clause has the effect of asserting that `vowel` must be one of the five vowel letters.

When should you write runtime assertions?
As you write the code, not after the fact.
When you're writing the code, you have the invariants in mind.
If you postpone writing assertions, you're less likely to do it, and you're liable to omit some important invariants.

### What Not to Assert

Runtime assertions are not free.
They can clutter the code, so they must be used judiciously.
Avoid trivial assertions, just as you would avoid uninformative comments.
For example:
```java
// don't do this:
x = y + 1;
assert x == y+1;
```
This assertion doesn't find bugs in your code.
It finds bugs in the compiler or Java virtual machine, which are components that you should trust until you have good reason to doubt them.
If an assertion is obvious from its local context, leave it out.

Never use assertions to test conditions that are external to your program, such as the existence of files, the availability of the network, or the correctness of input given by the user.
Assertions test the internal state of your program to ensure that it is within the bounds of its specification.
When an assertion fails, it indicates that the program has run off the rails in some sense, into a state in which it was not designed to function properly.
Assertion failures therefore indicate bugs.
External failures are not bugs, and there is no change you can make to your program in advance that will prevent them from happening.
External failures should be handled using exceptions instead.

Many assertion mechanisms are designed so that assertions are executed only during testing and debugging, and turned off when the program is released to users.
Java's assert statement behaves this way.
The advantage of this approach is that you can write very expensive assertions that would otherwise seriously degrade the performance of your program.
For example, a procedure that searches an array using binary search has a requirement that the array be sorted.
Asserting this requirement requires scanning through the entire array, however, turning an operation that should run in logarithmic time into one that takes linear time.
You should be willing (eager!) to pay this cost during testing, since it makes debugging much easier, but not after the program is released to users.

However, disabling assertions in release has a serious disadvantage.
With assertions disabled, a program has far less error checking when it needs it most.
Novice programmers are usually much more concerned about the performance impact of assertions than they should be. 
Most assertions are cheap, so they should not be disabled in the official release.

Since assertions may be disabled, the correctness of your program should never depend on whether or not the assertion expressions are executed.
In particular, asserted expressions should not have *side-effects*.
For example, if you want to assert that an element removed from a list was actually found in the list, don't write it like this:
```java
// don't do this:
assert list.remove(x);
```
If assertions are disabled, the entire expression is skipped, and x is never removed from the list. Write it like this instead:
```java
boolean found = list.remove(x);
assert found;
```

### Incremental Development

A great way to localize bugs to a tiny part of the program is incremental development.
Build only a bit of your program at a time, and test that bit thoroughly before you move on.
That way, when you discover a bug, it's more likely to be in the part that you just wrote, rather than anywhere in a huge pile of code.

Our class on testing talked about two techniques that help with this:

+ Unit testing: when you test a module in isolation, you can be confident that any bug you find is in that unit -- or maybe in the test cases themselves.
+ Regression testing: when you're adding a new feature to a big system, run the regression test suite as often as possible.  If a test fails, the bug is probably in the code you just changed.

### Modularity &amp; Encapsulation

You can also localize bugs by better software design.

**Modularity.**
Modularity means dividing up a system into components, or modules, each of which can be designed, implemented, tested, reasoned about, and reused separately from the rest of the system.
The opposite of a modular system is a monolithic system -- big and with all of its pieces tangled up and dependent on each other.

A program consisting of a single, very long main() function is monolithic -- harder to understand, and harder to isolate bugs in.
By contrast, a program broken up into small functions and classes is more modular.

**Encapsulation.**
Encapsulation means building walls around a module (a hard shell or capsule) so that the module is responsible for its own internal behavior, and bugs in other parts of the system can't damage its integrity.
We'll talk more about encapsulation using public and private modifiers when we get to abstract data types, but one important kind of encapsulation that is relevant to the kinds of methods we're writing now is **variable scope**.
The *scope* of a variable is the portion of the program text over which that variable is defined, in the sense that expressions and statements can refer to the variable.
Keeping variable scopes as small as possible makes it much easier to reason about where a bug might be in the program.
For example, suppose you have a loop like this:
```java
for (i = 0; i &lt; 100; ++i) {
    ...
    doSomeThings();
    ...
}
```
...and you've discovered that this loop keeps running forever -- `i` never reaches 100.  Somewhere, somebody is changing `i`.
But where?
If `i` is declared as a global variable like this:
```java
public static int i;
...
for (i =0; i &lt; 100; ++i) {
    ...
    doSomeThings();
    ...
}
```
...then its scope is the entire program.
It might be changed anywhere in your program: by `doSomeThings()`, by some other method that `doSomeThings()` calls, by a concurrent thread running some completely different code.
But if `i` is instead declared as a local variable with a narrow scope, like this:
```java
for (int i = 0; i &lt; 100; ++i) {
    ...
    doSomeThings();
    ...
}
```
...then the only place whjere `i` can be changed is within the for statement -- in fact, only in the ... parts that we've omitted.
You don't even have to consider `doSomeThings()`, because `doSomeThings()` doesn't have access to this local variable.

**Minimizing the scope of variables** is a powerful practice for bug localization.
Here are a few rules that are good for Java:

+ **Always declare a loop variable in the for-loop initializer.**  So rather than declaring it before the loop:
```java
int i;
for (i = 0; i &lt; 100; ++i) {
```
which makes the scope of the variable the entire rest of the outer curly-brace block containing this code, you should do this:
```java
for (int i = 0; i &lt; 100; ++i) {
```
which makes the scope of `i` limited just to the for loop.

+ **Declare a variable only when you first need it, and in the innermost curly-brace block that you can.**
Variable scopes in Java are curly-brace blocks, so put your variable declaration in the innermost one that contains all the expressions that need to use the variable.
Don't declare all your variables at the start of the function -- it makes their scopes unnecessarily large.
But note that in languages without static type declarations, like Python and Javascript, the scope of a variable is normally the entire function anyway, so you can't restrict the scope of a variable with curly braces, alas.

+ **Avoid global variables.**
Very bad idea, especially as programs get large.

## Last Ditch Defense: Debug Systematically

Sometimes you have no choice but to debug, however -- particularly when the bug is found only when you plug the whole system together, or reported by a user after the system is deployed, in which case it may be hard to localize it to a particular module.
For those situations, we can suggest a systematic strategy for more effective debugging.

### Reproduce the Bug

Start by finding a small, repeatable test case that produces the failure.
If the bug was found by regression testing, then you're in luck; you already have a failing test case in your test suite.
If the bug was reported by a user, it may take some effort to reproduce the bug.
For graphical user interfaces and multithreaded programs, a bug may be hard to reproduce consistently if it depends on timing of events or thread execution.

Nevertheless, any effort you put into making the test case small and repeatable will pay off, because you'll have to run it over and over while you search for the bug and develop a fix for it.
Furthermore, after you've successfully fixed the bug, you'll want to add the test case to your regression test suite, so that the bug never crops up again.
Once you have a test case for the bug, making this test work becomes your goal.

### Understand the Location and Cause of the Bug

To localize the bug and its cause, you can use the scientific method:

0. **Study the data.**
Look at the test input that causes the bug, and the incorrect results, failed assertions, and stack traces that result from it.

0. **Hypothesize.**
Propose a hypothesis, consistent with all the data, about where the bug might be, or where it *cannot* be.
It's good to make this hypothesis general at first.
Here's an example.
You're developing a web browser, and a user has found that displaying a certain web page causes the wrong text to appear on the screen.
You might hypothesize that the bug is not in the networking code that fetches the page from the server, but in one of the modules that parses the web page or displays it.

0. **Experiment.**
Devise an experiment that tests your hypothesis.
The experiment might be a different test case.
In our web browser example, you might test your hypothesis by downloading the page to disk and loading it from a disk file instead of over the network.
Another experiment inserts probes in the running program -- print statements, assertions, or debugger breakpoints.
It's tempting to try to insert *fixes* to the hypothesized bug, instead of mere probes.
This is almost always the wrong thing to do, because your fixes may just mask the true bug.
For example, if you're getting an ArrayOutOfBoundsException, try to understand what's going on first.  Don't just add code to avoid the exception without fixing the real problem.

0. **Repeat.**
Add the data you collected from your experiment to what you knew before, and make a fresh hypothesis.

**Bug localization by binary search.**
Debugging is a search process, and you can sometimes use binary search to speed up the process.
For example, in a web browser, the web page might flow through four modules before being displayed on the screen.
To do a binary search, you would divide this workflow in half, guessing that the bug is found somewhere in the first two modules, and insert probes (like breakpoints, print statements, or assertions) after the second module to check its results.
From the results of that experiment, you would further divide in half.

**Priotize your hypotheses.**
When making your hypothesis, you may want to keep in mind that different parts of the system have different likelihoods of failure.
For example, old, well-tested code is probably more trustworthy than recently-added code.
Java library code is probably more trustworthy than yours.
The Java compiler and runtime, operating system platform, and hardware are increasingly more trustworthy, because they are more tried and tested.
You should trust these lower levels until you've found good reason not to.

**Make sure your source code and object code are up to date.**
Pull the latest version from the repository, and delete all your .class files and recompile everything (in Eclipse, this is done by Project / Clean).

**Swap components.**
If you have another implementation of a module that satisfies the same interface, and you suspect the module, you may try swapping in the alternative.
For example, if you suspected java.util.ArrayList, you could swap in java.util.LinkedList instead.
If you suspect the binarySearch() method, then substitute a simpler linearSearch() instead.
If you suspect the Java runtime, run with a different version of Java.
If you suspect the operating system, run your program on a different OS.
If you suspect the hardware, run on a different machine.
You can waste a lot of time swapping unfailing components, however, so don't do this unless you have good reason to suspect a component.

**Get help.**
It often helps to explain your problem to someone else, even if the person you're talking to has no idea what you're talking about.
Teaching assistants and fellow students usually do know what you're talking about, so they're even better.

**Sleep on it.**
If you're too tired, you won't be an effective debugger.
Trade latency for efficiency.

### Fix the Bug

Once you've found the bug and understand its cause, the third step is to devise a fix for it. Avoid the temptation to slap a patch on it and move on. Ask yourself whether the bug was a coding error, like a misspelled variable or interchanged method parameters, or a design error, like an underspecified or insufficient interface. Design errors may suggest that you step back and revisit your design, or at the very least consider all the other clients of the failing interface to see if they suffer from the bug too.

Think also whether the bug has any relatives. If I just found a divide-by-zero error here, did I do that anywhere else in the code?  Try to make the code safe from future bugs like this.
Also consider what effects your fix will have. Will it break any other code?

Finally, after you have applied your fix, add the bug's test case to your regression test suite, and run all the tests to assure yourself that (a) the bug is fixed, and (b) no new bugs have been introduced.

### Summary

In this reading, we looked at instance diagrams to understand the difference between assignment and mutation.  In instance diagrams,
+ objects are represented by circles with a type and fields inside them
+ immutable objects have a double border
+ a variable or field reference is represented by an arrow
+ an immutable reference is an double arrow

We also looked at some ways to deal with bugs:

+ avoid debugging
    + make bugs impossible with techniques like static typing, automatic dynamic checking, and immutable types and references
+ keep bugs confined 
    + failing fast with assertions keeps a bug's effects from spreading 
    + incremental development and unit testing confine bugs to your recent code
    + scope minimization reduces the amount of the program you have to search
+ debug systematically
    + reproduce the bug as a test case, and put it in your regression suite
    + find the bug using the scientific method
    + fix the bug thoughtfully, not slapdash

Thinking about our three main measures of code quality:

+ **Safe from bugs.**  We're trying to prevent them and get rid of them.
+ **Easy to understand.** Techniques like static typing, final declarations, and assertions are additional documentation of the assumptions in your code.  Variable scope minimization makes it easier for a reader to understand how the variable is used, because there's less code to look at.
+ **Ready for change.**  Assertions and static typing document the assumptions in an automatically-checkable way, so that when a future programmer changes the code, accidental violations of those assumptions are detected.
