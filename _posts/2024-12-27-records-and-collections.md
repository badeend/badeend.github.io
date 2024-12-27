---
tags: DDD, Value-Objects, C#, Badeend.ValueCollections
---

# The missing piece of C# records: collections

> This post showcases my [ValueCollections](https://badeend.github.io/ValueCollections/) C# library.

---

There is a general trend to refactor to immutability. The simplest reason for this is because "things that change" are harder to reason about than "things that don't change".

A fundamental building block in Domain-Driven Design (DDD) is the Value Object. Value objects are immutable data structures without identity. They are compared based on their contents (value equality), not on their memory location (reference equality).

### Current state of C#

With the introduction of C# 9.0 `record`s, it's easier than ever to create value objects. For example, a `Point` value object can be defined as:

```cs
public record Point(float X, float Y);
```

Aside from being extremely consise, this also automatically provides immutability and value equality. The following test case passes just fine:

```cs
[Fact]
public void PointEquality()
{
    // Two dinstinct instances with the same contents:
    var a = new Point(1, 2);
    var b = new Point(1, 2);

    // a.X = 42; // Won't compile. Point is immutable. Yay!

    Assert.Equal(a, b); // Succeeds. Yay!
}
```

### The problem: collections

So far, so good. But now we want to create a `Polygon` value object that consists of a collection of points. Just like `Point`, a `Polygon` is only defined by its contents and has no intrinsic identity. How should we go about that?

The initial reaction could be to: "Just use a record again!":

```cs
public record Polygon(List<Point> Points); // Not what we're after.
```

But, alas, this has some problems:
- `List<T>` is mutable. We want our `Polygon` to be immutable.
- `List<T>` doesn't provide value equality. We want our `Polygon` to be compared based on the contents of its `Points`.

At the time of writing, .NET has no built-in collection type that provides the properties we're looking for:
- `IReadOnlyList<T>` only promises that _our_ reference to it is read-only, but it doesn't guarantee that the contents of the list are immutable. Some other part of the codebase could still have a mutable reference to it and modify the list. Also, it doesn't provide value equality.
- `ImmutableList<T>` like the name suggests _is_ immutable, but it doesn't provide value equality. Not to mention its suboptimal performance characteristics.

### A solution: ValueCollections

Of course I wouldn't be writing all this if I wasn't trying to push you my [ValueCollections](https://badeend.github.io/ValueCollections/) library :). It provides a handful of collection types that are both immutable and provide value equality. Just what we're looking for:

```cs
using Badeend.ValueCollections;

public record Polygon(ValueList<Point> Points); // Notice the list type.

[Fact]
public void PolygonEquality()
{
    // Two dinstinct instances with the same contents:
    var a = new Polygon([new(3, 1), new(4, 1), new(4, 3)]);
    var b = new Polygon([new(3, 1), new(4, 1), new(4, 3)]);

    // a.Points[0] = new Point(42, 42); // Won't compile. ValueList is immutable. Yay!

    Assert.Equal(a, b); // Succeeds. Yay!
}
``` 

Success!

### Other features

The ValueCollections are designed to be a zero-overhead drop-in replacement for the `System.Collections.Generic.*` collections. They mostly have the same API and performance characteristics as their mutable counterparts to make the switch as "boring" as possible. Other notable features:
- Fully NRT annotated.
- First-class support for .NET Framework. Even in combination with functionalities not originally present in .NET Framework, such as: Spans & C#12 collection expressions.
- Support for `System.Text.Json` & `Newtonsoft.Json` serialization.
- Support for loading `EntityFramework` & `EntityFrameworkCore` queries directly into ValueCollections. E.g. `.ToValueListAsync()`.
- Passes the .NET Runtime's own testsuite wherever possible.

See https://badeend.github.io/ValueCollections/ for more information.
