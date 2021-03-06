<pre class='metadata'>
Title: Deducing this
Status: D
ED: wg21.tartanllama.xyz/deducing-this
Shortname: dXXXX
Level: 0
Editor: Barry Revzin, barry.revzin@gmail.com
Editor: Simon Brand, simon@codeplay.com
Abstract: Put abstract here
Group: wg21
Audience: EWG
Markup Shorthands: markdown yes
Default Highlight: C++
</pre>


# Motivation

In C++03, member functions could have cv-qualifications, so it was possible to have scenarios where a particular class would want both a `const` and non-`const` overload of a particular member (Of course it was possible to also want `volatile` overloads, but those are less common). In these cases, both overloads do the same thing - the only difference is in the types accessed and used. This was handled by either simply duplicating the function, adjusting types and qualifications as necessary, or having one delegate to the other. An example of the latter can be found in Scott Meyers' "Effective C++", Item 3:

```
class TextBlock {
public:
  const char& operator[](std::size_t position) const {
     // ...
    return text[position];
  }

  char& operator[](std::size_t position) {
    return const_cast<char&>(
      static_cast<const TextBlock&>(this)[position]
    );
  }
  // ...
};
```

Neither the duplication or the delegation via `const_cast` are arguably great solutions, but they work.

In C++11, member functions acquired a new axis to specialize on: ref-qualifiers. Now, instead of potentially needing two overloads of a single member function, we might need four: `&`, `const&`, `&&`, or `const&&`. We have three approaches to deal with this: we implement the same member four times, we can have three of the overloads delegate to the fourth, or we can have all four delegate to a helper, private static member function. One example might be the overload set for `optional<T>::value()`. The way to implement it would be something like:

## Quadruplication

```
template <typename T>
class optional {
    // ...
    constexpr T& value() & {
        if (has_value()) {
            return this->m_value;
        }
        throw bad_optional_access();
    }

    constexpr const T& value() const& {
        if (has_value()) {
            return this->m_value;
        }
        throw bad_optional_access();
    }

    constexpr T&& value() && {
        if (has_value()) {
            return std::move(this->m_value);
        }
        throw bad_optional_access();
    }

    constexpr const T&&
    value() const&& {
        if (has_value()) {
            return std::move(this->m_value);
        }
        throw bad_optional_access();
    }
    // ...
};
```

## Delegate to 4th

```
template <typename T>
class optional {
    // ...
    constexpr T& value() & {
        return const_cast<T&>(
            static_cast<optional const&>(
                *this).value());
    }

    constexpr const T& value() const& {
        if (has_value()) {
            return this->m_value;
        }
        throw bad_optional_access();
    }

    constexpr T&& value() && {
        return const_cast<T&&>(
            static_cast<optional const&>(
                *this).value());
    }

    constexpr const T&&
    value() const&& {
        return static_cast<const T&&>(
            value());
    }
    // ...
};
```

## Delegate to helper

```
template <typename T>
class optional {
    // ...
    constexpr T& value() & {
        return value_impl(*this);
    }

    constexpr const T& value() const& {
        return value_impl(*this);
    }

    constexpr T&& value() && {
        return value_impl(std::move(*this));
    }

    constexpr const T&&
    value() const&& {
        return value_impl(std::move(*this));
    }

private:
    template <typename Opt>
    static decltype(auto)
    value_impl(Opt&& opt) {
        if (!opt.has_value()) {
            throw bad_optional_access();
        }
        return std::forward<Opt>(opt).m_value;
    }


    // ...
};
```

It's not like this is a complicated function. Far from. But more or less repeating the same code four times, or artificial delegation to avoid doing so, is the kind of thing that begs for a rewrite. Except we can't really. We _have_ to implement it this way. It seems like we should be able to abstract away the qualifiers. And we can... sort of. As a non-member function, we simply don't have this problem:

```
template <typename T>
class optional {
    // ...
    template <typename Opt>
    friend decltype(auto) value(Opt&& o) {
        if (o.has_value()) {
            return std::forward<Opt>(o).m_value;
        }
        throw bad_optional_access();
    }
    // ...
};
```

This is great - it's just one function, that handles all four cases for us. Except it's a non-member function, not a member function. Different semantics, different syntax, doesn't help.

