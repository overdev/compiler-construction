# 2. Language and Syntax

Every language displays a structure called its grammar or syntax. For example, a correct sentence always consists of a subject followed by a predicate, correct here meaning well formed. This fact can be described by the following formula:

> sentence = subject predicate.

If we add to this formula the two further formulas

> subject = "John" | "Mary".
> predicate = "eats" | "talks".

then we define herewith exactly four possible sentences, namely

> John eats Mary eats
> John talks Mary talks

where the symbol | is to be pronounced as or. We call these formulas syntax rules, productions, or simply syntactic equations. Subject and predicate are syntactic classes. A shorter notation for the above omits meaningful identifiers:

```
S = AB. L = {ac, ad, bc, bd}
A = "a" | "b".
B = "c" | "d".
```

We will use this shorthand notation in the subsequent, short examples. The set `L` of sentences which can be generated in this way, that is, by repeated substitution of the left-hand sides by the right-hand sides of the equations, is called the language.

The example above evidently defines a language consisting of only four sentences. Typically, however, a language contains infinitely many sentences. The following example shows that an infinite set may very well be defined with a finite number of equations. The symbol ∅ stands for the empty sequence.

```
S = A. L = {∅, a, aa, aaa, aaaa, ... }
A = "a" A | ∅.
```

The means to do so is recursion which allows a substitution (here of `A` by `"a"A`) be repeated arbitrarily often.

Our third example is again based on the use of recursion. But it generates not only sentences consisting of an arbitrary sequence of the same symbol, but also nested sentences:

```
S = A. L = {b, abc, aabcc, aaabccc, ... }
A = "a" A "c" | "b".
```
It is clear that arbitrarily deep nestings (here of `A`s) can be expressed, a property particularly important in the definition of structured languages.
Our fourth and last example exhibits the structure of expressions. The symbols `E`, `T`, `F`, and `V` stand for expression, term, factor, and variable.
```
E = T | E "+" T.
T = F | T "*" F.
F = V | "(" E ")".
V = "a" | "b" | "c" | "d".
```
From this example it is evident that a syntax does not only define the set of sentences of a language, but also provides them with a structure. The syntax decomposes sentences in their constituents as shown in the example of Figure 2.1. The graphical representations are called structural trees or syntax trees.
