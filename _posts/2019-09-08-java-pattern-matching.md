---
layout: post
title: Sum Types and Pattern Matching in Java
---

# Sum Types and Pattern Matching in Java

So you're a Java programmer and while hanging out with your hip dev friends you've
heard of sum types (aka union types or variants). Now you're wondering if they make
sense for you and how would you use them in Java. Actually you probably already know
them under a different form. In this post I'll explain what they are, how they're
usefull, and how to use them in Java.

## Product and Sum Types

Type theory says that you can model any real world data using two concepts: product
and sum types. As an example, think of the following way of representing a person:

```java
class Person {
  String name;
  int age;
}
```

`Person` is a product type between the type `String` and the type `int`, meaning that
that any string and number are possible (of course within some limits for the age).
It's called a product type from [cartesian product](https://en.wikipedia.org/wiki/Cartesian_product), i.e., the number of values this type can take is the multiplication between the number of possible values for string and integer.

As another example, think of a binary tree that has values only in its leaves. There are
two types of nodes here:

* internal nodes which have two children nodes (`left` and `right`)
* leaf nodes that have a `value`

One way to represent the `Node` type is this:

```java
class Node<T> {
  NodeType nodeType;
  Node left;
  Node right;
  T value;
}

enum NodeType { Internal, Leaf }
```

The logic using this type would have to first check `NodeType` in order to know
whether it should expect `left` and `right` to be valid (non-null) or `value` to be
valid.

Now you might be thinking: "Yep, I've been doing this for ages and everything turned
out hunky-dory". But your new colleague Jimmy or your future self might be of a
different opinion:

* By looking at this type he doesn't know which data he should expect. He thinks all
  the combinations are valid. He has to read your other code to understand
  your intentions.
* Even if he gets the drill, he's still a human and makes mistakes by the
  accessing the
  `value` data even if he's checked this to be an internal node. The compiler smiles
  hapilly though instead of helping him.

Can we do better than this? You guessed it: `Node` is a sum type between the type
`InternalNode` and the
type `LeafNode`. That is, it can be either one or the other, and depending on what it
is, you can expect different data.

Many languages have sum types as first class citizens. However, in Java you
can still achieve the same by using inheritance. So for our example:

```java
interface Node<T> {
    class InternalNode<T> implements Node<T> {
        Node<T> left;
        Node<T> right;
    }

    class LeafNode<T> implements Node<T> {
        T value;
    }
}
```

This representation has several advantages:

* by just looking at the type you can see what data pertains to each variant
* you are forced to narrow down the type `Node` to one of its *variants*,
  `InternalNode` or `LeafNode`, before accessing the data. Otherwise, the
  compiler will bark.

## Narrowing Down a Sum Type, aka Pattern Matching

Let's say we want to write some algorithms using the `Node` type:

1. compute the depth of the tree
2. extract all the values from the tree
3. print down the tree

