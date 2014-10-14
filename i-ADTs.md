---
layout: page
title: Rep Invariants and Abstraction Functions
---

#### Objectives of this Reading

This reading introduces two important ideas:

+ abstraction functions
+ representation invariants

In this reading, we study a more formal mathematical idea of what it means for a class to implement an ADT, via the notions of *abstraction functions* and *rep invariants*.  These mathematical notions are eminently practical in software design.  The abstraction function will give us a way to cleanly define the equality operation on an abstract data type, and the rep invariant will make it easier to catch bugs caused by a corrupted data structure.

## Rep Invariant and Abstraction Function

We now take a deeper look at the theory underlying abstract data types. This theory is not only elegant and interesting in its own right; it also has immediate practical application to the design and implementation of abstract types. If you understand the theory deeply, you'll be able to build better abstract types, and will be less likely to fall into subtle traps.

In thinking about an abstract type, it helps to consider the relationship between two spaces of values.

The space of representation values (or rep values for short) consists of the values of the actual implementation entities. In simple cases, an abstract type will be implemented as a single object, but more commonly a small network of objects is needed, so this value is actually often something rather complicated. For now, though, it will suffice to view it simply as a mathematical value.

The space of abstract values consists of the values that the type is designed to support. These are a figment of our imaginations. They're platonic entities that don't exist as described, but they are the way we want to view the elements of the abstract type, as clients of the type. For example, an abstract type for unbounded integers might have the mathematical integers as its abstract value space; the fact that it might be implemented as an array of primitive (bounded) integers, say, is not relevant to the user of the type.

Now of course the implementor of the abstract type must be interested in the representation values, since it is the implementor's job to achieve the illusion of the abstract value space using the rep value space.

Suppose, for example, that we choose to use a string to represent a `Set<Character>` (a set of characters).

```java
public class CharSet implements Set<Character> {
    private String s;
    ...
}
```

<img src="https://dl.dropboxusercontent.com/u/567187/EECE%20210/Images/ADTs-RIAF/charset-af-ri.png" alt="the abstract space and rep space of CharSet" width="300"></img>

Then the rep space *R* contains `String`s, and the abstract space *A* is mathematical sets of characters. We can show the two value spaces graphically, with an arc from a rep value to the abstract value it represents.  There are several things to note about this picture:

+ **Every abstract value is mapped to by some rep value**. The purpose of implementing the abstract type is to support operations on abstract values. Presumably, then, we will need to be able to create and manipulate all possible abstract values, and they must therefore be representable.
+ **Some abstract values are mapped to by more than one rep value**. This happens because the representation is not a tight encoding. There's more than one way to represent an unordered set of characters as a string.
+ **Not all rep values are mapped**. For example, the string "abbc" is not mapped. In this case, we have decided that the string should not contain duplicates. This will allow us to terminate the `remove()` method when we hit the first instance of a particular character, since we know there can be at most one.

In practice, we can only illustrate a few elements of the two spaces and their relationships; the graph as a whole is infinite. So we describe it by giving two things:

1\.  An *abstraction function* that maps rep values to the abstract values they represent: 

> AF : R &rarr; A

The arcs in the diagram show the abstraction function. In the terminology of functions, the properties we discussed above can be expressed by saying that the function is surjective (also called *onto*), not necessarily bijective (also called *one-to-one*), and often partial.

2\.  A *rep invariant* that maps rep values to booleans: 

> RI : R &rarr; boolean

For a rep value *r*, *RI(r)* is true if and only if *r* is mapped by *AF*. In other words, *RI* tells us whether a given rep value is well-formed. Alternatively, you can think of *RI* as a set: it's the subset of rep values on which *AF* is defined.

Both the rep invariant and the abstraction function should be documented in the code, right next to the declaration of the rep itself:

```java
public class CharSet_NoRepeatsRep implements Set<Character> {
    private String s;
    // Rep invariant:
    //    s contains no repeated characters
    // Abstraction Function:
    //   represents the set of characters found in s
    ...
}
```

A common confusion about abstraction functions and rep invariants is that they are determined by the choice of rep and abstract value spaces, or even by the abstract value space alone. If this were the case, they would be of little use, since they would be saying something redundant that's already available elsewhere.

The abstract value space alone doesn't determine *AF* or *RI*: there can be several representations for the same abstract type. A set of characters could equally be represented as a string, as above, or as a bit vector, with one bit for each possible character. Clearly we need two different abstraction functions to map these two different rep value spaces.

It's less obvious why the choice of both spaces doesn't determine *AF* and *RI*. The key point is that defining a type for the rep, and thus choosing the values for the space of rep values, does not determine which of the rep values will be deemed to be legal, and of those that are legal, how they will be interpreted. Rather than deciding, as we did above, that the strings have no duplicates, we could instead allow duplicates, but at the same time require that the characters be sorted, appearing in nondecreasing order. This would allow us to perform a binary search on the string and thus check membership in logarithmic rather than linear time. Same rep value space ——different rep invariant:

```java
public class CharSet_SortedRep implements Set<Character> {
    private String s;
    // Rep invariant:
    //    s[0] < s[1] < ... < s[s.length()-1]
    // Abstraction Function:
    //   represents the set of characters found in s
    ...
}
```

