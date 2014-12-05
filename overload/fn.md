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


    sort(first, last, [](const T& a, const T& b) { return b > a; }

True, it may be easier to write this:

    sort(first, last, greater<T>{});

Or even this:

    sort(first, last, greater<>{});

But that's an extra level indirection that seems unnecessary. What I
really want to do is write;


    sort(first, last, operator>);

Where operator> is an *id-expression* that names an overloaded set of
functions. This should have exactly the same behavior as the the call
to `sort` above that includes the lambda expression.

TODO: Give better motivating examples.

More generally, we should be able to pass any set of overloaded functions
as an argument to a function. For example, we could compute
the cosine of a sequence of values as:

    template<typename T>
    vector<T> fn(const vector<T>& v) {
      vector<double> r(v.size());
      transform(v.begin(), v.end(), result, cos);
      return result;
    }

In the call to `transform`, `cos` names the set of overloaded `cos` 
functions, presumably including those in `<cmath>` and any others
that happen to be in scope. Here, the call to `transform` would have the
same behavior as calling:

    transform(v.begin(), v.end(), result, [](const auto& x) { return cos(x); });

In other words, the use of an overload set as a function argument results
in the synthesis of a generic lambda that calls the function or operator
with the given name.

Note that this proposal also allows the declaration of variables
that refer to overload sets. Here is an alternative formulation of
`std::plus`.

    auto plus = operator+;

This would be equivalent to writing:

    auto plus = [](auto&& a, auto&& b) { 
      return std::forward<decltype(a)>(b) + std::forward<decltype(b)>(b); 
    };


## Related work

[N3617](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3617.htm)
describes "lifting expressions", which satisfy many of the same aims of
this proposal. However, it requires the *lambda-introducer* before the
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
from operators have been adopted here.

## Errata 
TODO: What about overload sets based on things like this:

    using std::cos;
    using boost::math::cos;

    transform(v.begin(), v.end(), result, cos);


TODO: The transformation described below doesn't work for unary operators.
We need a mechanism to select between a unary and binary operator when
the lambda is instantiated. Effectively, we want something that does
this.

    template<typename T>
      auto do_op(T&& x) { return op std::forward<T>(x); }
    
    template<typename T, typename U>
      auto do_op(T&& a, U&& b) { return std::forward<T>(a) op std:forward<T>(b); }

    [](auto&&... args) { return do_op(std::forward<Args>(args)...); }

This could be compiler magic: an internal "forwarding operator" that
selects its appropriate arity when instantiated and not otherwise.

Note that we could also expand this discussion to include fold expressions.
That is, if a binary operator lambda is called with more than 2 arguments,
we could generate this;


    (arg1 op ... op args)


where arg1 is the first in the sequence and args are the rest. Again, this
requires some compiler magic where the actual formation of the operator
is delayed until instantiation.


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

In general, if the  *id-expression* `E`, form a generic lambda as:

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
// FIXME: Write examples examples.
</pre>
&mdash; <i>end example</i> ]

</body>
