# 4 Analysis of Context-free Languages

## 4.1 The method of Recursive Descent

Regular languages are subject to the restriction that no nested structures can be expressed. Nested structures can be expressed with the aid of recursion only (see Chapter 2).

A finite state machine therefore cannot suffice for the recognition of sentences of context free languages. We will nevertheless try to derive a parser program for the third example in Chapter 2, by using the methods explained in Chapter 3. Wherever the method will fail - and it must fail - lies the clue for a possible generalization. It is indeed surprising how small the necessary additional programming effort turns out to be.

The construct
```
A = "a" A "c" | "b".
```
leads, after suitable simplification and the use of an IF instead of a CASE statement, to the following piece of program:
```
IF sym = "a" THEN
	next;
	IF sym = A THEN next ELSE error END ;
	IF sym = "c" THEN next ELSE error END
ELSIF sym = "b" THEN next
ELSE error
END
```
Here we have blindly treated the nonterminal symbol A in the same fashion as terminal symbols. This is of course not acceptable. The purpose of the third line of the program is to parse a construct of the form A (rather than to read a symbol A). However, this is precisely the purpose of our program too. Therefore, the simple solution to our problem is to give the program a name, that is, to give it the form of a procedure, and to substitute the third line of program by a call to this procedure. Just as in the syntax the construct A is recursive, so is the procedure A recursive:
```
PROCEDURE A;
	BEGIN
		IF sym = "a" THEN
		next; A;
		IF sym = "c" THEN next ELSE error END
		ELSIF sym = "b" THEN next
		ELSE error
	END
END A
```
The necessary extension of the set of translation rules is extremely simple. The only additional rule is:

> A parsing algorithm is derived for each nonterminal symbol, and it is formulated as a procedure carrying the name of the symbol. The occurrence of the symbol in the syntax is translated into a call of the corresponding procedure.

**Note**: this rule holds regardless of whether the procedure is recursive or not.

It is important to verify that the conditions for a deterministic algorithm are satisfied. This implies among other things that in an expression of the form
```
term0 | term1
```
the terms must not feature any common start symbols. This requirement excludes left recursion. If we consider the left recursive production
```
A = A "a" | "b".
```
we recognize that the requirement is violated, simply because `b` is a start symbol of `A` (`b` IN `first(A)`), and because therefore first(A"a") and first("b") are not disjoint. `"b"` is the common element.

The simple consequence is: left recursion can and must be replaced by repetition. In the example above `A = A "a" | "b"` is replaced by `A = "b" {"a"}`.

Another way to look at our step from the state machine to its generalization is to regard the latter as a set of state machines which call upon each other and upon themselves. In principle, the only new condition is that the state of the calling machine is resumed after termination of the called state machine. The state must therefore be preserved. Since state machines are nested, a stack is the appropriate form of store. Our extension of the state machine is therefore called a pushdown automaton. Theoretically relevant is the fact that the stack (pushdown store) must be arbitrarily deep. This is the essential difference between the finite state machine and the infinite pushdown automaton.

The general principle which is suggested here is the following: consider the recognition of the sentential construct which begins with the start symbol of the underlying syntax as the uppermost goal. If during the pursuit of this goal, that is, while the production is being parsed, a nonterminal symbol is encountered, then the recognition of a construct corresponding to this symbol is considered as a subordinate goal to be pursued first, while the higher goal is temporarily suspended. This strategy is therefore also called goal-oriented parsing. If we look at the structural tree of the parsed sentence we recognize that goals (symbols) higher in the tree are tackled first, lower goals (symbols) thereafter. The method is therefore called topdown parsing (Knuth, 1971; Aho and Ullman, 1977). Moreover, the presented implementation of this strategy based on recursive procedures is known as recursive descent parsing.

Finally, we recall that decisions about the steps to be taken are always made on the basis of the single, next input symbol only. The parser looks ahead by one symbol. A lookahead of several symbols would complicate the decision process considerably, and thereby also slow it down. For this reason we will restrict our attention to languages which can be parsed with a lookahead of a single symbol.

