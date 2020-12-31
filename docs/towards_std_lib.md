# Towards a Standard Library: Fold and Unfold
Our Scheme is almost complete now, but it's still rather hard to use. At the very least, we'd like a library of standard list manipulation functions that we can use to perform some common computations.

Rather than using a typical Scheme implementation, defining each list function in terms of a recursion on lists, we'll implement two primitive recursion operators (fold and unfold) and then define our whole library based on those. 

First create a file `stdlib.scm`.
We'll start by defining a few obvious helper functions. not and null are defined exactly as you'd expect it, using if statements:

```scheme
(define (not x)            (if x #f #t))
(define (null? obj)        (if (eqv? obj '()) #t #f))
```

We can use the varargs feature to define list, which just returns a list of its arguments:
```scheme
(define (list . objs)       objs)
```
We also want an `id` function, which just returns its argument unchanged. This may seem completely useless – if you already have a value, why do you need a function to return it? However, several of our algorithms expect a function that tells us what to do with a given value. By defining `id`, we let those higher-order functions work even if we don't want to do anything with the value.

```scheme
(define (id obj)           obj)
```
Similarly, it'd be nice to have a `flip` function, in case we want to pass in a function that takes its arguments in the wrong order:

```scheme
(define (flip func)        (lambda (arg1 arg2) (func arg2 arg1)))
```

Finally, we add `curry` and `compose`, which works as partial application and the `>>` operator, respectively.
```scheme
(define (curry func arg1)  (lambda (arg) (apply func (cons arg1 (list arg)))))
(define (compose f g)      (lambda (arg) (f (apply g arg))))
```
We might as well define some simple library functions that appear in the Scheme standard:

```scheme
(define zero?              (curry = 0))
(define positive?          (curry < 0))
(define negative?          (curry > 0))
(define (odd? num)         (= (mod num 2) 1))
(define (even? num)        (= (mod num 2) 0))
```
These are basically done just as you'd expect them. Note the usage of `curry` to define `zero?`, `positive?` and `negative?`. We bind the variable `zero?` to the function returned by `curry`, giving us a unary function that returns true if its argument is equal to zero.

Next, we want to define a `fold` function that captures the basic pattern of recursion over a list. The best way to think about `fold` is to picture a list in terms of its infix constructors: [1; 2; 3; 4] = 1::2::3::4::[] in F# or (1 . (2 . (3 . (4 . NIL)))) in Scheme. A `fold` function replaces every constructor with a binary operation, and replaces `NIL` with the accumulator. So, for example, `(fold + 0 '(1 2 3 4))` = `(1 + (2 + (3 + (4 + 0))))`.

With that definition, we can write our `fold` function. Start with a right-associative version to mimic the above examples:

```scheme
(define (foldr func end lst)
  (if (null? lst)
      end
      (func (car lst) (foldr func end (cdr lst)))))
```

The structure of this function mimics our definition almost exactly. If the list is `null`, replace it with the `end` value. If not, apply the function to the car of the list and to the result of folding this function and end value down the rest of the list. Since the right-hand operand is folded up first, you end up with a right-associative fold.

We also want a left-associative version. For most associative operations like `+` and `*,` the two of them are completely equivalent. However, there is at least one important binary operation that is not associative: cons. For all our list manipulation functions, then, we'll need to deliberately choose between left- and right-associative folds.

```scheme
(define (foldl func accum lst)
  (if (null? lst)
      accum
      (foldl func (func accum (car lst)) (cdr lst))))
```

This begins the same way as the right-associative version, with the test for null that returns the accumulator. This time, however, we apply the function to the accumulator and first element of the list, instead of applying it to the first element and the result of folding the list. This means that we process the beginning first, giving us left-associativity. Once we reach the end of the list, `'()`, we then return the result that we've been progressively building up.

Note that func takes its arguments in the opposite order from `foldr`. In `foldr`, the accumulator represents the rightmost value to tack onto the end of the list, after you've finished recursing down it. In `foldl`, it represents the completed calculation for the leftmost part of the list. In order to preserve our intuitions about commutativity of operators, it should therefore be the left argument of our operation in `foldl`, but the right argument in `foldr`.

