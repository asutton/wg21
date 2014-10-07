Folding expressions
===================

<div style="text-align:center">
Andrew Sutton<br/>
Richard Smith<br/>
Date: 2014-10-07<br/>
Document number: NXXX
</div>

## Introduction

This paper introduces a new kind of primary expression that allows
parameter packs to be expanded as a sequence of operators.
We call these "fold expressions", named after then ubiquitous fold
algorithm that applies a binary operator to a sequence values
(also called `accumulate` in the standard library).

Today, when we want to compute folds over a parameter pack, we have to 
resort to a handful of overloads of a variadic templates in order to compute 
the result. For example, a simple summation might be implemented like
this:

    template<typename... Ts>
      auto sum(Ts... args);

    auto sum() { return 0; }

    template<typename T>
      auto sum(T t) { return t; }

    template(typename T, typename... Ts)
      auto sum(T t, Ts... ts) { return t + sum(ts...); }

We should be able to do better. In particular, given a function
parameter pack `args`, we would like to be able to compute its
summation like this.


    (args + ...)

This is a significant savings in keystrokes, and programmers are much
less likely to get this wrong than the implementation above.

There are a number of binary operators for which folding can defined. One
such is the `,` operator. The `,` operator can be used, for example, to apply 
a function to a sequence of elements in a parameter pack. For example, 
printing can
be done like this:

    template<typename T> 
      void print(const T& x) { cout << x << '\n'; }
    
    template<typename T>
      void for_each(const auto&... args) {
        (print(args), ...);
      }


## Extra motivation

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

What are the corresponding requirements? Can we do this?

    template<typename... Ts>
      requires Integral<Ts>... // error: ill-formed
    void foo(Ts...);

This doesn't work because we can't expand an argument pack in this
context. What about this?

    template<typename... Ts>
      requires std::all_of({Ts...}) // OK?
    void foo(Ts...);

That might work if we had range algorithms and those algorithms were
declared `constexpr`. Unfortunately, we have neither.

However, with this fold syntax, that transformation is straightforward.

    template<typename... Ts>
      requires (Integral<Ts> && ...)
    void foo(Ts...);


## Proposal


The proposal adds a new kind of *primary-expression*, called a
*fold-expression*. A fold expression can be written like this:

    (args + ...)

The arguments are expanded over the `+` operator as left fold. That is:

    ((args$0 + args$1) + ...) + args$n

Or, you can write the expression with the parameter pack on the right
of the operator, like this:

    (... + args)

With this spelling, the arguments are expanded over the operator as
a right fold:

    args$0 + (... + (args$n-1 + args$n))

The fold of an empty parameter pack evaluates to a specific value. The choice
of value depends on the operator. The set of operators and their empty
expansions are in the table below.

<table>
<tr><th>Operator</th>    <th>Value when parameter pack is empty</th></tr>
<tr><td><tt>*</tt></td>  <td><tt>1</tt></td>                        </tr>
<tr><td><tt>+</tt></td>  <td><tt>0</tt></td>                        </tr>
<tr><td><tt>&</tt></td>  <td><tt>-1</tt></td>                       </tr>
<tr><td><tt>|</tt></td>  <td><tt>0</tt></td>                        </tr>
<tr><td><tt>&&</tt></td> <td><tt>true</tt></td>                     </tr>
<tr><td><tt>||</tt></td> <td><tt>false</tt></td>                    </tr>
<tr><td><tt>,</tt></td>  <td><tt>void()</tt></td>                   </tr>
</table>

If a fold of an empty parameter pack is produced for any other operator, the
program is ill-formed. However, all binary operators are supported to allow
the use of folds over arbitrary overloaded operators.

In order to support expansions over a parameter pack and other operands, or
to customize the behavior for a fold over an empty parameter pack, you
can also use the mathematically oriented phrasing:

    (args + ... + an)

This expands as a left fold, including the `an` as the last term in
the sequence. Only one of the operands can be a parameter pack. You can
also expand a right fold.

    (a0 + ... + args)

## Wording


### 5.1. Primary expressions \[expr.prim\]

Modify the grammar of *primary-expression* in [expr.prim] to include
fold expressions.

<pre><i>primary-expression</i>:
    <i>fold-expression</i></pre>

Add the a new subsection to [expr.prim] called "Fold expressions"

#### 5.1.3 Fold expressions \[expr.prim.fold\]

A fold expression allows a template parameter pack ([temp.variadic]) over 
a binary operator.

<pre><i>fold-expression</i>:
      ( <i>cast-expression</i> <i>fold-operator</i> ... )
      ( ... <i>fold-operator</i> <i>cast-expression</i> )
      ( <i>cast-expression</i> <i>fold-operator</i> ... <i>fold-operator</i> <i>cast-expression</i> )

<i>fold-operator</i>: one of
    +  -  *  /  %  ^  &  |  ~  =  &lt;  >  &lt;&lt;  >>
    +=  -=  *=  /=  %=  ^=  &=  |=  &lt;&lt;=  >>=
    ==  !=  <=  >=  &&  ||  ,  .*  ->*</pre>

An expression of the form `(e op ...)` where `op` is a *fold-operator* is
called a *left fold*. The *cast-expression* `e` shall contain an
unexpanded parameter pack. A left fold expands as expression
`((e$1 op e$2) op ...) op e$n` where `$n` is an index into the unexpanded
parameter pack.

An expression of the form `(... op e)` where `op` is a *fold-operator* is
called a *right-fold*. The *cast-expression* `e` shall contain an
unexpanded parameter pack. A right fold expands as expression
`e$1 op (... (e$n-1 op e$n))` where `$n` is an index into the unexpanded
parameter pack.

\[ *Example:*

    template<typename... Args>
      bool all(Args... args) { return (args && ...); }

    all(true, true, true, false);

Within the instantiation of `all`, the returned expression expands to
`((true && true) && true) && false`, which evalutes to `false`.
-- *end example* \]

In an expression of the form `(e1 op ... op e2)` either `e1` shall have
an unexpanded parameter pack or `e2` shall have an unexpanded parameter
pack, but not both. If `e1` contains an unexpanded parameter pack, the 
expression is a left fold and `e2` is rightmost operand the expansion. If
`e2` contains an unexpanded parameter pack, the expression is a right
fold and `e1` is the leftmost operand in the expansion.

\[ *Example:*

    template<typename... Args>
      bool f(Args... args) { 
        return (true + ... + args); // *OK*
      } 

    template<typename... Args>
      bool f(Args... args) { 
        return (args &lt ... && args); // *error: both operands contain unexpanded
      }                                // parameter packs
-- *end example* \]

When the unexpanded parameter pack in a fold expression expands to an
empty sequence, the value the expression is shown in Table N; the program
is ill-formed if the operator is not listed in Table N.

<table>
<caption>Table N. Value of folding empty sequences</caption>
<tr><th>Operator</th>    <th>Value when parameter pack is empty</th></tr>
<tr><td><tt>*</tt></td>  <td><tt>1</tt></td>                        </tr>
<tr><td><tt>+</tt></td>  <td><tt>0</tt></td>                        </tr>
<tr><td><tt>&</tt></td>  <td><tt>-1</tt></td>                       </tr>
<tr><td><tt>|</tt></td>  <td><tt>0</tt></td>                        </tr>
<tr><td><tt>&&</tt></td> <td><tt>true</tt></td>                     </tr>
<tr><td><tt>||</tt></td> <td><tt>false</tt></td>                    </tr>
<tr><td><tt>,</tt></td>  <td><tt>void()</tt></td>                   </tr>
</table>
