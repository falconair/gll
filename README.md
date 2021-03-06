General Parser Combinators in Racket
====================================

This article explains how to implement a general parser combinator
framework which handles left-recursive and ambiguous grammars. It is
implemented in [Racket](http://www.racket-lang.org/), a dialect of
Scheme. However, the principles are not specific to Racket and could
expressed in any language; porting is outlined at the end of the
article. All the code is available from
[GitHub](https://github.com/epsil/gll):

```
git clone https://github.com/epsil/gll.git
```

The repository also includes the full text of the article in
[GitHub Flavored Markdown](http://github.github.com/github-flavored-markdown/).
I welcome suggestions and improvements! Feel free to open an issue on
the [bug tracker](https://github.com/epsil/gll/issues), or to fork the
repository. You can also contact me, Vegard Øye, at
`vegard (underline) oye (at) hotmail (dot) com`.

The article aims to be an accessible introduction to the ideas found
in the papers
[Memoization in Top-Down Parsing](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.14.3000)
by Mark Johnson and
[Generalized Parser Combinators](http://www.cs.uwm.edu/~dspiewak/papers/generalized-parser-combinators.pdf)
by Daniel Spiewak. If you are interested in the topic, I especially
recommend you to read Spiewak's paper. It is very good. Other reading
suggestions are provided at the end.

Introduction
------------

Traditional top-down parsers cannot handle all recursive grammars,
even if recursion may be the most natural way to express the language.
An example is the following grammar for left-associative arithmetic on
0's and 1's:

```
expr -> expr "+" num
      | expr "-" num
      | num
 num -> "0" | "1"
```

If we attempt to translate the first rule directly into a self-calling
function, the resulting parser will never terminate; instead it enters
an infinite regress before consuming any input. The traditional
approaches to parsing left-recursive grammars are to rewrite the
grammar or use a bottom-up parser generator.

The parsers in this article, by contrast, can be freely composed
without regard for left-recursion. Defining a parser for the above
grammar is straightforward:

```Scheme
(define-parser expr
  (alt (seq expr (string "+") num)
       (seq expr (string "-") num)
       num))
(define-parser num
  (alt (string "0") (string "1")))
```

Furthermore, the parsers support ambiguity. Parse results are computed
one at a time and returned as a lazy stream:

```Scheme
> (expr "1+0")
#<stream>
```

The worst-case efficiency of the parser in this article is O(n^4),
but with more efficient data structures, O(n^3) is achievable.
Furthermore, by adorning the parsers with additional metadata,
LL(1) grammars can be parsed in O(n) time. See the references for
details on optimization.

The article is organized into several stages, leading up to a complete
interpreter for arithmetic expressions. To get there, the parsers will
be rewritten multiple times:

1. First, we will write a simple, top-down combinator framework,
   implementing things in the *conventional* way. This won't handle
   left-recursive grammars in any form, but introduces a simple syntax
   for composing parsers.

2. Next, we will rewrite the parsers to *continuation-passing style*.
   Although functionally equivalent to the previous version, the
   continuations make the code more flexible and set the stage for
   implementing general parsers.

3. We will now add support for left-recursive grammars. This is done
   by *memoizing* parse results and continuations, so that nothing is
   computed more than once. This is the most important step, and we
   will study it in some detail to develop our intuition.

4. To optimize the code, we will add a *trampoline* to store parse
   results and dispatch function calls. The trampoline is a shared
   data structure passed down from one parser to another.

5. Now we have the wherewithal to implement a lazy parse process. The
   parsers will return a *stream* of parse results, computing the
   results as they are requested.

The article aims to be easy to understand, but is necessarily of some
length. If you are familiar with parser combinators, you can skim
through the first sections. The key ideas are introduced in the
section on continuation-passing style. The rest of the text deals with
optimization of those ideas.

Simple parser combinators
-------------------------

Let us start by defining some terms. A *parser* is a function that
takes a string as input and returns a *parse result*. A parse result
is either a *success* or a *failure*. (All our parsers will work on
strings, but one could easily define parsers working on a stream of
symbols or tokens.)

A *parser combinator* is a function that takes parsers as input and
returns a new parser. In other words, it's a higher-order function,
taking functions as input and returning another function as output.
Using parser combinators, we can build bigger parsers out of smaller
parsers.

First we will define data types for parse results. A successful result
contains two values: a value and the current string position. The
value can for example be an abstract syntax tree, while the position
is represented as the rest of the string. A failing result just
contains the position where the parser failed, which can be used for
error reporting.

```Scheme
(struct success (val rest) #:transparent)
(struct failure (rest) #:transparent)
```

This Racket code defines `success` and `failure` as constructor
functions for parse results. For example, we can create a successful
result with the expression `(success val rest)`, and a failure with
`(failure rest)`. We can also *pattern match* against these
expressions, which will be demonstrated later. The `#:transparent`
option makes the values printable.

First, we look at a trivial parser that accepts any input and returns
a successful result containing some predefined value. For creating
such parsers, we define the function `succeed`, which takes the
predefined value as a parameter. (This very common function is also
known as "empty", "epsilon", "result", "yield", or "return".)

```Scheme
(define (succeed val)
  (lambda (str)
    (success val str)))
```

Note that `succeed` returns another function `(lambda (str) ...)`,
that is, a parser. For example, `(succeed '())` creates a parser which
succeeds with the empty list:

```Scheme
> (succeed '())
#<procedure>
> ((succeed '()) "foo")
(success '() "foo")
```

The next step is to compare the beginning of the input against some
match string. Let us create a general function `string` for
constructing such a parser. This function takes a matching string as
input and returns a parser for matching against that string:

```Scheme
(define (string match)
  (lambda (str)
    (let* ((len (min (string-length str) (string-length match)))
           (head (substring str 0 len))
           (tail (substring str len)))
      (if (equal? head match)
          (success head tail)
          (failure str)))))
```

We can create a parser for matching, say, `"foo"` with
`(string "foo")`. If the input matches, then the parser returns a
successful result where the `"foo"` part is consumed; otherwise it
returns a failure. For example, the input `"foobar"` gives a value of
`"foo"` and a rest of `"bar"`:

```Scheme
> ((string "foo") "foobar")
(success "foo" "bar")
> ((string "foo") "bar")
(failure "bar")
```

The functions `succeed` and `string` could be regarded as very basic
"parser generators": both create parsers on the basis of a given
specification. The parsers they create are often called *terminal
parsers*, because they match against a terminal expression in the
grammar. All the terminal parsers in this article work on strings, but
one could easily modify them to accept a stream of symbols or tokens.

Now that we have some basic parsers, it is time to combine them. The
first combinator is the *alternatives combinator*, which chooses
between alternative parsers. It returns a combined parser that tries
each alternative in turn until one of them matches.

```Scheme
(define (alt a b)
  (lambda (str)
    (let ((result (a str)))
      (match result
        [(success val rest) result]
        [failure (b str)]))))
```

Here we see Racket's pattern matching in action. We invoke the first
parser with `(a str)` and match against its result. If we get a
successful parse result `(success val rest)`, we return that.
Otherwise, we invoke the second parser with `(b str)`.

The next combinator is the *sequence combinator*, which chains parsers
together. The output of one parser is taken as the input to another.
Here is our first attempt at defining this combinator, using pattern
matching:

```Scheme
(define (seq a b)
  (lambda (str)
    (match (a str)
      [(success val1 rest1)
       (match (b rest1)
         [(success val2 rest2)
          (success (list val1 val2) rest2)]
         [failure failure])]
      [failure failure])))
```

If we get a successful result `(success val1 rest1)` from the first
parser, we pass its remainder to the next parser with `(b rest1)`. If
it also succeeds, we return a combined parse result. In all other
cases, we return failure. (For simplicity, the `alt` and `seq`
combinators take exactly two arguments; later we will write general
versions accepting any number of arguments.)

However, because we are going to rewrite the parsers several times, it
will be beneficial to express this combinator in a more abstract way.
We therefore define a low-level "plumbing" function, `bind`, for
chaining things together:

```Scheme
(define (bind p fn)
  (lambda (str)
    (match (p str)
      [(success val rest)
       ((fn val) rest)]
      [failure failure])))
```

The `bind` function takes two arguments, a parser `p` and a function
`fn`, and returns another parser. The `fn` function must be of the
same kind as the "parser generators" we saw earlier: it must return a
parser on the basis of an input value. Together, `p` and `fn` become a
parser which initially invokes `p` on the input. If the result is
successful, then its value-part is passed to `fn` to create another
parser. That parser is then invoked on the rest of the input.

We can now define `seq` in terms of `bind`:

```Scheme
(define (seq a b)
  (bind a (lambda (x)
            (bind b (lambda (y)
                      (succeed (list x y)))))))
```

This terse definition can be read as: "Invoke the first parser `a` and
bind the value-part of the result to `x`. Then invoke the the second
parser `b` and bind the value-part of the result to `y`. Finally,
create a combined parse result from `x` and `y`." If any of the
parsers fail, then `bind` stops; only successful results are passed
down the chain. Thus, `bind` frees us from the burden of repeatedly
doing pattern matching and discarding failing results.

Let us now look at a complete example. The following implements a
simple linguistic grammar taken from
*[SICP](http://mitpress.mit.edu/sicp/full-text/book/book-Z-H-28.html#%_sec_Temp_618)*:

```Scheme
(define article
  (alt (string "the ")
       (string "a ")))

(define noun
  (alt (string "student ")
       (string "professor ")))

(define verb
  (alt (string "studies ")
       (string "lectures ")))

(define noun-phrase
  (seq article noun))

(define verb-phrase
  (seq verb noun-phrase))

(define sentence
  (seq noun-phrase verb-phrase))
```

We can parse a sentence with:

```Scheme
> (sentence "the professor lectures the student ")
(success '(("the " "professor ")
           ("lectures " ("the " "student "))) "")
> (sentence "not a sentence ")
(failure "not a sentence ")
```

For the time being, the parse result isn't too informative; later we
will define a combinator for modifying it. In any case, we can
definitely see that there is parsing going on!

Memoization
-----------

We now turn to the issue of efficiency. Currently, none of the
functions cache their results. That is wasteful because parsing is a
such a repetitious task. If each function maintains a table over its
input and output values, it can avoid calculating things twice by
returning the cached value instead. This is called *memoization*.

In Racket, it is easy to write a `memo` function that takes any
function as input and wraps it in a memoization routine. The wrapper
takes the input arguments and looks them up in a memoization table. If
it finds an output value, it just returns that. If not, then it calls
the original function, saves its output in the table, and returns the
output. Future calls with the same arguments will return the memoized
value. The following is loosely adapted from
*[SICP](http://mitpress.mit.edu/sicp/full-text/book/book-Z-H-22.html#%_thm_3.27)*:

```Scheme
(define (memo fn)
  (let ((alist (mlist)))
    (lambda args
      (match (massoc args alist)
        [(mcons args result) result]
        [_ (let* ((result (apply fn args))
                  (entry (mcons args result)))
             (set! alist (mcons entry alist))
             result)]))))
```

We implement the memoization table as a mutable *association list*,
using the Racket function `massoc` to access it. It is actually a list
of mutable cons cells `(args . result)`. If `massoc` returns a cons
cell, we match against it and return the result. Otherwise (the
wildcard pattern is denoted by `_`), we call the original function
with `(apply fn args)`, store the result in a cons cell, insert it
into the table, and then return the result.

Now we can easily memoize our definitions. For example, here is a
memoized version of `alt`:

```Scheme
(define (alt a b)
  (memo
   (lambda (str)
     (let ((result (a str)))
       (match result
         [(success val rest) result]
         [failure (b str)])))))
```

This combinator returns a memoized parser. We can memoize all the
functions in this way.

There is one more wrinkle to sort out. Because Racket evaluates
expressions eagerly, we run into a "tying the knot" problem when
defining self-referential parsers:

```Scheme
(define r
  (alt (seq (string "a") r)
       (string "a")))
```

This will give an error because `r` is evaluated as an argument to
`seq` before `r` is defined. We need to delay the evaluation somehow.
One solution is to wrap the code in a function:

```Scheme
(define r
  (lambda (arg)
    ((alt (seq (string "a") r)
          (string "a"))
     arg)))
```

Now the code is evaluated not when the parser is defined, but when it
is invoked. To make things more convenient, we can create a
`delay-parser` macro which automatically delays the code for us, as
well as a `define-parser` macro for delayed parser definitions.

```Scheme
(define-syntax-rule (delay-parser parser)
  (lambda args
    (apply parser args)))

(define-syntax-rule (define-parser parser body)
  (define parser
    (delay-parser body)))
```

Now we can write:

```Scheme
(define-parser r
  (alt (seq (string "a") r)
       (string "a")))
```

However, because the evaluation is delayed, a new parser instance is
created each time the parser is used! This is solved by memoizing the
combinators too: each combinator will only create a given parser once.
Recall that `(define (alt a b) ...)` is syntactic sugar for
`(define alt (lambda (a b) ...))`:

```Scheme
(define alt
  (lambda (a b)
    (memo
     (lambda (str) ...))))
```

This is the same combinator as before, just written in a slightly
different way that emphasizes the function that is bound to `alt`. Now
we can memoize the combinator itself:

```Scheme
(define alt
  (memo
   (lambda (a b)
     (memo
      (lambda (str) ...)))))
```

When we memoize both the parsers and the parser combinators, parsing
is efficient and the delayed execution only creates a single parser
instance.

The current parser combinators support recursion to a limited degree.
The `r` parser is right-recursive because the self-reference is "to
the right" in the sequence. Therefore, `r` will always consume some
part of the input string (an `"a"`) before recursing. In a
left-recursive grammar, on the other hand, the self-reference comes
first:

```Scheme
(define-parser s
  (alt (seq s (string "a"))
       (string "a")))
```

This parser will enter an infinite regress because it repeatedly calls
itself before consuming any input. To handle left-recursive grammars,
we need to reimagine the parser combinators.

Continuation-passing style
--------------------------

So far, we have taken advantage of the fact that many grammars can be
translated directly into a program. Such a program will have a
straightforward, hierarchical structure, with functions calling
functions all the way down to the level of string matching. It will
either return a single result or no result at all.

Not all grammars are this simple, however. Once we introduce
recursion, there is not guarantee that the grammar will translate into
a terminating program, even if the grammar is well-defined.
Furthermore, grammars can be ambiguous: with several matching
alternatives, a string can parsed in multiple, equally valid ways. For
simplicity, our `alt` combinator only returned a single result (the
first that matched). A more complete implementation would return the
*set* of results.

To address these issues, we will rewrite and express our parsers in a
more flexible way: *continuation-passing style*. Instead of having our
parsers return their results to the caller, they will pass them to a
continuation. The continuation then carries on the parsing. All the
parsers will have an additional argument for the continuation they are
to pass their results to. The continuation itself is a function of one
argument. (Racket actually has native continuations, but we will use
functions as continuations to make the implementation more portable.)

Let us start by rewriting the `succeed` function. Recall the original
definition:

```Scheme
(define (succeed val)
  (lambda (str)
    (success val str)))
```

To transform the returned parser to continuation-passing style, we add
a second argument, `cont`. Instead of returning the parse result, we
pass it to `cont`:

```Scheme
(define (succeed val)
  (lambda (str cont)
    (cont (success val str))))
```

To use the parser, we need to supply it with a continuation. Any
function of one argument will do. For example, we can use `print`,
which will cause the result to be printed to standard output:

```Scheme
> ((succeed '()) "foo" print)
(success '() "foo")
```

Of course, this is a bit cumbersome, so in the final version we will
provide a simpler interface for invoking parsers. For now, we proceed
with `string`:

```Scheme
(define (string match)
  (lambda (str cont)
    (let* (...)
      (if (equal? head match)
          (cont (success head tail))
          (cont (failure tail))))))
```

The definitions so far are quite similar to the original. For the
`seq` combinator, however, we need to change the definition of `bind`:

```Scheme
(define (bind p fn)
  (lambda (str cont)
    (p str (lambda (result)
             (match result
               [(success val rest)
                ((fn val) rest cont)]
               [failure
                (cont failure)])))))

(define (seq a b)
  (bind a (lambda (x) ...)))
```

Here we use continuations to chain things together. The parser `p` is
called with a continuation `(lambda (result) ...)` that receives the
result and matches against it. If it is successful, then the
continuation carries on and invokes `fn`. The final result is passed
to `cont`, the continuation for the combined parser. Conveniently, we
only need to modify `bind`; the definition of `seq` is unchanged.

While expressed in a different style, all the code so far functions
the same as before. For the `alt` combinator, however, we will change
the semantics. It will now try *all* alternatives, branching out in
parallel:

```Scheme
(define (alt a b)
  (lambda (str cont)
    (a str cont)
    (b str cont)))
```

The continuation will be invoked twice, once for the first parser and
once for the second. If the first parser succeeds and the second
fails, then the continuation will first receive a successful result,
and then a failing result.

We have now rewritten our parsers to continuation-passing style. In
itself, this doesn't solve the problems we had with recursive
grammars, but it sets the stage for the solution. Observe that while
the `alt` combinator produces two results, it doesn't pass them at the
same time. The execution is more fine-grained: the combinator creates
a separate *branch* for the calculation of each result.

In other words, there is a kind of concurrency here (even if the
current implementation is sequential). The key insight is that in a
recursive grammar, one branch may depend on another: the recursive
branch cannot continue before the base branch has produced a result.
Is there a way we can make the branches cooperate, regardless of their
order of execution?

As it turns out, the answer is memoization! That is because when we
memoize a continuation-passing style function, not only do we keep
track of input values and output values, but also the continuations
that are interested in those values. Each table entry will contain a
list of results (since the function may output more than one value),
and a list of continuations (since the same function may be called in
different places).

Thus, we can broadcast results from one branch of the grammar to
another, "reawakening" the interested branch. How do we awaken a
branch? By calling its continuation! In fact, the same continuation
may be called several times in a recursive grammar, which we will see
below. Like in a video game, each continuation is a "save point" in
our program, and we can reload any part of it as our knowledge
progresses.

To memoize functions written in continuation-passing style, we define
a `memo-cps` wrapper. For clarity, we define a few local functions:
`push-continuation!` adds a continuation to an entry, `push-result!`
adds a result to the entry, `result-subsumed?` checks if the entry
already contains a given result, `make-entry` creates an empty entry,
and `table-ref` looks up an memoization entry, creating an empty entry
if one does not exist.

```Scheme
(define (memo-cps fn)
  (let ((table (mlist)))
    (define entry-continuations mcar)
    (define entry-results mcdr)
    (define (push-continuation! entry cont)
      (set-mcar! entry (mcons cont (entry-continuations entry))))
    (define (push-result! entry result)
      (set-mcdr! entry (mcons result (entry-results entry))))
    (define (result-subsumed? entry result)
      (mmember result (entry-results entry)))
    (define (make-entry)
      (mcons (mlist) (mlist)))
    (define (table-ref str)
      (match (massoc str table)
        [(mcons str entry) entry]
        [_ (let ((entry (make-entry)))
             (set! table (mcons (mcons str entry) table))
             entry)]))
    (lambda (str cont)
      (let ((entry (table-ref str)))
        (match entry
          ;; first time memoized procedure has been called with str
          [(mcons (mlist) (mlist))
           (push-continuation! entry cont)
           (fn str (lambda (result)
                     (unless (result-subsumed? entry result)
                       (push-result! entry result)
                       (for ((cont (entry-continuations entry)))
                            (cont result)))))]
          ;; memoized procedure has been called with str before
          [_
           (push-continuation! entry cont)
           (for ((result (entry-results entry)))
                (cont result))])))))
```

There are two cases to consider: when the memoized function is called
for the first time, and when it has been called before. When the
function is called for the first time, we insert the original
continuation, `cont`, into the table. Then we invoke the function with
a custom continuation `(lambda (result) ...)` which in turn will
invoke `cont`, as well as any other continuations which may have been
inserted into the table in the meantime. We can think of the
continuation `(lambda (result) ...)` as our "man on the inside": it
alone will do the work of being passed into the function and receive
its results. Then it broadcasts those results to the continuations on
the outside.

Thus, in the second case when the function has been called before, we
just insert the continuation into the list of continuations. Our
"inside man" will then notify the continuation of new results as they
are produced. Meanwhile, the continuation goes through the results
that have already been memoized, and then it "goes to sleep".

Now we are ready to memoize the definitions. We use `memo-cps` for the
returned parsers and `memo` for the parser combinators, which are
regular functions. As before, the use of `memo` is necessary because
`delay-parser` delays the combinators to being called at runtime.

```Scheme
(define succeed
  (memo
   (lambda (val)
     (memo-cps
      (lambda (str cont) ...)))))

(define string
  (memo
   (lambda (match)
     (memo-cps
      (lambda (str cont) ...)))))

(define seq
  (memo
   (lambda (a b)
     (memo-cps
      (bind a (lambda (x) ...))))))

(define alt
  (memo
   (lambda (a b)
     (memo-cps
      (lambda (str cont) ...)))))
```

Now we can define our left-recursive grammar:

```Scheme
(define-parser s
  (alt (seq s (string "a"))
       (string "a")))
```

Let's parse the string `"aaa"`:

```Scheme
> (s "aaa" print)
(success "a" "aa")
(success '("a" "a") "a")
(success '(("a" "a") "a") "")
(failure "")
```

We get three results before the parser reaches the end of the string
and terminates with a failure. This lets us see how the continuations
are repeated: when the self-reference in `(seq s ...)` is encountered,
the branch "goes to sleep" since `s` has been called before. Then the
second branch matches a single `"a"`. Since this is one of the results
of `s`, it is printed to standard output, but it is also broadcast to
the first branch. That branch matches another `"a"`, and the combined
sequence becomes another result for `s`. The result is again printed
and broadcast to the first branch, and so on, until we reach the end
of the string.

Let us now define a more convenient interface for invoking parsers.
The `run-parser` function runs a parser with a continuation that
collects all successful results in a list, which is then returned.
Only results that consume the whole string (with a remainder of `""`)
are collected.

```Scheme
(define (run-parser parser str)
  (let ((results '()))
    (parser str (lambda (result)
                  (match result
                    [(success val "")
                     (set! results (cons result results))]
                    [failure failure])))
    results))
```

We can implement an even simpler interface by exploiting the fact that
Racket allows functions to have *optional arguments*. Thus, we can
make the continuation argument optional! If the parser is invoked
without a continuation, then the default is to use the continuation of
`run-parser`. A wrapper for this interface can be defined as follows:

```Scheme
(define (make-parser parser)
  (lambda (str (cont #f))
    (if cont
        (parser str cont)
        (run-parser parser str))))
```

Then we can incorporate this wrapper into `define-parser`:

```Scheme
(define-syntax-rule (define-parser parser body)
  (define parser
    (make-parser
     (delay-parser body))))
```

Our parsers can now be invoked in two ways: as a CPS function passing
the results to a continuation, or as a regular function returning the
results to the caller. Here is an example of the latter:

```Scheme
> (s "aaa")
(list (success '(("a" "a") "a") ""))
```

The parser returns a list containing a single result matching the
whole input.

Trampoline
----------

A weakness of the current implementation is that the memoized parser
results are scattered all over the place. Each parser has its own
memoization table, storing the accumulated results from both the
current parsing and from previous parsings. This is difficult to
maintain and optimize.

Another issue is that when dealing with ambiguous grammars, all the
results are produced at once. For such grammars, it would be more
flexible to return a lazy stream of results, producing results one at
a time. (An infinitely ambiguous grammar may produce infinitely many
results!)

To achieve this, we will encapsulate parser results and parser calls
in a shared data structure called a *trampoline*. The trampoline
contains a loop that iterates through parser calls and dispatches
parsers. Each parser will have an extra `tramp` argument for the
trampoline.

We will define the trampoline as a Racket class, with fields and
methods. This is just for convenience; we could also piece together a
mutable list structure from scratch, like we did with the memoization
tables. In the end, the trampoline is just a stateful object passed
down from one parser to another.

Here is an outline of our `trampoline%` class (by convention, class
names end with `%`):

```Scheme
(define trampoline%
  (class object% (super-new)
    (define stack (mlist))
    (define table (mlist))

    (define/public (has-next?) ...)
    (define/public (step) ...)
    (define/public (push-stack fn . args) ...)
    (define/public (push fn arg continuation) ...)
    (define/public (run) ...)))
```

The trampoline contains two fields, the *stack* and the *table*. The
stack contains function calls, while the table contains memoized
values. Both are mutable lists, modified by the public methods
`has-next?`, `step`, `push-stack`, `push` and `run`.

The parsers will be modified to save their execution on the call
stack. That is, instead of calling a parser directly, the parser call
is pushed onto the stack. The trampoline loop then iterates through
the stack until it is exhausted, checking with the `has-next?` method.
This method returns true if the stack is nonempty and false if it is
empty.

```Scheme
(define/public (has-next?)
  (not (empty? stack)))
```

The `push-stack` method pushes a function call onto the stack. The
call is a cons cell `(fn . args)`, containing a function and its
arguments.

```Scheme
(define/public (push-stack fn . args)
  (let ((call (mcons fn args)))
    (set! stack (mcons call stack))))
```

The `step` method pops a parser call off the stack and invokes it. We
obtain the first element with `(mcar stack)`, matching against it to
obtain the function and its arguments. We advance the stack pointer to
the next element, and then apply the function to its arguments with
`apply`.

```Scheme
(define/public (step)
  (when (has-next?)
    (match (mcar stack)
      [(mcons fn args)
       (set! stack (mcdr stack))
       (apply fn args)])))
```

The `run` method repeatedly invokes `step` until the stack is
exhausted.

```Scheme
(define/public (run)
  (do () ((not (has-next?)))
    (step)))
```

The other part of the trampoline is the memoization table, where every
parser caches its results. The memoization logic is contained in the
`push` method, which works as a memoizing front-end for `push-stack`.
It is similar to the `memo-cps` function from earlier, except that it
operates on a two-level table. Note that instead of invoking the
function directly when called for the first time, the function is
passed to `push-stack`.

```Scheme
(define/public (push fn str cont)
  (define entry-continuations ...)
  (define entry-results ...)
  (define (push-continuation! entry cont) ...)
  (define (push-result! entry result) ...)
  (define (result-subsumed? entry result) ...)
  (define (make-entry) ...)
  (define (table-ref fn str) ...)
  (let ((entry (table-ref fn str)))
    (match entry
      [(mcons (mlist) (mlist))
       (push-continuation! entry cont)
       ;; push the parser on the stack
       (push-stack fn str this
                   (lambda (result)
                     (unless (result-subsumed? entry result)
                       (push-result! entry result)
                       (for ((cont (entry-continuations entry)))
                            (cont result)))))]
      [_
       (push-continuation! entry cont)
       (for ((result (entry-results entry)))
            (cont result))])))
```

As mentioned, the table has two levels (it is a nested association
list). The first level maps parsers to memoization records, and the
second level maps input to output. This is all handled by the local
`table-ref` function, which automatically creates an empty entry when
a function or its input is referenced for the first time.

```Scheme
(define (table-ref fn str)
  (let ((pair (massoc fn table)))
    (match pair
      [(mcons fn memo)
       (match (massoc str memo)
         ;; parser has been called with str before
         [(mcons str entry) entry]
         ;; first time parser has been called with str
         [_ (let ((entry (make-entry)))
              (set-mcdr! pair (mcons (mcons str entry) memo))
              entry)])]
      ;; first time parser has been called
      [_ (let* ((entry (make-entry))
                (memo (mlist (mcons str entry))))
           (set! table (mcons (mcons fn memo) table))
           entry)])))
```

We are now ready to rewrite the parsers. For the most part, this is
just a matter of adding an extra argument, `tramp`, for the
trampoline. The `succeed` and `string` parsers don't use the
trampoline at all:

```Scheme
(define succeed
  (memo
   (lambda (val)
     (lambda (str tramp cont)
       (cont (success val str))))))

(define string
  (memo
   (lambda (match)
     (lambda (str tramp cont)
       (let* (...)
         (if (equal? head match)
             (cont (success head tail))
             (cont (failure tail))))))))
```

Note the absence of `memo-cps`, since parser memoization is handled by
the trampoline. The `seq` combinator just passes the `tramp` argument
down from one parser to another:

```Scheme
(define (bind p fn)
  (lambda (str tramp cont)
    (p str tramp
       (lambda (result)
         (match result
           [(success val rest)
            ((fn val) rest tramp cont)]
           [failure
            (cont failure)])))))

(define seq
  (memo
   (lambda (a b)
     (bind a (lambda (x) ...)))))
```

As before, the changes are in `bind`, which passes the trampoline
around. The definition of `seq` stays the same, except that `memo-cps`
is gone.

Only the `alt` combinator will use the trampoline directly. Instead of
invoking the alternative parsers itself, it pushes them on the stack:

```Scheme
(define alt
  (memo
   (lambda (a b)
     (lambda (str tramp cont)
       (send tramp push a str cont)
       (send tramp push b str cont)))))
```

The `send` function is Racket's way of accessing the methods of a
object. To invoke the `push` method of the `tramp` object, we write
`(send tramp push ...)` and the method's arguments.

Now we can create a trampoline, pass it to a parser along with a
continuation, and invoke the `run` method:

```Scheme
> (define tramp (new trampoline%))
> (s "aaa" tramp print)
> (send tramp run)
(success "a" "aa")
(success '("a" "a") "a")
(success '(("a" "a") "a") "")
(failure "")
```

Of course, this is not a very convenient interface, so we will
redefine the `run-parser` function to return the successful results as
a lazy stream. To that end, we first define a `make-stream`
convenience macro, which lets us use the stream constructor
`stream-cons` in a simpler manner. The `stream-cons` function takes
two expressions, one for producing the first element and one for
producing the rest of the stream. Our `make-stream` macro just takes a
single expression for producing the stream, which is easier in case
multiple results are produced.

```Scheme
(define-syntax-rule (make-stream body ...)
  (stream-rest
   (stream-cons '() (begin body ...))))
```

Now we can define `run-parser` as a call to `make-stream`:

```Scheme
(define (run-parser parser str)
  (let ((tramp (new trampoline%))
        (results '()))
    (define (compute)
      (when (send tramp has-next?)
        (do () ((not (and (empty? results)
                          (send tramp has-next?))))
          (send tramp step)))
      (stream))
    (define (stream)
      (let ((result (sequence->stream results)))
        (set! results (mlist))
        (if (send tramp has-next?)
            (stream-append result (make-stream (compute)))
            result)))
    (make-stream
     (parser str tramp
             (lambda (result)
               (match result
                 [(success val "")
                  (set! results (cons result results))]
                 [failure failure])))
     (compute))))
```

First, we create a new trampoline and an empty `results` list. After
defining a few local functions, we make a stream that invokes the
parser with a continuation that will add successful results to
`results`. The local function `compute` is invoked to step through the
trampoline's stack until at least one result shows up in `results` or
the stack is exhausted. The results are returned lazily by calling the
local `stream` function, which creates a stream where the first
results are taken from `results`, and the rest are produced by calling
`compute` again.

In other words, instead of stepping through the whole call stack at
once with the `run` method, `run-parser` performs a more fine-grained
execution with `step`. Whenever it obtains a result, it stops
execution, unless forced to produce more (and the call stack is not
exhausted).

Finally, we can create a cleaner interface where the `tramp` and
`cont` arguments are optional:

```Scheme
(define-syntax-rule (define-parser parser body)
  (define parser
    (make-parser
     (delay-parser body))))

(define (make-parser parser)
  (lambda (str (tramp #f) (cont #f))
    (if (and tramp cont)
        (parser str tramp cont)
        (run-parser parser str))))
```

When we now invoke our parsers in the regular way, we get a stream:

```Scheme
> (s "aaa")
#<stream>
```

We can force the stream by converting it to a list:

```Scheme
> (stream->list (s "aaa"))
(list (success '(("a" "a") "a") ""))
```

Final improvements
------------------

As defined, the `alt` and `seq` combinators only take two arguments:
we can write `(alt a b)`, but not `(alt a b c)`. We can generalize
`alt` by iterating over a rest argument `parsers`:

```Scheme
(define alt
  (memo
   (lambda parsers
     (lambda (str tramp cont)
       (for ((fn parsers))
            (send tramp push fn str cont))))))
```

Generalizing `seq` is most easily done with a fold. In essence, we
take the binary definition and rename it to a local function `seq2`,
swap the arguments around, and then use it to reduce the list of
parsers to a single value. We also adjust the way the combined result
is created so that we get a flat list `(val1 val2 val3)` instead of
`((val1 val2) val3)`.

```Scheme
(define seq
  (memo
   (lambda parsers
     (define (seq2 b a)
       (bind a (lambda (x)
                 (bind b (lambda (y)
                           (succeed (append x (list y))))))))
     (foldl seq2 (succeed '()) parsers))))
```

Now we will add some new definitions. First we define the `regexp`
parser, which is similar to `string` except that it matches against a
regular expression:

```Scheme
(define regexp
  (memo
   (lambda (pattern)
     (lambda (str tramp cont)
       (match (regexp-match-positions (string-append "^" pattern) str)
         [(cons (cons beg end) _)
          (let* ((len (string-length str))
                 (head (substring str beg end))
                 (tail (substring str end len)))
            (cont (success head tail)))]
         [_ (cont (failure str))])))))
```

This lets us define terminal parsers in a simpler way: for example,
`(regexp "[0-9]+")` matches a whole number.

Furthermore, we will provide semantic actions for our parsers by
adding a new combinator, the *reduction combinator*. This combinator
transforms the parse result using a given function.

```Scheme
(define red
  (memo
   (lambda (p fn)
     (bind p (lambda (val)
               (match val
                 [(list val ...) (succeed (apply fn val))]
                 [_ (succeed (fn val))]))))))
```

For example, we can write `(red (regexp "[0-9]+") string->number)` to
convert the match string to an actual number. The arity of the
supplied function must match the value it is applied to.

We can now define an interpreter for the following grammar for
arithmetic expressions:

```
  expr -> expr "+" term
        | expr "-" term
        | term
  term -> term "*" factor
        | term "/" factor
        | factor
factor -> "(" expr ")"
        | num
   num -> "[0-9]+"
```

Using anonymous functions for semantic actions, we can express the
interpreter as follows:

```Scheme
(define-parser expr
  (alt (red (seq expr (string "+") term)
            (lambda (x _ y) (+ x y)))
       (red (seq expr (string "-") term)
            (lambda (x _ y) (- x y)))
       term))

(define-parser term
  (alt (red (seq term (string "*") factor)
            (lambda (x _ y) (* x y)))
       (red (seq term (string "/") factor)
            (lambda (x _ y) (/ x y)))
       factor))

(define-parser factor
  (alt (red (seq (string "(") expr (string ")"))
            (lambda (_ x __) x))
       num))

(define-parser num
  (red (regexp "[0-9]+")
       string->number))
```

When we invoke the interpreter, the returned result contains the
calculated value:

```Scheme
> (stream->list (expr "1*2+3*4"))
(list (success 14 ""))
> (stream->list (expr "9-(5+2)"))
(list (success 2 ""))
```

Since the grammar is unambiguous, each expression only has one value.

Further reading
---------------

This concludes the article. Several improvements are possible from
here. For example, one could specify a monadic interface for composing
parsers. Racket's macro system could be used to define an even simpler
DSL. Branches in the grammar could be parallelized by using a monitor
for the trampoline. Finally, one can optimize the combinators by
adorning the parsers with metadata for calculating the FIRST and
FOLLOW sets (Spiewak).

The code can be ported to an object-oriented language by creating a
`Parser` interface and composing parser objects. Continuations can be
implemented as functor objects if the language lacks first-order
functions. The trampoline becomes another class. Some languages, like
Ruby and Scala, offer facilities for creating a terse, DSL-like
syntax.

For more information, follow the references:

* [Memoization in Top-Down Parsing](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.14.3000),
  Mark Johnson, Brown University, 1995. Published in *Computational
  Linguistics*, Volume 21, Number 3. Covers regular memoization,
  continuation-passing style, and memoization of continuation-passing
  style functions.
* [Generalized Parser Combinators](http://www.cs.uwm.edu/~dspiewak/papers/generalized-parser-combinators.pdf),
  Daniel Spiewak, University of Wisconsin, 2010. Implemented as the
  [gll-combinators](https://github.com/djspiewak/gll-combinators)
  Scala library, using continuation-passing style and trampolined
  dispatch. Offers a very accessible introduction to the GLL
  algorithm.
* [Parsing Techniques: A Practical Guide](http://dickgrune.com/Books/PTAPG_2nd_Edition/)
  (Second Edition), Dick Grune and Ceriel J. H. Jacobs,
  Springer, 2008. Chapter 11 contains a richly illustrated
  description of generalized LL parsing.
* [GLL Parsing](http://ldta.info/2009/ldta2009proceedings.pdf),
  Adrian Johnstone and Elizabeth Scott, University of London, 2009.
  Published in *Proceedings of LDTA*. Explains the GLL algorithm in
  abstract terms.
* [Modelling GLL Parser Implementations](http://link.springer.com/chapter/10.1007%2F978-3-642-19440-5_4),
  Adrian Johnstone and Elizabeth Scott, University of London, 2011.
  Lecture Notes in *Computer Science*, Volume 6563. Models an
  implementation of the GLL algorithm in a theoretical language.

Comments? Suggestions? Post them at the
[bug tracker](https://github.com/epsil/gll/issues)!

---

[Vegard Øye](https://github.com/epsil) | 2012
