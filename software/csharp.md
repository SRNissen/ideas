# General

Class|Purpose
-|-
FF | Free functions
Check | validation functions returning bool
Require | validation functions that throw 

Or something like that.


```csharp

var hasNulls = Check.For.Nulls(params object?[] objects)

var isValid = Check.If.Valid<T>(T? t);

Throw.IfNot.Valid<T>(T? t);
Throw.Unless.Valid<T>(T? t);
Require.Valid<T>(T? t);
```

# Nullable Null Finder

I want to do this:

```csharp
var t = Lib.Deserialize<T>(someSource());

var valid = FF.Validate<T>(t); //Validate<T>(T? t)
```

Starting from the very root, `Validate` reflects over `t`, recursively checking for null and returns some kind of validation result (or possibly throws) that ensures the nullability invariants of `t` are correct.

Possibly it also takes a param list of functions that operate on T and ensure they all pass too in case there are other invariants on `t` that might have gotten lost during serialization apart from just nulls.

At this point you might argue that most deserializers already have something *like* this that I can pass in as option parameters to them and/or set somehow.

That is true but: How many deserializers do I want to learn the intricacies of? It's an O(format x library) size problem, or I can write 1 (one) standalone validator that will work for all of them.

# Linear null finder

```csharp
var anyNull = FF.AnyNull(a, b, c, d, ...) // AnyNull(params object?[] objects)
```