```scheme
(define fold foldl)
(define reduce foldr)
```
These are just new variables bound to the existing functions: they don't define new functions. Most Schemes call `fold` "`reduce`" or plain old "`fold`", and don't make the distinction between `foldl` and `foldr`. We define it to be `foldl`, which happens to be tail-recursive and hence runs more efficiently than `foldr` (it doesn't have to recurse all the way down to the end of the list before it starts building up the computation). Not all operations are associative, however; we'll see some cases later where we have to use `foldr` to get the right result.

Next, we want to define a function that is the opposite of `fold`. Given a unary function, an initial value, and a unary predicate, it continues applying the function to the last value until the predicate is true, building up a list as it goes along.

```scheme
(define (unfold func init pred)
  (if (pred init)
      (cons init '())
      (cons init (unfold func (func init) pred))))
```

As usual, our function structure basically matches the definition. If the predicate is true, then we cons a `'()` onto the last value, terminating the list. Otherwise, `cons` the result of unfolding the next value (func init) onto the current value.

In academic functional programming literature, folds are often called _catamorphisms_, unfolds are often called _anamorphisms_, and the combinations of the two are often called _hylomorphisms_. They're interesting because any for-each loop can be represented as a _catamorphism_. To convert from a loop to a `foldl`, package up all mutable variables in the loop into a data structure (records work well for this, but you can also use an algebraic data type or a list). The initial state becomes the accumulator; the loop body becomes a function with the loop variables as its first argument and the iteration variable as its second; and the list becomes, well, the list. The result of the `fold` function is the new state of all the mutable variables.

Similarly, every _for-loop (without early exits) can be represented as a hylomorphism_. The initialization, termination, and step conditions of a for-loop define an anamorphism that builds up a list of values for the iteration variable to take. Then, you can treat that as a for-each loop and use a catamorphism to break it down into whatever state you wish to modify.

Let's go through a couple examples. We'll start with typical `sum`, `product`, `and` `&` `or` functions:
```scheme
(define (sum . lst)         (fold + 0 lst))
(define (product . lst)     (fold * 1 lst))
(define (and . lst)         (fold && #t lst))
(define (or . lst)          (fold || #f lst))
```

These all follow from the definitions:

1. `(sum 1 2 3 4)` = `1 + 2 + 3 + 4 + 0` = `(fold + 0 '(1 . (2 . (3 . (4 . NIL)))))`
1. `(product 1 2 3 4)` = `1 * 2 * 3 * 4 * 1 `= `(fold * 1 '(1 . (2 . (3 . (4 . NIL)))))`
1. `(and #t #t #f)` = `#t && #t && #f && #t `= `(fold && #t '(#t . (#t . (#f . NIL))))`
1. `(or #t #t #f)` = `#t || #t || #f || #f `= `(fold || #f '(#t . (#t . (#f . NIL))))`

Since all of these operators are associative, it doesn't matter whether we use `foldr` or `foldl`. We replace the cons constructor with the operator, and the `nil` constructor with the identity element for that operator.

Next, let's try some more complicated operators. `max` and `min` find the maximum and minimum of their arguments, respectively:

```scheme
(define (max first . rest) (fold (lambda (old new) (if (> old new) old new)) first rest))
(define (min first . rest) (fold (lambda (old new) (if (< old new) old new)) first rest))
```

It's not immediately obvious what operation to fold over the list, because none of the built-ins quite qualify. Instead, think back to fold as a representation of a foreach loop. The accumulator represents any state we've maintained over previous iterations of the loop, so we'll want it to be the maximum value we've found so far. That gives us our initialization value: we want to start off with the leftmost variable in the list (since we're doing a `foldl`). Now recall that the result of the operation becomes the new accumulator at each step, and we've got our function. If the previous value is greater, keep it. If the new value is greater, or they're equal, return the new value. Reverse the operation for `min`.

How about `length? `We know that we can find the length of a list by counting down it, but how do we translate that into a `fold`?
```scheme
(define (length lst)        (fold (lambda (x y) (+ x 1)) 0 lst))
```

Again, think in terms of its definition as a loop. The accumulator starts off at 0 and gets incremented by 1 with each iteration. That gives us both our initialization value, 0, and our function, `(lambda (x y) (+ x 1))`. Another way to look at this is "The length of a list is 1 + the length of the sublist to its left".

Let's try something a bit trickier: `reverse`.

```scheme
(define (reverse lst)       (fold (flip cons) '() lst))
```

The function here is fairly obvious: if you want to reverse two cons cells, you can just `flip cons`so it takes its arguments in the opposite order. However, there's a bit of subtlety at work. Ordinary lists are right associative: `(1 2 3 4)` = `(1 . (2 . (3 . (4 . NIL))))`. If you want to reverse this, you need your fold to be left associative: `(reverse '(1 2 3 4))` = `(4 . (3 . (2 . (1 . NIL))))`. Try it with a `foldr` instead of a `foldl` and see what you get.

There's a whole family of `member` and `assoc` functions, all of which can be represented with folds. The particular lambda expression is fairly complicated though, so let's factor it out:

```scheme
(define (mem-helper pred op) (lambda (acc next) (if (and (not acc) (pred (op next))) next acc)))
(define (memq obj lst)       (fold (mem-helper (curry eq? obj) id) #f lst))
(define (memv obj lst)       (fold (mem-helper (curry eqv? obj) id) #f lst))
(define (member obj lst)     (fold (mem-helper (curry equal? obj) id) #f lst))
(define (assq obj alist)     (fold (mem-helper (curry eq? obj) car) #f alist))
(define (assv obj alist)     (fold (mem-helper (curry eqv? obj) car) #f alist))
(define (assoc obj alist)    (fold (mem-helper (curry equal? obj) car) #f alist))
```

The helper function is parameterized by the predicate to use and the operation to apply to the result if found. Its accumulator represents the first value found so far: it starts out with `#f` and takes on the first value that satisfies its predicate. We avoid finding subsequent values by testing for a non-`#f` value and returning the existing accumulator if it's already set. We also provide an operation that will be applied to the next value each time the predicate tests: this lets us customize `mem-helper`to check the value itself (for `member`) or only the key of the value (for `assoc`).

The rest of the functions are just various combinations of `eq?`, `eqv?`, `equal?` and `id`, `car`, folded over the list with an initial value of `#f`.

Next, let's define the functions `map` and `filter`. `map` applies a function to every element of a list, returning a new list with the transformed values:

```scheme
(define (map func lst)      (foldr (lambda (x y) (cons (func x) y)) '() lst))
```

Remember that `foldr's` function takes its arguments in the opposite order as `fold`, with the current value on the left. Map's lambda applies the function to the current value, then conses it with the rest of the mapped list, represented by the right-hand argument. It's essentially replacing every infix cons constructor with one that conses, but also applies the function to its left-side argument.

`filter` keeps only the elements of a list that satisfy a predicate, dropping all others:
```scheme
(define (filter pred lst)   (foldr (lambda (x y) (if (pred x) (cons x y) y)) '() lst))
```

This works by testing the current value against the predicate. If it's true, replacing cons with `cons`, i.e., don't do anything. If it's `false`, drop the `cons` and just return the rest of the list. This eliminates all the elements that don't satisfy the predicate, consing up a new list that includes only the ones that do.

We can use the standard library by starting up our Lisp interpreter and typing `(load "stdlib.scm")`:
```bash
$ dotnet run
Lisp>>>(load "stdlib.scm")
(lambda (func lst) ...)
Lisp>>>(map (curry + 2) '(1 2 3 4))
(3 4 5 6)
Lisp>>>(filter even? '(1 2 3 4))
(2 4)
Lisp>>>quit
$
```

There are many other useful functions that could go into the standard library, including `list-tail,` `list-ref,` `append`, and various string-manipulation functions. Try implementing them as folds. Remember, the key to successful fold programming is thinking only in terms of what happens on each iteration. Fold captures the pattern of recursion down a list, and recursive problems are best solved by working one step at a time.





