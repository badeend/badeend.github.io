# Mutable collections are hard: List.InsertRange

Quick question: given the following list:

```cs
List<int> myList = [1, 2, 3, 4];
```

What does the following code produce?:

```cs
// A:
myList.InsertRange(2, myList);
```

and how about this?:

```cs
// B:
myList.InsertRange(2, myList.AsReadOnly());
```

or this one?:

```cs
// C:
myList.InsertRange(2, myList.Where(_ => true));
```

---

I'll assure you:
- All of these snippets call out to exactly the same `InsertRange(int, IEnumerable<T>)` method.
- The `IEnumerable<T>` arguments passed to `InsertRange` produce exactly the same sequences, albeit through increasingly convoluted mechanisms.

So, all examples call the same method with effectively the same argument. Surely, regardless of what the outcome ought to be, they will all be the same, right?

### Wrong!

- **Example A** produces a list of eight elements: `[1, 2, 1, 2, 3, 4, 3, 4]`.
- **Example B** also produces a list of eight elements, but with garbage content: `[1, 2, 1, 2, 0, 0, 3, 4]`. Notice the zeroes? Where did they come from?
- **Example C** throws an `InvalidOperationException` but not before leaving behind a list of **five (!)** elements: `[1, 2, 1, 3, 4]`

AFAIK, this behavior isn't documented anywhere.

Out of these results, example B is clearly the worst offender: it corrupts the list without letting us know that anything went wrong. Example C is somewhat better, in that it at least throws an exception, but it still leaves the list in an inconsistent state. If the list continues to be used after the exception has been caught this could still lead to hard-to-debug issues.

### Why?

As always, [the source code](https://github.com/dotnet/runtime/blob/194ad1667552ed2538bbbb83e336979b02ae6482/src/libraries/System.Private.CoreLib/src/System/Collections/Generic/List.cs#L809-L861) sheds some light. There are three distinct code paths in the `InsertRange` method:
1. `if (this == c)`: For the case when the collection being inserted is exactly the same reference as the list itself.
2. `if (collection is ICollection<T> c)`: For the case when the provided enumerable implements `ICollection<T>`. So that it can be copied using a single virtual method dispatch to `CopyTo`.
3. `else`: For all other cases, the method falls back to the slowest possible way of copying the elements: enumerating the collection and inserting them one by one.

This explains the results from above:
- Example A succeeds because this is deliberately special cased.
- Example B produces incorrect results because the `AsReadOnly` method returns a `ReadOnlyCollection<T>` which _does_ implement `ICollection<T>` but it is not exactly the same reference as the original list. The `ReadOnlyCollection<T>.CopyTo` implementation forwards the call to the underlying list's `CopyTo` method, which ends up overwriting the list's backing array with itself, exposing uninitialized data.
- Example C fails because it falls back to the List's `IEnumerator<T>` implementation, which is [designed to fail](https://github.com/dotnet/runtime/blob/194ad1667552ed2538bbbb83e336979b02ae6482/src/libraries/System.Private.CoreLib/src/System/Collections/Generic/List.cs#L1219-L1222) _after_ it notices the list has been modified. This is the infamous _"Collection was modified; enumeration operation may not execute"_ exception you may have encountered in other contexts.

### Conclusion

There are a couple of takeaways here:
- Implementing `IEnumerable<T>` on mutable data structures is _hard_ :). Or at least: doing it _correctly_ is.
- Consider adding explicit checks or safeguards in your code to handle cases where a mutable data structure might be modified by itself, to prevent subtle bugs and data corruption.
- .NET ships with a lot of different ways to create & compose enumerables. When comparing two `IEnumerable<T>` instances, be aware that while reference equality implies an identical underlying source, reference *in*equality does **not** imply the opposite.

### How this impacts Badeend.ValueCollections

I came across this issue while implementing the `ValueList<T>.Builder` type in my [ValueCollections](https://badeend.github.io/ValueCollections/) library. I chose to err on the side of caution and always throw an exception when attempting to mutate a collection with itself. If anyone ever needs it, I'm open to adding an `.InsertSelf(int at)`-like method, but so far I've never needed it.
