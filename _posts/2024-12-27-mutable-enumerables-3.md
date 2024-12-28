# Mutable collections are hard: HashSet.RemoveWhere (cont.)

A quick follow-up to the [previous post]({% post_url 2024-12-27-mutable-enumerables-2 %}). This time we'll abuse `RemoveWhere` to silently cause undefined behavior without throwing an exception.

### Setup

First, we'll prepare two distinct but equivalent sets:

```cs
HashSet<int> setA = [42];
HashSet<int> setB = [0, 42];
setB.Remove(0);

Assert.True(setA.SetEquals([42])); // Sanity check.
Assert.True(setB.SetEquals([42])); // Sanity check.
```

### Pure malice

Next, we'll apply the following operation to both sets:

```cs
set.RemoveWhere(_ =>
{
    set.Add(314);
    return true;
});
```

### The result

Unlike last time, this time there's no exception to warn us we're doing something naughty. Instead, we silently get undefined results:
- `setA` is left behind empty: `[]`.
- `setB` is left behind with a single element: `[314]`.

The difference has to do with how the HashSet internally lays out its content in memory. The exact details are a bit much to dive into here, but conclusion stays the same:

### Conclusion

Like [last time]({% post_url 2024-12-27-mutable-enumerables-2 %}), just don't do it :).