As a further example to demonstrate the technique of recursive descent parsing, let us consider a parser for EBNF, whose syntax is summarized here once again:
```
syntax = {production}.
production = identifier "=" expression "." .
expression = term {"|" term}.
term = factor {factor}.
factor = identifier | string | "(" expression ")" | "[" expression "]" | "{" expression
"}".
```
By application of the given translation rules and subsequent simplification the following parser results. It is formulated as an Oberon module:
```
MODULE EBNF;
	IMPORT Viewers, Texts, TextFrames, Oberon;
	CONST IdLen = 32;
		ident = 0; literal = 2; lparen = 3; lbrak = 4; lbrace = 5; bar = 6; eql = 7;
		rparen = 8; rbrak = 9; rbrace = 10; period = 11; other = 12;
	TYPE Identifier = ARRAY IdLen OF CHAR;
	VAR ch: CHAR;
		sym: INTEGER;
		lastpos: LONGINT;
		id: Identifier;
		R: Texts.Reader;
		W: Texts.Writer;
	
	PROCEDURE error(n: INTEGER);
		VAR pos: LONGINT;
	BEGIN pos := Texts.Pos(R);
		IF pos > lastpos+4 THEN (*avoid spurious error messages*)
			Texts.WriteString(W, " pos"); Texts.WriteInt(W, pos, 6);
			Texts.WriteString(W, " err"); Texts.WriteInt(W, n, 4); lastpos := pos;
			Texts.WriteString(W, " sym "); Texts.WriteInt(W, sym, 4);
			Texts.WriteLn(W); Texts.Append(Oberon.Log, W.buf)
		END
	END error;
	
	PROCEDURE GetSym;
	BEGIN ... (*see Chapter 3*)
	END GetSym;

	PROCEDURE record(id: Identifier; class: INTEGER);
	BEGIN (*enter id in appropriate list of identifiers*)
	END record;
	
	PROCEDURE expression;
		PROCEDURE term;
			PROCEDURE factor;
			BEGIN
				IF sym = ident THEN record(id, 1); GetSym
				ELSIF sym = literal THEN record(id, 0); GetSym
				ELSIF sym = lparen THEN
					GetSym; expression;
					IF sym = rparen THEN GetSym ELSE error(2) END
				ELSIF sym = lbrak THEN
					GetSym; expression;
					IF sym = rbrak THEN GetSym ELSE error(3) END
				ELSIF sym = lbrace THEN
					GetSym; expression;
					IF sym = rbrace THEN GetSym ELSE error(4) END
				ELSE error(5)
			END
			END factor;
		
		BEGIN (*term*) factor;
			WHILE sym < bar DO factor END
		END term;
		
	BEGIN (*expression*) term;
		WHILE sym = bar DO GetSym; term END
	END expression;
	
	PROCEDURE production;
	BEGIN (*sym = ident*) record(id, 2); GetSym;
		IF sym = eql THEN GetSym ELSE error(7) END ;
		expression;
		IF sym = period THEN GetSym ELSE error(8) END
	END production;

	PROCEDURE syntax;
	BEGIN
		WHILE sym = ident DO production END
	END syntax;

	PROCEDURE Compile*;
	BEGIN (*set R to the beginning of the text to be compiled*)
		lastpos := 0; Texts.Read(R, ch); GetSym; syntax;
		Texts.Append(Oberon.Log, W.buf)
	END Compile;

	BEGIN Texts.OpenWriter(W)
END EBNF.
```

## 4.2 Table-driven Top-down Parsing

The method of recursive descent is only one of several techniques to realize the top-down parsing principle. Here we shall present another technique: table-driven parsing.

The idea of constructing a general algorithm for top-down parsing for which a specific syntax is supplied as a parameter is hardly far-fetched. The syntax takes the form of a data structure which is typically represented as a graph or table. This data structure is then interpreted by the general parser. If the structure is represented as a graph, we may consider its interpretation as a traversal of the graph, guided by the source text being parsed.