There are many, many cases in code-bases where we need two or four overloads of the same member function for different `const`- or ref-qualifiers. More than that, there are likely many cases that a class should have four overloads of a particular member function, but doesn't simply due to laziness by the developer. We think that there are sufficiently many such cases that they merit a better solution than simply: write it, then write it again, then write it two more times.

# Proposal

This paper proposes a new way of declaring a member function that will allow for deducing the type and value category of the class instance parameter, while still being invokable as a member function.

We propose allowing the naming of the first parameter of a cv/ref-unqualified member function `this`, which shall be of reference type. The `this` parameter will bind to the implicit object argument, as if it were passed as the first argument to a non-member function with the same signature. 

```
struct X {
    template <typename This>
    void foo(This&& this, int i);
};

// X::foo is a member function taking one argument of type int
X x;
x.foo(0);

// and is deduced as if the call were to this non-member function
template <typename This>
void foo(This&& obj, int i);
foo(x, 0);
```

The usual template deduction rules apply to the `this` parameter. While the naming of the parameter `this` is significant, the naming of the template type parameter as `This` is not. It is used throughout merely a suggested convention.

```
struct Y {
    template <typename This, typename T>
    void bar(This&& this, T&& );

    template <typename Self>
    void quux(Self& this);
};

void demo(Y y, const Y* py) {
    y.bar(4);     // invokes Y::bar<Y&, int>
    py->bar(2.0); // invokes Y::bar<const Y&, double>

    Y{}.quux();   // ill-formed
    y.quux();     // invokes Y::quux<Y>
    py->quux();   // invokes Y::quux<const Y>
}
```
It will still be possible to take pointers to these member functions. Their types would be qualified based on the deduced qualification of the instance object. That is, `decltype(&Y::bar<Y, int>)` is `void (Y::*)(int) &&` and `decltype(&Y::quux<const Y>)` is `void (Y::*)() const&`. These member functions can be invoked via pointers to members as usual. 

While the type of the `this` parameter is deduced, it will always be some qualified form of the class type in which the member function is declared, never a derived type:

```
struct B {
    template <typename This>
    void do_stuff(This&& this);
};

struct D : B { };

D d;
d.do_stuff();  // invokes B::do_stuff<B&>, not B::do_stuff<D&>
```

Within these member functions, the keyword `this` will be used as a reference, not as a pointer. While inconsistent with usage in normal member functions, it is more consistent with its declaration as a parameter of reference type and its ability to be deduced as a forwarding reference. This difference will be a signal to users that this is a different kind of member function, additionally obviating any questions about checking against `nullptr`. 

Accessing members would be done via `this.mem` and not `this->mem`. There is no implicit `this` object, since we now have an _explicit_ instance parameter, so all member access must be qualified:

```
template <typename T>
class Z {
    T value;
public:
    template <typename Object>
    decltype(auto) get(Object&& this) {
        return value; // error: unknown identifier 'value'
        return std::forward<Object>(this).value; // ok
    }
};
```

While the examples so far all have `this` as a parameter whose type is deduced, we will also allow using the _injected-class-name_ as the type of the parameter. No other form will be allowed. We do not expect this form to be used very often, but likewise we see no reason to artificially limit the proposal to templates.

```
struct A {
    int i;

    // C++11
    void foo() & { std::cout << i; }

    // this proposal: ok, if unlikely to be used
    void foo(A& this) { std::cout << this.i; }

    // error: this as a parameter must have reference type
    void foo(A* this) { std::cout << this->i; }
};

template <typename T>
struct B {
    // preferred
    template <typename This>
    void best(This&& this);

    // ok
    void good(B& this); 

    // not supported: injected-class-name only
    void bad1(B<T>& this);

    // not supported: template parameter type must be of
    // the form T& or T&& only
    template <typename U>
    void bad2(B<U*>& this);
};
```

Since in many ways member functions act as if they accepted an instance of class type as their first parameter (for instance, in `INVOKE` and all the functional objects that rely on this), we believe this is a logical extension of the language rules to solve a common and growing source of frustration. This sort of code deduplication is, after all, precisely what templates are for.

Overload resolution between new-style and old-style member functions would compare the explicit this parameter of the new functions with the implicit this parameter of the old functions:

```
struct C {
    template <typename This>
    void foo(This& this); // #1
    void foo() const;     // #2
};

void demo(C* c, C const* d) {
    c->foo(); // calls #1, better match
    d->foo(); // calls #2, non-template preferred to template
}
```

