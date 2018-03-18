# 3. Regular Languages

Syntactic equations of the form defined in EBNF generate context-free languages. The term "context-free" is due to Chomsky and stems from the fact that substitution of the symbol left of `=` by a sequence derived from the expression to the right of `=` is always permitted, regardless of the context in which the symbol is embedded within the sentence. It has turned out that this restriction to context freedom (in the sense of Chomsky) is quite acceptable for programming languages, and that it is even desirable. Context dependence in another sense, however, is indispensible. We will return to this topic in Chapter 8.

Here we wish to investigate a subclass rather than a generalization of context-free languages. This subclass, known as regular languages, plays a significant role in the realm of programming languages. In essence, they are the context-free languages whose syntax contains no recursion except for the specification of repetition. Since in EBNF repetition is specified directly and without the use of recursion, the following, simple definition can be given:

>A language is regular, if its syntax can be expressed by a single EBNF expression.

The requirement that a single equation suffices also implies that only terminal symbols occur in the expression. Such an expression is called a regular expression.

Two brief examples of regular languages may suffice. The first defines identifiers as they are common in most languages; and the second defines integers in decimal notation. We use the nonterminal symbols letter and digit for the sake of brevity. They can be eliminated by substitution, whereby a regular expression results for both identifier and integer.
```
identifier = letter {letter | digit}.
integer = digit {digit}.
letter = "A" | "B" | ... | "Z".
digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9".
```
The reason for our interest in regular languages lies in the fact that programs for the recognition of regular sentences are particularly simple and efficient. By "recognition" we mean the determination of the structure of the sentence, and thereby naturally the determination of whether the sentence is well formed, that is, it belongs to the language. Sentence recognition is called syntax analysis.

For the recognition of regular sentences a finite automaton, also called a state machine, is necessary and sufficient. In each step the state machine reads the next symbol and changes state. The resulting state is solely determined by the previous state and the symbol read. If the resulting state is unique, the state machine is deterministic, otherwise nondeterministic. If the state machine is formulated as a program, the state is represented by the current point of program execution.

The analysing program can be derived directly from the defining syntax in EBNF. For each EBNF construct K there exists a translation rule which yields a program fragment Pr(K). The translation rules from EBNF to program text are shown below. Therein sym denotes a global variable always representing the symbol last read from the source text by a call to procedure next. Procedure error terminates program execution, signalling that the symbol sequence read so far does not belong to the language.
```
| K                           | Pr(K)
|-----------------------------+----------------------------------------
| "x"                         | IF sym = "x" THEN next ELSE error END
| (exp)                       | Pr(exp)
| [exp]                       | IF sym IN first(exp) THEN Pr(exp) END
| {exp}                       | WHILE sym IN first(exp) DO Pr(exp) END
| fac0 fac1 ... facn          | Pr(fac0); Pr(fac1); ... Pr(facn)
| term0 | term1 | ... | termn | CASE sym OF
|                             |   first(term0): Pr(term0)
|                             |   first(term1): Pr(term1)
|                             |    ...
|                             |   first(termn): Pr(termn)
|                             | END
```
The set first(K) contains all symbols with which a sentence derived from construct K may start. It is the set of start symbols of K. For the two examples of identifiers and integers they are:
```
first(integer) = digits = {"0", "1", "2", "3", "4", "5", "6", "7", "8", "9"}
first(identifier) = letters = {"A", "B", ... , "Z"}
```
The application of these simple translations rules generating a parser from a given syntax is, however, subject to the syntax being deterministic. This precondition may be formulated more concretely as follows:
```
| K               | Cond(K)
|-----------------+-------------------------------------------------
| term0 | term1   | The terms must not have any common
|                 | start symbols.
| fac0 fac1       | If fac0 contains the empty sequence,
|                 | then the factors must not have any common
|                 | start symbols.
| [exp] or {exp}  | The sets of start symbols of exp and of
|                 | symbols that may follow K must be disjoint.
```
These conditions are satisfied trivially in the examples of identifiers and integers, and therefore we obtain the following programs for their recognition:
```
IF sym IN letters THEN next ELSE error END ;
WHILE sym IN letters + digits DO
  CASE sym OF
   "A" .. "Z": next
   | "0" .. "9": next
  END
END
IF sym IN digits THEN next ELSE error END ;
WHILE sym IN digits DO next END
```
Frequently, the program obtained by applying the translation rules can be simplified by eliminating conditions which are evidently established by preceding conditions. The conditions sym IN letters and sym IN digits are typically formulated as follows:
```
("A" <= sym) & (sym <= "Z")    ("0" <= sym) & (sym <= "9")
```
The significance of regular languages in connection with programming languages stems from the fact that the latter are typically defined in two stages. First, their syntax is defined in terms of a vocabulary of abstract terminal symbols. Second, these abstract symbols are defined in terms of sequences of concrete terminal symbols, such as ASCII characters. This second definition typically has a regular syntax. The separation into two stages offers the advantage that the definition of the abstract symbols, and thereby of the language, is independent of any concrete representation in terms of any particular character sets used by any particular equipment.