First, we must determine a data representation of the structural graph. We know that EBNF contains two repetitive constructs, namely sequences of factors and sequences of terms. Naturally, they are represented as lists. Every element of the data structure represents a (terminal) symbol. Hence, every element must be capable of denoting two successors represented by pointers. We call them next for the next consecutive factor and alt for the next alternative term. Formulated in the language Oberon, we declare the following data types:
```
Symbol = POINTER TO SymDesc;
SymDesc = RECORD alt, next: Symbol END
```
Then formulate this abstract data type for terminal and nonterminal symbols by using
Oberon's type extension feature (Reiser and Wirth, 1992). Records denoting terminal symbols
specify the symbol by the additional attribute sym:
```
Terminal = POINTER TO TSDesc;
TSDesc = RECORD (SymDesc) sym: INTEGER END
```
Elements representing a nonterminal symbol contain a reference (pointer) to the data structure representing that symbol. Out of practical considerations we introduce an indirect reference: the pointer refers to an additional header element, which in turn refers to the data structure. The header also contains the name of the structure, that is, of the nonterminal symbol. Strictly speaking, this addition is unnecessary; its usefulness will become apparent later.
```
Nonterminal = POINTER TO NTSDesc;
NTSDesc = RECORD (SymDesc) this: Header END
Header = POINTER TO HDesc;
HDesc = RECORD sym: Symbol; name: ARRAY n OF CHAR END
```
As an example we choose the following syntax for simple expressions. Figure 4.1 displays the corresponding data structure as a graph. Horizontal edges are next pointers, vertical edges are alt pointers.
```
expression = term {("+" | "-") term}.
term = factor {("*" | "/") factor}.
factor = id | "(" expression ")" .
```
Now we are in a position to formulate the general parsing algorithm in the form of a concrete procedure:
```
PROCEDURE Parsed(hd: Header): BOOLEAN;
	VAR x: Symbol; match: BOOLEAN;
BEGIN x := hd.sym; Texts.WriteString(Wr, hd.name);
	REPEAT
		IF x IS Terminal THEN
			IF x(Terminal).sym = sym THEN match := TRUE; GetSym
			ELSE match := (x = empty)
			END
		ELSE match := Parsed(x(Nonterminal).this)
		END ;
		IF match THEN x := x.next ELSE x := x.alt END
	UNTIL x = NIL;
	RETURN match
END Parsed;
```
![Syntax as data strucure](https://github.com/overdev/compiler-construction/blob/master/images/cc_figure_4_1.png)

The following remarks must be kept in mind:

1. We tacitly assume that terms always are of the form `T = f0 | f1 | ... | fn` where all factors except the last start with a distinct, terminal symbol. Only the last factor may start with either a terminal or a nonterminal symbol. Under this condition is it possible to traverse the list of alternatives and in each step to make only a single comparison.
2. The data structure can be derived from the syntax (in EBNF) automatically, that is, by a program which compiles the syntax.
3. In the procedure above the name of each nonterminal symbol to be recognized is output. The header element serves precisely this purpose.
4. Empty is a special terminal symbol and element representing the empty sequence. It is needed to mark the exit of repetitions (loops).

## 4.3 Bottom-up Parsing

Both the recursive-descent and table-driven parsing shown here are techniques based on the principle of top-down parsing. The primary goal is to show that the text to be analysed is derivable from the start symbol. Any nonterminal symbols encountered are considered as subgoals. The parsing process constructs the syntax tree beginning with the start symbol as its root, that is, in the top-down direction.

However, it is also possible to proceed according to a complementary principle in the bottomup direction. The text is read without pursuit of a specific goal. After each step a test checks whether the read subsequence corresponds to some sentential construct, that is, the right part of a production. If this is the case, the read subsequence is replaced by the corresponding nonterminal symbol. The recognition process again consists of consecutive steps, of which there are two distinct kinds:

1. Shifting the next input symbol into a stack (shift step),
2. Reducing a stacked sequence of symbols into a single nonterminal symbol according to a production (reduce step).

Parsing in the bottom-up direction is also called shift-reduce parsing. The syntactic constructs are built up and then reduced; the syntax tree grows from the bottom to the top (Knuth, 1965; Aho and Ullman, 1977; Kastens, 1990).

Once again, we demonstrate the process with the example of simple expressions. Let the syntax be as follows:
```
E = T | E "+" T. expression
T = F | T "*" F. term
F = id | "(" E ")". factor
```
and let the sentence to be recognized be `x * (y + z)`. In order to display the process, the remaining source text is shown to the right, whereas to the left the - initially empty - sequence of recognized constructs is listed. At the far left, the letters `S` and `R` indicate the kind of step taken
```
          x * (y + z)
S    x      * (y + z)
R    F      * (y + z)
R    T      * (y + z)
S    T *      (y + z)
S    T * (     y + z)
S    T * (y      + z)
R    T * (F      + z)
R    T * (T      + z)
R    T * (E      + z)
S    T * (E +      z)
S    T * (E + z     )
R    T * (E + F     )
R    T * (E + T     )
R    T * (E         )
S    T * (E)
R    T * F
R    T
R    E
```
At the end, the initial source text is reduced to the start symbol `E`, which here would better be called the stop symbol. As mentioned earlier, the intermediate store to the left is a stack.

In analogy to this representation, the process of parsing the same input according to the topdown principle is shown below. The two kinds of steps are denoted by `M` (match) and `P` (produce, expand). The start symbol is `E`.
```
	  E         x * (y + z)
P   T         x * (y + z)
P   T* F      x * (y + z)
P   F  * F    x * (y + z)
P   id * F    x * (y + z)
M      * F      * (y + z)
M        F        (y + z)
P       (E)       (y + z)
M        E)        y + z)
P        E + T)    y + z)
P        T + T)    y + z)
P        F + T)    y + z)
P       id + T)    y + z)
M          + T)      + z)
M            T)        z)
P            F)        z)
P           id)        z)
M             )         )
M
```
Evidently, in the bottom-up method the sequence of symbols read is always reduced at its right end, whereas in the top-down method it is always the leftmost nonterminal symbol which is expanded. According to Knuth the bottom-up method is therefore called LR-parsing, and the top-down method LL-parsing. The first L expresses the fact that the text is being read from left to right. Usually, this denotation is given a parameter k (LL(k), LR(k)). It indicates the extent of the lookahead being used. We will always implicitly assume k = 1.

Let us briefly return to the bottom-up principle. The concrete problem lies in determining which kind of step is to be taken next, and, in the case of a reduce step, how many symbols on the stack are to be involved in the step. This question is not easily answered. We merely state that in order to guarantee an efficient parsing process, the information on which the decision is to be based must be present in an appropriately compiled way. Bottom-up parsers always use tables, that is, data structured in an analogous manner to the table-driven top-down parser presented above. In addition to the representation of the syntax as a data structure, further tables are required to allow us to determine the next step in an efficient manner. Bottom-up parsing is therefore in general more intricate and complex than top-down parsing.

There exist various LR parsing algorithms. They impose different boundary conditions on the syntax to be processed. The more lenient these conditions are, the more complex the parsing process. We mention here the SLR (DeRemer, 1971) and LALR (LaLonde et al., 1971) methods without explaining them in any further detail.

## 4.4 Exercises

### 4.4.1
Algol 60 contains a multiple assignment of the form `v1 := v2 := ... vn := e`. It is defined by the following syntax:
```
assignment = leftpartlist expression.
leftpartlist = leftpart | leftpartlist leftpart.
leftpart = variable ":=" .
expression = variable | expression "+" variable.
variable = ident | ident "[" expression "]" .
```
Which is the degree of lookahead necessary to parse this syntax according to the top-down principle? Propose an alternative syntax for multiple assignments requiring a lookahead of one symbol only.

### 4.4.2
Determine the symbol sets first and follow of the EBNF constructs production, expression, term, and factor. Using these sets, verify that EBNF is deterministic.
```
syntax = {production}.
production = id "=" expression "." .
expression = term {"|" term}.
term = factor {factor}.
factor = id | string | "(" expression ")" | "[" expression "]" | "{" expression "}".
id = letter {letter | digit}.
string = """ {character} """.
```

### 4.4.3
Write a parser for EBNF and extend it with statements generating the data structure (for table-driven parsing) corresponding to the read syntax.