As `this` cannot be used as a parameter name today, this proposal is purely a language extension. All current syntax remains valid.

## Alternative syntax

Rather than naming the first parameter `this`, we can also consider introducing a dummy template parameter where the qualifications normally reside. This syntax is also ill-formed today, and is purely a language extension:

```
template <typename T>
struct X {
    T value;

    // as proposed
    template <typename This>
    decltype(auto) foo(This&& this) {
        return std::forward<This>(this).value;
    }

    // alternative
    template <typename This>
    decltype(auto) foo() This&& {
        return std::forward<This>(*this).value;
    }
};
```

# Examples

## Deduplicating Code

This proposal can de-duplicate and de-quadruplicate a large amount of code. In each case, the single function is only slightly more complex than the initial two or four, which makes for a huge win. What follows are a few examples of how repeated code can be reduced.

The particular implementation of optional is Simon's, and can be viewed on [GitHub](https://github.com/TartanLlama/optional), and this example includes some functions that are proposed in P0798, with minor changes to better suit this format:

<table>
<tr><th>C++17</th><th>This proposal</th>
<tr>
<td>
```
class TextBlock {
public:
  const char&
  operator[](std::size_t position) const {
    // ...
    return text[position];
  }

  char& operator[](std::size_t position) {
    return const_cast<char&>(
      static_cast<const TextBlock&>
        (this)[position]
    );
  }
  // ...
};
```
<td>
```
class TextBlock {
public:
  template <typename This>
  auto& operator[](This& this,
                   std::size_t position) {
    // ...
    return this.text[position];
  }
  // ...
};
```
</td>
</tr>
<tr>
<td>
```
template <typename T>
class optional {
  // ...
  constexpr T* operator->() {
    return std::addressof(
      this->m_value);
  }

  constexpr const T*
  operator->() const {
    return std::addressof(
      this->m_value);
  }
  // ...
};
```
</td>
<td>
```
template <typename T>
class optional {
  // ...
  template <typename This>
  constexpr auto operator->(This& this) {
    return std::addressof(this.m_value);
  }
  // ...
};
```
</td>
</tr>

<tr>
<td>
```
template <typename T>
class optional {
  // ...
  constexpr T& operator*() & {
    return this->m_value;
  }

  constexpr const T& operator*() const& {
    return this->m_value;
  }

  constexpr T&& operator*() && {
    return std::move(this->m_value);
  }

  constexpr const T&&
  operator*() const&& {
    return std::move(this->m_value);
  }

  constexpr T& value() & {
    if (has_value()) {
      return this->m_value;
    }
    throw bad_optional_access();
  }

  constexpr const T& value() const& {
    if (has_value()) {
      return this->m_value;
    }
    throw bad_optional_access();
  }

  constexpr T&& value() && {
    if (has_value()) {
      return std::move(this->m_value);
    }
    throw bad_optional_access();
  }

  constexpr const T&& value() const&& {
    if (has_value()) {
      return std::move(this->m_value);
    }
    throw bad_optional_access();
  }
  // ...
};
```
</td>
<td>

```
template <typename T>
class optional {
  // ...
  template <typename This>
  constexpr auto&& operator*(This&& this) {
    return forward<This>(this).m_value;
  }

  template <typename This>
  constexpr auto&& value(This&& this) {
    if (this.has_value()) {
      return forward<This>(this).m_value;
    }
    throw bad_optional_access();
  }
  // ...
};
```
</td>

