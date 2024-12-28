# Mutable collections are hard: HashSet.RemoveWhere

[HashSet<T>.RemoveWhere](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.hashset-1.removewhere) enumerates the set and invokes the provided function for each element. As long as you return `false` from this function, it is functionally equivalent to a `ForEach` method as found on [`List<T>`](List<T>.ForEach). Doing this is also a surefire way to get your team members to hate you, but let's ignore that for now :).

### Setup

This contrived example iterates the hashset and adds the negation of each element to the set:

```cs
HashSet<int> set = [1, 2, 3];

set.RemoveWhere(value =>
{
    set.Add(-value);
    return false;
});
```

### Watch it fail

The `HashSet` implementation does not like this at all and fails miserably with an unhelpful exception:

```text
System.IndexOutOfRangeException : Index was outside the bounds of the array.
Stack Trace:
    at System.Collections.Generic.HashSet`1.RemoveWhere(Predicate`1 match)
```

Apparently, we've tricked the `RemoveWhere` implementation into accessing elements that it shouldn't have.

### Why

At first glance, [the implementation](https://github.com/dotnet/runtime/blob/194ad1667552ed2538bbbb83e336979b02ae6482/src/libraries/System.Private.CoreLib/src/System/Collections/Generic/HashSet.cs#L1208-L1236) seems perfectly fine. It's just a simple for loop over the elements, invokes the provided predicate for each element and clears the entry if the predicate returns `true`. You may ignore the `ref` and `entry.Next >= -1` stuff, they're not relevant here.

Something to note: Clearly, `RemoveWhere` is designed to handle _some_ mutation from within the predicate, as can be seen from the comment:
```cs
// Cache value in case delegate removes it
...
```

So even though our initial example is obviously a bad idea, it's not immediately clear why removing within the predicate is OK but adding is not. The reason is actually quite subtle. I've abbreviated the actual code:

```cs
Entry[]? entries = _entries;
int numRemoved = 0;
for (int i = 0; i < _count; i++)
{
    T value = entries![i].Value;
    if (match(value))
    {
        Remove(value);
    }
}
```

The `_entries` array reference is stashed in a local variable at the beginning of the loop, while `_count` refers to a class instance field that reflects the most up-to-date count. Because our `Add` grows the set, the `_count` will be immediately increased, yet the local `entries` variable continues to point to a now stale array. The loop counter `i` will eventually exceed the bounds of the array, causing the exception.

### Conclusion

- To consumers: Just don't mutate a collection while enumerating it. You're gonna have a bad time :).
- To authors: When designing mutable data structures, be careful with accepting arguments that may execute arbitrary code. Your consumers may (accidentally!) get more creative than you had anticipated. As we've seen, even seemingly fine code can be subtly broken by such misuse. In this case it was easy to spot due to the exception, but in [other cases]({% post_url 2024-12-28-mutable-enumerables-3 %}) we may not be so lucky and end up with silent data corruption.

### How this impacts Badeend.ValueCollections

I came across this issue while implementing the `ValueSet<T>.Builder` type in my [ValueCollections](https://badeend.github.io/ValueCollections/) library. I chose to err on the side of caution and always throw an exception when attempting to mutate a collection during its enumeration.
