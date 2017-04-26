# Recursion that doesn't need so many stack frames
by [Yi-Xian Chen(Ethan)](#); from the series [Six different styles to solve one programming problem](//github.com/ethan-yxchen/Programming_Style)

__Concepts: depth of recursion, stack frame, higher order function, tail recursion__<br>
__Language feature: closure, rest operator (javascript), argument unpacking (Python)__

"To iterate is human; to recurse, divine." - L. Peter Deutsch.

Here I propose a transformation strategy, a wrapper, to solve the stack size problem 
where tail recursion elimination (TRE) may not be supported or not applicable.

It transforms a single recursive function to a new function
in a way that much less stack is needed. It needs only O(log n), 
where n is the levels of stack required for the original recursive function. 

This transformation is applicable as long as the input of the problem 
can be sliced into chunks and processed chunk-by-chunk.

In the book *Exercises in Programming Style*, 
the 7th style implements word count program by recursion, instead of iteration;
and it hits the stack size limitation of Python.
I will demonstrate how to transform this to [another recursive solution](#final-solution-logn-levels-of-stack), which turns out to require only O(log n) levels of stack.
I use higher order function to make it a general solution.
But beforehand, let's first briefly survey the background story.

## Recursion and iteration: different mental activity in problem-solving

Recursion is a programming strategy to solve a problem by solving one or 
several smaller instances of the same problem and add the last tip onto 
the solution. 

One problem can have both pure-recursive solutions and pure-iterative solutions.
Each of them provides different **insight** to the nature of the problem.

For example, consider a program calculating sum of natural numbers up to _n_. Here is an
iterative implementation in C++:

```c++
int sum_of_natural ( int n ) {
    int answer = 0;
    for ( int i = 1; i <= n; i++ )
        answer += i;
    return answer;
}
```

The semantic of this iterative implementation is: the line "*answer += i;* "
repeats (iterates) *n* times. We can compare it to the recursive version:

```c++
int sum_of_natural ( int n ) {
    if ( n == 1 )
        return 1;
        
    return sum_of_natural( n - 1 ) + n;
}
```

which can be read as the inductive definition: 
*f(n) = f(n-1) + n where f(1) = 1*.

## Cost of recursion

However, function calls are not necessarily free or cheap
in some compilers and/or languages. 
Hence recursion can be expensive, as compared to iteration.
If without optimization, at least it will 
cost you the space of one stack frame for each level of recursion. 
On the other hand, iterative version only needs a conditional jump 
(let's focus on single recursion and leave multiple recursion to another article).


Regarding to this cost, *tail call optimization* (TCO) is very helpful 
for those recursive functions whose recursive function call only be invoked as a tail call,
so that the number of stack frame does not grow. *Tail recursion elimination* (TRE) is a special
case of TCO where the *caller* and *callee* have the same layout of local variables in its stack frame,
hence is easier than general TCO and often more efficient.

The former example can be re-written into tail-recursive:

```c++
int sum_of_natural ( int n, int partial = 0 ) {
    if ( n == 1 )
        return partial + 1;
        
    return sum_of_natural( n - 1, partial + n );
}
```

If we run *sum_of_natural(4, 0)*, the trace would be like this pseudo code
(each level of indent is one level of stack frame):
```
push 4, push 0, call sum_of_natural
    push 3, push 4, call sum_of_natural
        push 2, push 7, call sum_of_natural
            push 1, push 9, call sum_of_natural
            return 10
        return 10
    return 10
return 10
```

with TRE, it would become
```
push 4, push 0, call sum_of_natural
    calculate (4-1, 0+4) -> (3, 4)
    replace (4, 0) with (3, 4), call sum_of_natural
    calculate (3-1, 4+3) -> (2, 7)
    replace (3, 4) with (2, 7), call sum_of_natural
    calculate (2-1, 7+2) -> (1, 9)
    replace (2, 7) with (1, 9), call sum_of_natural
    return 10
return 10
```

in the later case, the stack won't keep growing as the recursion goes.

If without TCO/TRE support, 
the number of levels of recursion is limited, otherwise it will overflow the stack.
It depends on either language specification or implementation of compiler/JIT.

For C/C++, both gcc and clang support TCO; for javascript, TCO is part of standard in ECMAScript 6,
currently WebKit supports TCO but not does V8 and Mozilla (at this time of writing). 
Theoretically it is possible to implement TRE in JAVA since JVM has a *goto* bytecode, but neither Oracle nor openjdk implements TRE. 
None of mainstream Python interpreter supports TCO; Guido van Rossum, the creator of Python,
has the opinion that TCO breaks the stack trace and hence is unwantted.

On the other hand, recursion is a natural style in LISP and is the only option in Prolog. 
This kind of languages usually support TCO/TRE; Prolog has had TCO since its early days decades ago.

## The problem

In the chapter 7 of *Exercises in Programming Style*, *Infinite Mirror*, 
we can see an example of such problem in Python. 

The story of this chapter is to go through a list by processing one item in the list
and then recursively calling the same function with rest of the list.
Without TCO/TRE, it will require levels of stack frame equal to the number of tokens.
For the given test input, the full text of *Pride and Prejudice*, the number is 124,588.

We can easily implement the recursive *count()* function in javascript like this:

```javascript
function count( tokens, stopwords, freq ) {
    if ( tokens.length == 0 )
        return freq;

    const token = tokens.pop();
    if ( !(token in stopwords) ) {
        freq[token] = (freq[token] || 0) + 1;
    }
    
    return count( tokens, stopwords, freq );
}
```

This function is tail-recursive. 
Unfortunately, V8 5.4.500 / nodejs 7.4.0 does not implement TRE yet.
If you [test it](#) with *Pride and Prejudice*, you will soon get an

```
RangeError: Maximum call stack size exceeded
```

## First transformation: batch processing

Let's start with the chunk-by-chunk approach. 
*Exercises in Programming Style* provides a similar workaround, but they implement it with iteration 
[(here, line #40)](https://github.com/crista/exercises-in-programming-style/blob/master/07-infinite-mirror/tf-07.py#L40).
Instead, we can do it in a recursive style.
And we can make it __generic__ by writing a higher order function, takes a straightforward recursive function such as
our naive *count()*, and return a trasformed function with the same signature, which use less stack frames. 

Here we assume the first argument of the recursive function to be the array that can be sliced into chunks.

```javascript
// func is the original function we want to transform
var batch_processing = function ( batch_size, func ) {
    var batch = function( ...args ) {

        var first = args[0], rest = args.slice(1);
        
        if ( first.length <= batch_size ) {
            func( ...args );
        } else {
            const last_part = first.length - batch_size;
            
            // call func() with the last part of the data
            func( first.slice( last_part ), ...rest );            
            // recursive call to batch() itself
            batch( first.slice(0, last_part), ...rest );
        }
    };

    return batch;
};
```

*batch_processing* is the trasformer. We use it like this:

```javascript
count = batch_processing( 100, count );
// count is replaced the new function, with identical 
// interface and functionality as count, but much efficient!
count( words, load_stopwords(), freq );
```

Here provides a purely recursive solution:
this transformed function *batch()* itself is also recursive, 
on the contrary to the iterative implementation in *Exercises in Programming Style* --
which is a deviation from the principle of that chapter.

## Optimal size of batch

Now, how many levels of stack frames do we need to run the recursion, given the input a list of 124,588 tokens?
The answer is *batch_size* levels of *count()* for each chunk, plus *(124588 / batch_size)* levels of *batch()*.
Minimum of *f(x) = x + n/x* is *2\*sqrt(n)* , at *x = sqrt(n)*.
So the optimal batch size can be deteremined when the size of input is given. We can revise our *batch_processing()*:

```javascript
var batch_processing2 = function ( func ) {
    var f = function( ...args ) {
        var batch_size = Math.sqrt( args[0].length );
        
        var batch = function( ...args ) {
            var first = args[0], rest = args.slice(1);
            if ( first.length <= batch_size ) {
                func( ...args );
            } else {
                const last_part = first.length - batch_size;
                
                // call func() with the last part of the data
                func( first.slice( last_part ), ...rest );
                // recursive call to batch() itself
                batch( first.slice(0, last_part), ...rest );
            }
        };
        
        batch( ...args );
    };    

    return f;
};
```

When we want to execute *count()*, we can get a transformed version of *count()* before we go:

```javascript
count = batch_processing2( count );
count( words, load_stopwords(), freq );
```

## Going even higher order

Here we are in the half way hiking to the summit. *O( sqrt(n) )* levels of stack frames, not bad, right?

If we are using a small batch size, say, *x*, the recursion depth of *count()* will be small,
but the recursion depth of *batch()*, which is *n/x*, will be large. 
However, we can transform the transformed function again! For example:

```javascript
var batch_processing = function ( batch_size, func ) {
    var batch = function( ...args ) {
        var first = args[0], rest = args.slice(1);
        
        if ( first.length <= batch_size ) {
            func( ...args );
        } else {
            const last_part = first.length - batch_size;
            func( first.slice( last_part ), ...rest );            
            batch( first.slice(0, last_part), ...rest );
        }
    };

    return batch;
};

var b1 = batch_processing( 50, count ); 
// each batch we pass 50 items to count()

var b2 = batch_processing( 3000, b1 );  
// each batch we pass 3000 items to b1; 
// b1 has recursion of 3000/50 = 60 levels

b2( words, load_stopwords(), f );
```

In this example, we will need 50 + 60 + 124588/3000 ~ 152 levels of stack frames.
Much less than 2 * sqrt(124588) ~ 706 frames.

We can repeat this transformation *p* times with batch sizes of 
*b, b<sup>2</sup>, b<sup>3</sup>, ... b<sup>p</sup>*
until *b<sup>p</sup> >= n*, where *n* is the size of input.

Given the constraint: *b*, *p* are integers; *p =* ⌈ *log<sub>b</sub> n* ⌉.
We want to minimize the total number of levels of stack frames, which is *b\*p*. 

Then,

*b\*p = b\** ⌈ *log<sub>b</sub> n* ⌉

*\>= log(n)\*b / log(b)*

and we know that for integer *b*, *b/log(b)* has minimum at *b* = 3. It is independent to *n*.

Thus base *b = 3*, power *p =* ⌈ *log<sub>3</sub>(n)* ⌉ gives the optimal configuration.

## Final solution: *log(n)* levels of stack

Here comes the final solution, with a higher order function 
that generates and configurates higher order functions.
And this even-higher-order function is recursive, too!

```javascript
var super_batch_processing = function ( func ) {
    var f = function( ...args ) {
        var data_size = args[0].length;
        var base = 3;

        // Here it is the higher-higher-order transformer.
        // It recursively use batch_processing() to wrap the
        // target function, with batch_size growing exponentially
        var transform = function ( batch_size, func ) {
            if ( batch_size >= data_size ) {
                return func;
            }
            
            var transformed = batch_processing( batch_size, proc );
            return transform( batch_size * base, transformed );
        };
        
        return transform( base, func )( ...args );
    };
    
    return f;
};

```

Now we have a *batch_processing* transformer, which transforms the original function to batch-wise processing function; 
and a *super_batch_processing*, which applies the transformation multiple times. 

Purely recursive, no for-loops neither while-loops, and (virtually) never overflows your stack -- 
for *Pride and Prejudice*, the number of levels of stack frames used for *batch()* and *count()* is only 33;
even for a input of length 2<sup>64</sup>, it needs only 122 stack frames.

## Comment

This strategy is implemented as a wrapper function, and there is no need to 
modify the compiler/interpreter. I write example of generic wrappers in Python
and in javascript. Two features make it easier to write this wrapper more __genericly__: one is closure, 
and the other is argument unpacking (in javascript, the similar concept is the rest operator).


## Appendix: auxilary functions for testing this code

enter()
leave()
wrapper()