<tr>
<td>
```
template <typename T>
class optional {
  // ...
  template <typename F>
  constexpr auto and_then(F&& f) & {
    using result =
      invoke_result_t<F, T&>;
    static_assert(
      is_optional<result>::value,
      "F must return an optional");

    return has_value()
        ? invoke(forward<F>(f), **this)
        : nullopt;
  }

  template <typename F>
  constexpr auto and_then(F&& f) && {
    using result =
      invoke_result_t<F, T&&>;
    static_assert(
      is_optional<result>::value,
      "F must return an optional");

    return has_value()
        ? invoke(forward<F>(f),
                 std::move(**this))
        : nullopt;
  }

  template <typename F>
  constexpr auto and_then(F&& f) const& {
    using result =
      invoke_result_t<F, const T&>;
    static_assert(
      is_optional<result>::value,
      "F must return an optional");

    return has_value()
        ? invoke(forward<F>(f), **this)
        : nullopt;
  }

  template <typename F>
  constexpr auto and_then(F&& f) const&& {
    using result =
      invoke_result_t<F, const T&&>;
    static_assert(
      is_optional<result>::value,
      "F must return an optional");

    return has_value()
        ? invoke(forward<F>(f),
                 std::move(**this))
        : nullopt;
  }
  // ...
};
```
</td>
<td>
```
template <typename T>
class optional {
  // ...
  template <typename This, typename F>
  constexpr auto
  and_then(This&& this, F&& f) & {
    using val = decltype(
        forward<This>(this).m_value);
    using result = invoke_result_t<F, val>;

    static_assert(
      is_optional<result>::value,
      "F must return an optional");

    return this.has_value()
        ? invoke(forward<F>(f),
                 forward<This>
                   (this).m_value)
        : nullopt;
  }
  // ...
};
```
</td>
</table>

Keep in mind that there are a few more functions in P0798 that have this lead to this explosion of overloads, so the code difference and clarity is dramatic.

For those that dislike returning auto in these cases, it is very easy to write a metafunction that matches the appropriate qualifiers from a type. Certainly simpler than copying and pasting code and hoping that the minor changes were made correctly in every case.


## SFINAE-friendly callables

A seemingly unrelated problem to the question of code quadruplication is that of writing these numerous overloads for function wrappers, as demonstrated in [P0826]. `std::not_fn` is specified to behave as if:

```
template <typename F>
class call_wrapper {
    F f;
public:
    // ...
    template <typename... Args>
    auto operator()(Args&&... ) &
        -> decltype(!declval<invoke_result_t<F&, Args...>>());

    template <typename... Args>
    auto operator()(Args&&... ) const&
        -> decltype(!declval<invoke_result_t<const F&, Args...>>());

    // ...
};
```

which has the surprise result of incorrectly propagating the cv-qualifiers of the call wrapper. From the paper:

```
struct fun {
    template <typename... Args>
    void operator()(Args&&...) = delete;

    template <typename... Args>
    bool operator()(Args&&...) const { return true; }
};

std::not_fn(fun{})(); // ok? Returns false
```

`fun` shouldn't be invocable unless it's `const`, but the simple approach of quadruplicating the overloads led to a situation where we can fallback to the const overload of the call wrapper (the `&&`-qualified overload led to a substitution failure so we were able to fall back to the `const&&`-qualified one, which succeeded). Implementing this correctly, while still preserving SFINAE-friendliness, is decidedly non-trivial.

This proposal, however, offers a very simple solution to this problem. Deduce `this`. The following is a complete implementation of `std::not_fn`:

```
template <typename F>
struct call_wrapper {
    F f;

    template <typename This, typename... Args>
    auto operator()(This&& this, Args&&... )
        -> decltype(!invoke(forward<This>(this).f,
                            forward<Args>(args)...));
    {
        return !invoke(forward<This>(this).f,
                       forward<Args>(args)...);
    }
};

template <typename F>
auto not_fn(F&& f) {
    return call_wrapper<std::decay_t<F>>{std::forward<F>(f)};
}

not_fn(fun{})(); // error
```

Here, there is only one overload with everything deduced together, with `This = fun`. As this overload is not viable (because `fun` is not invocable unless it's `const`), and there is no other overload, overload resolution is complete without a viable candidate. As desired.

## Recursive Lambdas

This proposal also allows for an alternative solution to implementing a recursive lambda, since now we open up the possibility of allowing a lambda to reference itself:

```
// as proposed in [P0839]
auto fib = [] self (int n) {
    if (n < 2) return n;
    return self(n-1) + self(n-2);
};

// this proposal
auto fib = [](auto& this, int n) {
    if (n < 2) return n;
    return this(n-1) + this(n-2);
};
```

In the specific case of lambdas, a lambda could both capture `this` and take a generic parameter named `this`. If this happens, use of `this` would refer to the parameter (and hence, the lambda itself) and not the `this` pointer of the outer-most class. 

```
struct A {
	int bar();

    auto foo() {
	    return [this](auto& this, int n) {
		    return this->bar() + n; // error: no operator->() for this lambda
	    };
    }
};
```