Even with the same type for the rep value space and the same rep invariant *RI*, we might still interpret the rep differently, with different abstraction functions *AF*. Suppose *RI* admits any string of characters. Then we could define AF, as above, to interpret the array's elements as the elements of the set. But there's no *a priori* reason to let the rep decide the interpretation. Perhaps we'll interpret consecutive pairs of characters as subranges, so that the string "acgg" represents the set *{a,b,c,g}*.

```java
public class CharSet_SortedRangeRep implements Set<Character> {
    private String s;
    // Rep invariant:
    //    s.length is even
    //    s[0] <= s[1] <= ... <= s[s.length()-1]
    // Abstraction Function:
    //   represents the union of the ranges
    //   {s[i]...s[i+1]} for each adjacent pair of characters 
    //   in s
    ...
}
```

The essential point is that designing an abstract type means **not only choosing the two spaces** -- the abstract value space for the specification and the rep value space for the implementation -- **but also deciding what rep values to use and how to interpret them**.

It's critically important to write down these assumptions in your code, as we've done above, so that future programmers (and your future self) are aware of what the representation actually means.  Why? What happens if different implementers disagree about the meaning of the rep?

### Example: Rational Numbers

Here's an example of an abstract data type for rational numbers.  Look closely at its rep invariant and abstraction function. 

```java
public class RatNum {
    private final int numer;
    private final int denom;

    // Rep invariant:
    //   denom > 0
    //   numer/denom is in reduced form

    // Abstraction Function:
    //   represents the rational number numer / denom

    /** Make a new Ratnum == n. */
    public RatNum(int n) {
        numer = n;
        denom = 1;
        checkRep();
    }

    /**
     * Make a new RatNum == (n / d).
     * @param n numerator
     * @param d denominator
     * @throws ArithmeticException if d == 0
     */
    public RatNum(int n, int d) throws ArithmeticException {
        // reduce ratio to lowest terms
        int g = gcd(n, d);
        n = n / g;
        d = d / g;

        // make denominator positive
        if (d < 0) {
            numer = -n;
            denom = -d;
        } else {
            numer = n;
            denom = d;
        }
        checkRep();
    }
}
```

<img src="https://dl.dropboxusercontent.com/u/567187/EECE%20210/Images/ADTs-RIAF/ratnum-af-ri.png" alt="the abstraction function and rep invariant of RatNum" width="500"></img>

Here is a picture of the abstraction function and rep invariant for this code. The *RI* requires that numerator/denominator pairs be in reduced form (i.e., lowest terms), so pairs like (2,4) and (18,12) above should be drawn as outside the *RI*.

It would be completely reasonable to design another implementation of this same ADT with a more permissive *RI*.  With such a change, some operations might become more expensive to perform, and others cheaper.

### Checking the Rep Invariant

The rep invariant isn't just a neat mathematical idea. If your implementation asserts the rep invariant at run time, then you can catch bugs early.  Here's a method for `RatNum` that tests its rep invariant:

```java
// Check that the rep invariant is true
// *** Warning: this does nothing unless you turn on assertion checking
// by passing -enableassertions to Java
private void checkRep() {
    assert denom > 0;
    assert gcd(numer, denom) == 1;
}
```

You should certainly call `checkRep()` to assert the rep invariant at the end of every operation that creates or mutates the rep -- in other words, creators, producers, and mutators.  Look back at the `RatNum` code above, and you'll see that it calls `checkRep()` at the end of both constructors.

Observer methods don't normally need to call `checkRep()`, but it's good defensive practice to do so anyway.  Why? Calling `checkRep()` in every method, including observers, means you'll be more likely to catch rep invariant violations caused by rep exposure.

Why is `checkRep()` private?  Who should be responsible for checking and enforcing a rep invariant -- clients, or the implementation itself?

### No Null Values in the Rep

Recall from the [reading on Specifications](../2-2-Specifications) that null values are troublesome and unsafe, so much so that we try to remove them from our programming entirely.

We extend that prohibition to the reps of abstract data types.  By default, in this course, the rep invariant implicitly includes `x != null` for every reference `x` in the rep that has object type (including references inside arrays or lists).  So if your rep is:

```java
class CharSet {
    String s;
}
```

then its rep invariant automatically includes `s != null`, and you don't need to state it in a rep invariant comment. (You can state it if that is more to your liking.)

When it's time to implement that rep invariant in a checkRep() method, however, you still must *implement* the `s != null` check, and make sure that your `checkRep()` correctly fails when `s` is `null`.  Often that check comes for free from Java, because checking other parts of your rep invariant will throw an exception if `s` is null.  For example, if your `checkRep()` looks like this: 

```java
private void checkRep() {
    assert s.length() % 2 == 0;
    ...
}
```

then you don't need `assert s!= null`, because the call to `s.length()` will fail just as effectively on a null reference.  But if `s` is not otherwise checked by your rep invariant, then assert `s != null` explicitly.


## Summary

+ The **Abstraction Function** maps a concrete representation to the abstract value it represents.
+ The **Rep Invariant** specifies legal values of the representation, and should be checked at runtime with `checkRep()`.

The topics of today's reading connect to our three properties of good software as follows:

+ **Safe from bugs.**  Stating the rep invariant explicitly, and checking it at runtime with `checkRep()`, catches misunderstandings and bugs earlier, rather than continuing on with a corrupt data structure. 

+ **Easy to understand.**  Rep invariants and abstraction functions explicate the meaning of a data type's representation, and how it relates to its abstraction.

+ **Ready for change.** Abstract data types separate the abstraction from the concrete representation, which makes it possible to change the representation without having to change client code. 
