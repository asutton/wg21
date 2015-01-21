Overload sets as function arguments
===================================

<div style="text-align:center">
Andrew Sutton<br/>
Date: 2014-12-05<br/>
Document number: DXXX
</div>

## Introduction

Lambdas (and generic lambdas) make it easier to program with generic
libraries. Passing a comparator as an argument to e.g., sort no longer
requires the program to step outside of their current scope to define
a function or function or object that they use exactly once in their
program. These features allow for less code, more concise code, and
(arguably) fewer programming errors.

However, in my experiences with this feature, I often find myself writing
lambdas that look like this:


    sort(first, last, [](const T& a, const T& b) { return b > a; });

This might be more appropriate:

    sort(first, last, greater<T>{});

Or even this:

    sort(first, last, greater<>{});

But that's an extra level indirection that seems unnecessary. What I
really want to do is write this:


    sort(first, last, operator>);

In other words, I want to pass the name of an overloaded function as an
argument, and this should have exactly the same behavior as all of the
functions above.

But what does it mean? How do you deduce the type of an overload set?
You don't. In this context, we replace the operator name with a generic
lambda that uses that expression, and then perform deduction against
that. That generic lambda should look like this:

    [](auto&& a, auto&& b) { 
      return std::forward<decltype(a)>(a) < std::forward<decltype(b)>(b); 
    }

Note that operator names are actually a special case of normal functions.
In general, we should be able to pass any set of overloaded functions
as an argument. For example, we could compute the cosine of a sequence of 
values as:

      vector<double> r(v.size());
      transform(v.begin(), v.end(), result, cos);

In the call to `transform`, `cos` names the set of overloaded `cos` 
functions, presumably including those in `<cmath>` and any others
that happen to be in scope. For functions like this, we can always
replace them with the following generic lambda:

    [](auto&&... xs) { return cos(std::forward<decltype(xs)>(xs)...); }

This lambda is a parameter pack, because it's not knowable which version
of `cos` will be called until the lambda is instantiated. But, it forwards
to the appropriate overload based on the types of its arguments.

## More about operators

The generic lambdas corresponding to operators have special rules since
a) we don't just call them as functions and b) some operator names have
multiple forms. In particular, `*`, `&`, `+`, and `-` have both unary
and binary forms, and `++` and `--` are both unary and postfix expresssions.

When synthesizing an argument for unary/binary operators, we actually
need *polymorphic lambdas*. That is, if we write:

    vector<int> v1 { ... };
    vector<int> v2(v1.size());
    transform(v1.begin(), v1.end(), v2.begin(), operator-);

The lambda for `operator-` would have a closure type like the following:

    struct lambda {
      auto operator()(auto&& x) const { 
        return -std::forward<decltype(x)>(x);
      }
      auto operator()(auto&& x, auto&& y) const { 
        return std::forward<decltype(x)>(x) - std::forward<decltype(y)>(y);
      }
    };

The other unary/binary operators have similar syntheses.

For the unary/postfix operators, I propose that the operator name select
only the prefix form. It seems to be the most common. That is:

    for_each(v1.begin(), v1.end(), operator++);

would have this lambda:

    [](auto&& x) { return ++(std::forward<decltype(x)>(x); }

## Function objects revisited

Note that this proposal also allows the declaration of variables
that refer to overload sets. Here is an alternative formulation of
`std::plus`.

    auto plus = operator+;

This would be equivalent to writing:

    auto plus = [](auto&& a, auto&& b) { 
      return std::forward<decltype(a)>(b) + std::forward<decltype(b)>(b); 
    };


## Qualified ids

If the *id-expression* naming the overloaded function is a *qualified-id*,
then the function called in the lambda expression should also be fully
qualified. So if you write this:

    template<typename T>
      void f(T) { }

    f(std::swap);

The corresponding lambda for the overload argument would be:

    [](auto&&... xs) { return std::swap(std::forward<decltype(xs)>(xs)...); }


## When not to use this feature

If you need to call an operator as a function.

If you need to use a postfix increment or decrement operator.

## Related work

[N3617](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3617.htm)
describes "lifting expressions", which have the same aims of this 
proposal. However, it requires the *lambda-introducer* before the
*id-expression*. This extra annotation seems unnecessary to me.

N3617 goes further and suggests that we allow projection functions like
this:

    struct user{
      int id;
      std::string name;
    };

    vector<user> users{ {4, "A"}, {1, "B"}, {3, "C"}, {0, "D"}, {2, "E"} };
    sort(users.begin(), users.end(), ordered_by([]id));

I think that the current trend is that this problem be solved in the library
and not in the language. For example, the `sort` function could be (easily)
extended to allow the following:

    sort(users.begin(), users.end(), &user::id);

I believe this would have the same effect as example given above, although
it's not clear what `ordered_by` should actually do.

[N3701](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3701.htm)
made brief mention of this feature, more or less in the form that it is
presented here. However, the rules from N3617 for forming lambda expressions
from operators have been adopted here and expanded to handle unary operators.

## Errata 
TODO: What about overload sets based on things like this:

    using std::cos;
    using boost::math::cos;

    transform(v.begin(), v.end(), result, cos);

## Wording


### 14.8.2.1 Deducing template arguments from a function call  [temp.deduct.call]


Add the following paragraphs. The only context where the synthesis of generic 
lambdas from an *id-expression* should be allowed is when the type of the
corresponding parameter is a type template parameter. Note that this context
should be separate from contexts where overload resolution is performed
to take the address of an overloaded function (as those require a target
function type).

If `P` has type `T` where `T` is a type template parameter and `A` is an 
*id-expression* that names a set of overloaded functions, then a new lambda 
expression is derived from `A` as follows.

In general, a generic lambda is formed from an *id-expression* `E` as:

<pre>
    [](auto&& args) { return E(std::forward<decltype(args)>(args)...); }
</pre>

However, if `E` is an *operator-id*, of the form `operator @`, the lambda 
expression depends on the operator:

- If the *operator-id* is `()`, the lambda is formed as:

<pre>
    [](auto&& a, auto...&& args) { return std::forward<decltype(a)>(a)(std::forward<decltype(args)>(args)...); }
</pre>

- Otherwise, if the operator is one of `[]`, the lambda is formed as:

<pre>
    [](auto&& a, auto&& b) { return std::forward<decltype(a)>(a)[std::forward<decltype(b)>(b)]; }
</pre>

- Otherwise, the lambda is formed as:

<pre>
    [](auto&& a, auto&& b) { return std::forward<decltype(a)>(a) @ std::forward<decltype(b)>(b); }
</pre>

The resulting lambda expression is used in place of `A` for type deduction.

[ <i>Example:</i>
<pre>
// FIXME: Write examples.
</pre>
&mdash; <i>end example</i> ]

</body>
