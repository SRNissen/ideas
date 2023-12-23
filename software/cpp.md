# Asciistring

Something like a std::string that only allows ascii characters, or possibly ascii + templating on 8-bit code pages for variations.

# Constraints / Fundamental

wrapper for fundamental types that define overflow behavior

Same as ^prev, with custom upper/lower bound

Same as 2x ^prev^ but with arbitrary types and arbitrary constraints

# non_unique_ptr<T>

Essentially like a std::vector that only has space for 1 object - a wrapper for a heap object that's like unique_ptr but copyable (copies the underlying object)

Sometimes I want explicit ownership semantic of heap objects but I'm not looking to disable copies.

Unlike unique_ptr, I feel no need to have an array version - if you need explicit ownership of a heap allocated array of T without move-only semantics, *use vector<T>*, or have `non_unique_ptr<std::array<T>>` if you don't need dynamic sizing;

# Mutable<T>

wrapper for mutable object that does the thread safety for you so you can e.g. have a const method update the a cache object without data races

Instead of:

```c++

struct Calculator
{
	mutable Data d_;
	std::mutex m_;
	Data calculate(Input const& i) const;
	Data(Const & Data d)
	{
		// here goes whatever code it takes to safely delete this nonsense
	}
};

```

I want to:

```c++
{
	Mutable<Data> d_; // has accessor functions that ensure thread correctness at all times so users don't need to think of mutexes.
	Data calculate(Input const& i) const;
	Data(Const & Data d) = default; //Mutable handles what you need.
};
```

# Better map iterator

Instead of:

```c++
std::map<key,value> m = getMap();

for(auto const& kv : m)
{
	use(kv.second);
}
```

I want to:

```c++
std::map<key,value> m = getMap();

for(auto const& v : ???)
{
	use(value);
}
```

## Implementation ideas

- A wrapper for maps that has begin/end which, in turn, iterate over the maps begin/end but a deref gets you value directly instead of the kv pair
- A wrapper for an *arbitrary container* (useful in generic contexts) that does the metaprogramming to find out whether iterators are a direct passthrough or whether they should return the value

# Extension methods

illegal:

```c++
bool IsIntegralEven(double a)
{
	a.Divide(2.0);
	return a.IsIntegral();
}
```

possibly possible/legal:

```c++
bool IsIntegralEven(double a)
{
	a ^ Do<Divide>(2.0);
	return a ^ True<IsIntegral>();
}
```

Or *something* like that. Expanded upon in [another file](cpp_extension_methods.md)

# AsBytes

function that takes *any* object by const reference returning an array of bytes exactly equal to those of the object.