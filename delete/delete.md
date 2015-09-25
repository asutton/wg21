Deleted declarations
===================================

<div style="text-align:center">
Andrew Sutton<br/>
Date: 2014-12-12<br/>
Document number: DXXX
</div>

## Overview

I often find myself declaring unconstrained generic functions or 
class templates that I plan to overload or specialize later.
The purpose is to simply introduce a name for use in a generic 
algorithm without defining any specific models for which the
algorithm should work. For example:

    template<typename T>
      T f(T) = delete;

    template<typename T>
      auto g(T a) { return f(a); }

Or, for classes:

    template<typename T>
      struct Foo;

    template<typename T>
      struct Bar {
        Foo<T> foo;
      };

Whenever an overload or specialization is not provided, the program
becomes ill-formed either because overload resolution finds a deleted
function or because a class template specialization is incomplete.

However, I can't do this for variable templates:

    template<typename T>
      T data; // Will always declare a variable

Instantiation will always succeed; this is not what I want. What
I really want to write is this:

    template<typename T>
      T data = delete;

So that, when this variable instantiated, the program becomes ill-formed.

For symmetry, we should also allow the ability to delete a class or
union definition.

    template<typename T>
      class S = delete;

Instantiating this class template results in an ill-formed program. This
allows programmers to clearly express the intent that a class does not
have a valid definition (as opposed to simply leaving it undefined).

This proposal does not include alias templates since they cannot be
specialized.

## Wording

### 8.5 Initializers [dcl.init]

Add `= delete` to the *brace-or-equal-initializer* grammar.

<pre><i>brace-or-equal-initializer</i>:
    = delete
</pre>

### 9 Classes [class]

Modify the *class-specifier* grammar.

<pre><i>class-specifier</i>:
    <i>class-head</i> <del>{ <i>member-specification<sub>opt</sub></i> }</del>
    <i>class-head</i> <i>class-definition</i>

<i>class-definition</i>
    { <i>member-specification<sub>opt</sub></i> }
    = delete
</pre>
