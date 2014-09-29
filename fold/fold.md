Folding expressions
===================

## Introduction

This paper seeks EWG guidance for a problem encountered in the semantics
of constrained-parameters in N4040 as they relate to variadic templates.
We explain the problem and suggest a solution for a broader class
of problems related to programming with parameter packs. The goal is to 
develop a concrete EWG proposal for Urbana-Champagne. Note that the proposed 
feature is independent of concepts and could reasonably be targeted for C++17.

## The concept problem

N4040 describes how a *constrained-type-specifier* is transformed into a 
declaration and its constraints. For example:

    template<typename T>
      concept bool Integral = std::is_integral<T>::value;

    template<Integral T>  // "Integral T" is a constrained-parameter
      void foo(T x, T y);

    template<typename T>
      requires Integral<T> // Becomes this.
    void foo(T x, T y);

Hypothetically, this works to declare constrained template parameter packs.

    template<Integral... Ts>  // "Integral Ts" is a constrained-parameter
      void foo(Ts...);

How do we generate the requirement? Can we do this?

    template<typename... Ts>
      requires Integral<Ts>... // error: ill-formed
    void foo(Ts...);

What about this?

    template<typename... Ts>
      requires std::all_of({Ts...}) // OK? If we had range algorithms
    void foo(Ts...);

We could try to make the syntax work in this context, but we prefer a more
general solution that is not tied to concepts.

## Proposal

The `requires` clause from the function `foo` would be rewritten using a
new kind of expression:

    template<typename... Ts>
      requires (Integral<Ts> && ...) // OK: fold-expression
    void foo(Ts...);

The fold expression `(Integral<Ts> &&...)` is a left fold over the parameters
in the template parameter pack, `Ts`. Let `T1`, `T2`, ..., `Tn` be the 
arguments in the template parameter pack. This would be equivalent to writing:

    requires (Integral<T1> && Integral<T2> && ... && Integral<Tn>)

Note that && associates to the left. This new expression gives us the desired 
result.

## Fold Expressions

A fold-expression can be defined over every binary operator defined in the IS,
not just &&. If `args` is function argument pack, the following are valid:

    (args ==...)   // True if all arguments are the same
    (args +...)    // Computes the sum of arguments
    (f(args), ...) // Applies f to each argument

A right folds are also be supported. For example:

    (... || args)

generates the expression `(((an || an-1) || an-2) || ... || a1)`

If the argument pack `args` contains a single element, then the result of 
folding `(args && ...)` is equivalent to `(args)`. This is the same for all
operators.

But what happens if the argument pack is empty? The result depends on the 
operator. Below is an initial list of result values for folds of empty 
sequences. Let `T` be the type of the substituted expression.

- `&&` becomes `T(true)`
- `||` becomes `T(false)`
- `+`, `-` becomes `T()`
- `*`, `/` becomes `T(1)` (maybe?)
- `%` becomes ???
- `&`, `|`, `^` becomes T() (maybe?)

We probably don't want to define a result unless an identity is actually
known. We could also extend the syntax to allow for a default value in the
empty case (with reasonably well-chosen defaults). 

Note that we probably don't want to fold relations since the rewrite of, say
`(args == ...)`, would be `a1 == a2 == a3 == ... == an`.

## Examples

One use of this feature is to apply a function to each element in a template
parameter pack: 

    void g(auto);
    
    void f(auto... args) {
      (g(args), ...); // Apply g to all arguments
    }