This separation also has consequences on the structure of a compiler. The process of syntax analysis is based on a procedure to obtain the next symbol. This procedure in turn is based on the definition of symbols in terms of sequences of one or more characters. This latter procedure is called a scanner, and syntax analysis on this second, lower level, lexical analysis. The definition of symbols in terms of characters is typically given in terms of a regular language, and therefore the scanner is typically a state machine.

We summarize the differences between the two levels as follows:

| Process          | Input Element | Algorithm | Syntax       |
|:----------------:|:-------------:|:---------:|:------------:|
| Lexical Analysis | Char          | Scanner   | Regular      |
| Syntaz Analysis  | Symbol        | Parser    | Context free |

As an example we show a scanner for a parser of EBNF. Its terminal symbols and their definition in terms of characters are
```
symbol = {blank} (identifier | string | "(" | ")" | "[" | "]" | "{" | "}" | "|" | "=" | ".") .
identifier = letter {letter | digit}.
string = """ {character} """.
```
From this we derive the procedure `GetSym` which, upon each call, assigns a numeric value representing the next symbol read to the global variable sym. If the symbol is an identifier or a string, the actual character sequence is assigned to the further global variable id. It must be noted that typically a scanner also takes into account rules about blanks and ends of lines.
Mostly these rules say: blanks and ends of lines separate consecutive symbols, but otherwise are of no significance. Procedure `GetSym`, formulated in Oberon, makes use of the following declarations.
```
CONST IdLen = 32;
ident = 0; literal = 2; lparen = 3; lbrak = 4; lbrace = 5; bar = 6; eql = 7;
rparen = 8; rbrak = 9; rbrace = 10; period = 11; other = 12;
TYPE Identifier = ARRAY IdLen OF CHAR;
VAR ch: CHAR;
sym: INTEGER;
id: Identifier;
R: Texts.Reader;
```
Note that the abstract reading operation is now represented by the concrete call `Texts.Read(R, ch)`. `R` is a globally declared Reader specifying the source text. Also note that variable ch must be global, because at the end of GetSym it may contain the first character belonging to the next symbol. This must be taken into account upon the subsequent call of `GetSym`.
```
PROCEDURE GetSym;
	VAR i: INTEGER;
	BEGIN
		WHILE ~R.eot & (ch <= " ") DO Texts.Read(R, ch) END ; (*skip blanks*)
	CASE ch OF
		"A" .. "Z", "a" .. "z": sym := ident; i := 0;
			REPEAT id[i] := ch; INC(i); Texts.Read(R, ch)
			UNTIL (CAP(ch) < "A") OR (CAP(ch) > "Z");
			id[i] := 0X
		| 22X: (*quote*)
			Texts.Read(R, ch); sym := literal; i := 0;
			WHILE (ch # 22X) & (ch > " ") DO
				id[i] := ch; INC(i); Texts.Read(R, ch)
			END ;
			IF ch <= " " THEN error(1) END ;
			id[i] := 0X; Texts.Read(R, ch)
		| "=" : sym := eql; Texts.Read(R, ch)
		| "(" : sym := lparen; Texts.Read(R, ch)
		| ")" : sym := rparen; Texts.Read(R, ch)
		| "[" : sym := lbrak; Texts.Read(R, ch)
		| "]" : sym := rbrak; Texts.Read(R, ch)
		| "{" : sym := lbrace; Texts.Read(R, ch)
		| "}" : sym := rbrace; Texts.Read(R, ch)
		| "|" : sym := bar; Texts.Read(R, ch)
		| "." : sym := period; Texts.Read(R, ch)
	ELSE sym := other; Texts.Read(R, ch)
	END
END GetSym
```

## 3.1 Exercise
### 3.1.1
Sentences of regular languages can be recognized by finite state machines. They are usually described by transition diagrams. Each node represents a state, and each edge a state transition. The edge is labelled by the symbol that is read by the transition. Consider the following diagrams and describe the syntax of the corresponding languages in EBNF.

![](https://github.com/overdev/compiler-construction/blob/master/images/cc_exercise_3_1.png)

