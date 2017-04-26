# Six different styles to solve one programming problem

The problem here is _Word Count_: given a input text file and a black list
of words (stopword), find the top-_N_ most frequent words in that text file
but not in the blacklist.


| Style         |Original Constraints   |Revised Constraints    |Design Features            |Language |
| :------------ | :-------------------- | :-------------------- | :------------------------ | :------ |
|Stack-friendly Recursion|- Infinite mirror <br>- Solve the problem with recursion      |- Even the language does **not** <br>support **tail recursion <br>optimization**, the usage of <br>stack frames should be less <br>than linear to input size  |- **high order function** <br>- high order decomposition of the problem <br>- only need __⌈*log<sub>3</sub>(n)*⌉__ levels of stack frames |javascript|


I wrote these six programs, each solves the same problem in a different style.

The origin of this exercise is from an interesting book, _Exercises in Programming Style_. 
This book is motivated by French writer Raymond Queneau's _Exercises in Style_,
which composed of 99 retellings of the same story with 99 different styles.

In the book _Exercises in Programming Style_, Crista Lopes illustrates 
many different ways to program a common computational task. Each way to 
solve the same problem shows a different decomposition of that problem, 
under a different set of constraints.

In my opinion, these constraints mainly come from the forever conundrum:
tradeoff between desired features must be made while resource is always finite. 
What do we want from a software program? Of course it need to be correct;
and we want it to be efficient (in terms of time and ammount of memory);
we also want it to be easy to understand; very often, we also want it to be 
easy to extend. The selection from sets of constraints will reflect the
decision of how to tradeoff those objectives.

Here, I provides a series of styles, based on 
the styles in _Exercises in Programming Style_, augmented with additional
constraints. I would like to make sense from the decision of each of these 
styles, and use them as platform to discuss several concepts in programming 
and programming languages, and see how the language features support the 
programming styles.





[//]:<>(the selection of constraints)

[//]:<>(constraints)
[//]:<>(resource)
[//]:<>(environment)
[//]:<>(efficiency)

[//]:<>(generalize)

[//]:<>(how different language or programming paradigm)