All these need to narrow down the `Node` type. How do we do that? The recommended
way is to use polymorphism in one
way or another, or in other words: "thou shalt not cast". If you are writing more than
one algorithm, you'd probably [use](https://www.baeldung.com/java-visitor-pattern)
the [visitor pattern](https://en.wikipedia.org/wiki/Visitor_pattern)

I risk getting burned at the stake by the OOP zealots, but I think using `instanceof`
and casting is not such a bad alternative. Actually many modern languages
provide a concept equivalent to this. This concept is called pattern matching (well,
pattern matching is a bit broader concept, but that's for another post).

This is how we'd implement computing the tree depth:

```java
static <V> int treeDepth(final Node<V> node) {
    if (node instanceof InternalNode) {
        InternalNode<V> internalNode = (InternalNode<V>) node;
        return 1 + max(treeDepth(internalNode.left), treeDepth(internalNode.right));
    } else if (node instanceof LeafNode) {
       return 1;
    } else {
        throw new RuntimeException("node type not known: " + node);
    }
}
```

I might be spared for doing this, because Java is also introducing [pattern matching for
instanceof](http://openjdk.java.net/jeps/305). I.e., if you use `instanceof` in a
conditional, the sum type gets automatically casted to that variant, so no manual casting
needed anymore.

## Hiding the instanceof

All those `instanceof`s and casts are pretty ugly and poluting. Can we abstract them
away somehow. Sure thing! We could make a method to which we provide:

* an object of the sum type to match
* a lambda for each variant of the sum type. It will be called if the object matches that
  variant.
* a default lambda to be called if the object doesn't match any of the provided variants

This is how we'd implement `treeDepth` with such a construct:

```java
import static java.lang.Integer.max;
import static patternmatch.patternmatch.PatternMatch.Case.exprCase;
import static patternmatch.patternmatch.PatternMatch.Default.exprDefault;
import static patternmatch.patternmatch.PatternMatch.match;

class Depth {
    private static <V> int treeDepth(final Node<V> node) {
        return match(node,
                exprCase(InternalNode.class, (InternalNode n) -> 1 + max(treeDepth(n.left), treeDepth(n.right))),
                exprCase(LeafNode.class, (LeafNode n) -> 1),
                exprDefault(() -> { throw new RuntimeException("node not known: " + node); }));
    }
}
```

As you can see, `exprCase` receives the variant to be matched (`InternalNode` or `Leaf`)
and a lambda to be called when the variant matches. `exprDefault` gets a lambda to be
called if the object doesn't match `InternalNode` or `LeafNode`. In some languages
it is possible to *seal* the sum type to certain variants, but in Java it's impossible
to stop someone from adding more variants. So adding a default is always a good idea,
to detect that in the future.

So how can we implement the `match` method? I have a poor man's implementation for it
on [github](https://github.com/sutiialex/java_pattern_matching/blob/master/src/patternmatch/patternmatch/PatternMatch.java). It's far from fancy, but it gets the job done. It basically
provides overloaded versions of `match` for 1, 2, ..., 7 variants. It's trivial
to add more if you need to.

Here's a glimpse of how it looks like:

```java
import java.util.Optional;
import java.util.function.Consumer;
import java.util.function.Function;
import java.util.function.Supplier;

public class PatternMatch {
    public static <T, V1 extends T, R> R match(final T t, final Case<V1, R> c1, final Default<R> defaultF) {
        return oMatch(t, c1).orElseGet(defaultF.f);
    }

    public static <T, V1 extends T, V2 extends T, R> R match(final T t, final Case<V1, R> c1, final Case<V2, R> c2, final Default<R> defaultF) {
        return oMatch(t, c1).orElseGet(() -> match(t, c2, defaultF));
    }

    ...

    private static <T, V extends T, R> Optional<R> oMatch(final T t, final Case<V, R> c) {
        return c.type.equals(t.getClass()) ? Optional.of(c.f.apply(c.type.cast(t))) : Optional.empty();
    }

    public static class Case<V, R> {
        final Class<V> type;
        final Function<V, R> f;

        private Case(final Class<V> t, final Function<V, R> f) {
            this.type = t;
            this.f = f;
        }

        public static <V, R> Case<V, R> exprCase(final Class<V> t, final Function<V, R> f) {
            return new Case<>(t, f);
        }
        ...
    }

    public static class Default<R> {
        final Supplier<R> f;

        private Default(final Supplier<R> f) {
            this.f = f;
        }

        public static <R> Default<R> exprDefault(final Supplier<R> f) {
            return new Default<>(f);
        }

        ...
    }
}
```

Basically the method `oMatch` gets the an object and a `Case` and if the object
matches the variant from the case, it runs the lambda from the case and returns the result
of the lambda wrapped in an `Optional`. If not, it returns an empty `Optional`.

The `match(o, case1, case2, ..., casen, defaultF)`, calls `oMatch(o, case1)`.
If that matches, the result is returned, otherwise
`match(o, case2, ..., casen, defaultF)` is called. When `match(o, casen, default)` is reached and `oMatch(o, casen)` doesn't match, we call `defaultF`.

As you've probably guessed, `exprCase` and `exprDefault` are useful when you want your
`match()` to return a value, i.e., it's an expression. If however you want to just
pass lambdas that return `void` (e.g., if you want to print the sum type), you can use
`stmtCase` and `stmtDefault` from [the same class](https://github.com/sutiialex/java_pattern_matching/blob/master/src/patternmatch/patternmatch/PatternMatch.java).

For a better example of how to use `match()` check [Github](https://github.com/sutiialex/java_pattern_matching/blob/master/src/patternmatch/Main.java).

## Wrapping up

In this post you've learned about sum types and how to implement them in Java.
Also we went through implementing a poor man's version of pattern matching in Java.
I hope that helps and see you next time.
