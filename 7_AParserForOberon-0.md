# 7. A Parser for Oberon-0

# 7.1. The Scanner

Before starting to develop a parser, we first turn our attention to the design of its scanner. The scanner has to recognize terminal symbols in the source text. First, we list its vocabulary:
```
* DIV MOD & + - OR
= # < <= > >= . , : ) ]
OF THEN DO UNTIL ( [ ~ := ;
END ELSE ELSIF IF WHILE REPEAT
ARRAY RECORD CONST TYPE VAR PROCEDURE BEGIN MODULE
```
The words written in upper-case letters represent single, terminal symbols, and they are called reserved words. They must be recognized by the scanner, and therefore cannot be used as identifiers. In addition to the symbols listed, identifiers and numbers are also treated as terminal symbols. Therefore the scanner is also responsible for recognizing identifiers and numbers.

It is appropriate to formulate the scanner as a module. In fact, scanners are a classic example of the use of the module concept. It allows certain details to be hidden from the client, the parser, and to make accessible (to export) only those features which are relevant to the client. The exported facilities are summarized in terms of the module's interface definition:
```
DEFINITION OSS; (*Oberon Subset Scanner*)
IMPORT Texts;
CONST IdLen = 16;
	(*symbols*) null = 0; times = 1; div = 3; mod = 4;
	and = 5; plus = 6; minus = 7; or = 8; eql = 9;
	neq = 10; lss = 11; leq = 12; gtr = 13; geq = 14;
	period = 18; int = 21; false = 23; true = 24;
	not = 27; lparen = 28; lbrak = 29;
	ident = 31; if = 32; while = 34;
	repeat = 35;
	comma = 40; colon = 41; becomes = 42; rparen = 44;
	rbrak = 45; then = 47; of = 48; do = 49;
	semicolon = 52; end = 53;
	else = 55; elsif = 56; until = 57;
	array = 60; record = 61; const = 63; type = 64;
	var = 65; procedure = 66; begin = 67; module = 69;
	eof = 70;
TYPE Ident = ARRAY IdLen OF CHAR;
VAR val: INTEGER;
	id: Ident;
	error: BOOLEAN;

PROCEDURE Mark(msg: ARRAY OF CHAR);
PROCEDURE Get(VAR sym: INTEGER);
PROCEDURE Init(T: Texts.Text; pos: LONGINT);
END OSS.
```
The symbols are mapped onto integers. The mapping is defined by a set of constant definitions. Procedure `Mark` serves to output diagnostics about errors discovered in the source text. Typically, a short explanation is written into a log text together with the position of the discovered error. Procedure `Get` represents the actual scanner. It delivers for each call the next symbol recognized.

The procedure performs the following tasks:

1. Blanks and line ends are skipped.
2. Reserved words, such as `BEGIN` and `END`, are recognized.
3. Sequences of letters and digits starting with a letter, which are not reserved words, are recognized as identifiers. The parameter sym is given the value ident, and the character sequence itself is assigned to the global variable id.
4. Sequences of digits are recognized as numbers. The parameter sym is given the value number, and the number itself is assigned to the global variable val.
5. Combinations of special characters, such as `:=` and `<=`, are recognized as a symbol.
6. Comments, represented by sequences of arbitrary characters beginning with `(*` and ending with `*)` are skipped.
7. The symbol null is returned, if the scanner reads an illegal character (such as `$` or `%`). The symbol `eof` is returned if the end of the text is reached. Neither of these symbols occur in a wellformed program text.

## 7.2. The parser
The construction of the parser strictly follows the rules explained in Chapters 3 and 4. However, before the construction is undertaken, it is necessary to check whether the syntax satisfies the restricting rules guaranteeing determinism with a lookahead of one symbol. For this purpose, we first construct the sets of start and follow symbols. They are listed in the following tables.

| S                    | First(S)                           |     |
|---------------------:|:-----------------------------------|:---:|
| selector             | . [                                | *   |
| factor               | ( ~ integer ident                  |     |
| term                 | ( ~ integer ident                  |     |
| SimpleExpression     | + - ( ~ integer ident              |     |
| expression           | + - ( ~ integer ident              |     |
| assignment           | ident                              | *   |
| ProcedureCall        | ident                              |     |
| statement            | ident IF WHILE REPEAT              | *   |
| StatementSequence    | ident IF WHILE REPEAT              | *   |
| FieldList            | ident                              |     |
| type ident           | ARRAY RECORD                       |     |
| FPSection            | ident VAR                          |     |
| FormalParameters     | (                                  |     |
| ProcedureHeading     | PROCEDURE                          |     |
| ProcedureBody        | END CONST TYPE VAR PROCEDURE BEGIN |     |
| ProcedureDeclaration | PROCEDURE                          |     |
| declarations         | CONST TYPE VAR PROCEDURE           | *   |
| module               | MODULE                             |     |

| S                    | Follow(S) |
|---------------------:|:-----------|
| selector             | * DIV MOD & + - OR = # < <= > >= , ) ] := OF THEN DO ; END ELSE ELSIF UNTIL
| factor               |  * DIV MOD & + - OR = # < <= > >= , ) ] OF THEN DO ; END ELSE ELSIF UNTIL
| term                 | + - OR = # < <= > >= , ) ] OF THEN DO ; END ELSE ELSIF UNTIL
| SimpleExpression     | = # < <= > >= , ) ] OF THEN DO ; END ELSE ELSIF UNTIL
| expression           | , ) ] OF THEN DO ; END ELSE ELSIF UNTIL
| assignment           | ; END ELSE ELSIF UNTIL
| ProcedureCall        | ; END ELSE ELSIF UNTIL
| statement            | ; END ELSE ELSIF UNTIL
| StatementSequence    | END ELSE ELSIF UNTIL
| FieldList            | ; END
| type                 | ) ;
| FPSection            | ) ;
| FormalParameters     | ;
| ProcedureHeading     | ;
| ProcedureBody        | ;
| ProcedureDeclaration | ;
| declarations         | END BEGIN

The subsequent checks of the rules for determinism show that this syntax of Oberon-0 may indeed be handled by the method of recursive descent using a lookahead of one symbol. A procedure is constructed corresponding to each nonterminal symbol. Before the procedures are formulated, it is useful to investigate how they depend on each other. For this purpose we design a dependence graph (Figure 7.1). Every procedure is represented as a node, and an edge is drawn to all nodes on which the procedure depends, that is, calls directly or indirectly. Note that some nonterminal symbols do not occur in this graph, because they are included in other symbols in a trivial way. For example, `ArrayType` and `RecordType` are contained in type only and are therefore not explicitly drawn. Furthermore we recall that the symbols ident and integer occur as terminal symbols, because they are treated as such by the scanner.

![Figure 7.1. Dependence diagram of parsing procedures](https://github.com/overdev/compiler-construction/blob/master/images/cc_figure_7_1.png)

Every loop in the diagram corresponds to a recursion. It is evident that the parser must be formulated in a language that allows recursive procedures. Furthermore, the diagram reveals how procedures may possibly be nested. The only procedure which is not called by another procedure is Module. The structure of the program mirrors this diagram. The parser, like the scanner, is also formulated as a module.

## 7.3. Coping with syntactic errors

So far we have considered only the rather simple task of determining whether or not a source text is well formed according to the underlying syntax. As a side-effect, the parser also recognizes the structure of the text read. As soon as an inacceptable symbol turns up, the task of the parser is completed, and the process of syntax analysis is terminated. For practical applications, however, this proposition is unacceptable. A genuine compiler must indicate an error diagnostic message and thereafter proceed with the analysis. It is then quite likely that further errors will be detected.

Continuation of parsing after an error detection is, however, possible only under the assumption of certain hypotheses about the nature of the error. Depending on this assumption, a part of the subsequent text must be skipped, or certain symbols must be inserted. Such measures are necessary even when there is no intention of correcting or executing the erroneous source program. Without an at least partially correct hypothesis, continuation of the parsing process is futile (Graham and Rhodes, 1975; Rechenberg and Mössenböck, 1985).

The technique of choosing good hypotheses is complicated. It ultimately rests upon heuristics, as the problem has so far eluded formal treatment. The principal reason for this is that the formal syntax ignores factors which are essential for the human recognition of a sentence. For instance, a missing punctuation symbol is a frequent mistake, not only in program texts, but an operator symbol is seldom omitted in an arithmetic expression. To a parser, however, both kinds of symbols are syntactic symbols without distinction, whereas to the programmer the semicolon appears as almost redundant, and a plus symbol as the essence of the expression. This kind of difference must be taken into account if errors are to be treated sensibly. To summarize, we postulate the following quality criteria for error handling:

1. As many errors as possible must be detected in a single scan through the text.
2. As few additional assumptions as possible about the language are to be made.
3. Error handling features should not slow down the parser appreciably.
4. The parser program should not grow in size significantly.

We can conclude that error handling strongly depends on a concrete case, and that it can be described by general rules only with limited success. Nevertheless, there are a few heuristic rules which seem to have relevance beyond our specific language, Oberon. Notably, they concern the design of a language just as much as the technique of error treatment. Without doubt, a simple language structure significantly simplifies error diagnostics, or, in other words, a complicated syntax complicates error handling unnecessarily.

Let us differentiate between two cases of incorrect text. The first case is where symbols are missing. This is relatively easy to handle. The parser, recognizing the situation, proceeds by omitting one or several calls to the scanner. An example is the statement at the end of factor, where a closing parenthesis is expected. If it is missing, parsing is resumed after emitting an error message:
```
IF sym = rparen THEN Get(sym) ELSE Mark(" ) missing") END
```
Virtually without exception, only weak symbols are omitted, symbols which are primarily of a syntactic nature, such as the comma, semicolon and closing symbols. A case of wrong usage is an equality sign instead of an assignment operator, which is also easily handled.

The second case is where wrong symbols are present. Here it is unavoidable to skip them and to resume parsing at a later point in the text. In order to facilitate resumption, Oberon features certain constructs beginning with distinguished symbols which, by their nature, are rarely misused. For example, a declaration sequence always begins with the symbol `CONST`, `TYPE`, `VAR`, or `PROCEDURE`, and a structured statement always begins with `IF`, `WHILE`, `REPEAT`, `CASE`, and so on. Such strong symbols are therefore never skipped. They serve as synchronization points in the text, where parsing can be resumed with a high probability of success. In Oberon's syntax, we establish four synchronization points, namely in factor, statement, declarations and type. At the beginning of the corresponding parser procedures symbols are being skipped. The process is resumed when either a correct start symbol or a strong symbol is read.
```
PROCEDURE factor;
BEGIN (*sync*)
	IF (sym < int) OR (sym > ident) THEN Mark("ident ?");
	REPEAT Get(sym) UNTIL (sym >= int) & (sym < ident)
END ;
...
END factor;

PROCEDURE StatSequence;
BEGIN (*sync*)
	IF ~((sym = OSS.ident) OR (sym >= OSS.if) & (sym <= OSS.repeat)
	OR (sym >= OSS.semicolon)) THEN Mark("Statement?");
	REPEAT Get(sym) UNTIL (sym = ident) OR (sym >= if)
END ;
...
END StatSequence;

PROCEDURE Type;
BEGIN (*sync*)
	IF (sym # ident) & (sym < array) THEN Mark("type ?");
	REPEAT Get(sym) UNTIL (sym = ident) OR (sym >= array)
END ;
...
END Type;

PROCEDURE declarations;
BEGIN (*sync*)
	IF (sym < const) & (sym # end) THEN Mark("declaration?");
	REPEAT Get(sym) UNTIL (sym >= const) OR (sym = end)
END ;
...
END declarations;
```
Evidently, a certain ordering among symbols is assumed at this point. This ordering had been chosen such that the symbols are grouped to allow simple and efficient range tests. Strong symbols not to be skipped are assigned a high ranking (ordinal number) as shown in the definition of the scanner's interface.

In general, the rule holds that the parser program is derived from the syntax according to the recursive descent method and the explained translation rules. If a read symbol does not meet expectations, an error is indicated by a call of procedure Mark, and analysis is resumed at the next synchronization point. Frequently, follow-up errors are diagnosed, whose indication may be omitted, because they are merely consequences of a formerly indicated error. The statement which results for every synchronization point can be formulated generally as follows:
```
IF ~(sym IN follow(SYNC)) THEN Mark(msg);
REPEAT Get(sym) UNTIL sym IN follow(SYNC)
END
```
where follow(SYNC) denotes the set of symbols which may correctly occur at this point.

In certain cases it is advantageous to depart from the statement derived by this method. An example is the construct of statement sequence. Instead of 
```
Statement;
WHILE sym = semicolon DO Get(sym); Statement END
```
we use the formulation
```
REPEAT (*sync*)
IF sym < ident THEN Mark("ident?"); ... END ;
Statement;
IF sym = semicolon THEN Get(sym)
ELSIF sym IN follow(StatSequence) THEN Mark("semicolon?")
END
UNTIL ~(sym IN follow(StatSequence))
```
This replaces the two calls of Statement by a single call, whereby this call may be replaced by the procedure body itself, making it unnecessary to declare an explicit procedure. The two tests after Statement correspond to the legal cases where, after reading the semicolon, either the next statement is analysed or the sequence terminates. Instead of the condition sym IN follow(StatSequence) we use a Boolean expression which again makes use of the specifically chosen ordering of symbols:
```
(sym >= semicolon) & (sym < if) OR (sym >= array)
```
The construct above is an example of the general case where a sequence of identical subconstructs which may be empty (here, statements) are separated by a weak symbol (here, semicolon). A second, similar case is manifest in the parameter list of procedure calls. The statement
```
IF sym = lparen THEN
Get(sym); expression;
WHILE sym = comma DO Get(sym); expression END ;
IF sym = rparen THEN Get(sym) ELSE Mark(") ?") END
END
```
is being replaced by
```
IF sym = lparen THEN Get(sym);
REPEAT expression;
	IF sym = comma THEN Get(sym)
	ELSIF (sym = rparen) OR (sym >= semicolon) THEN Mark(") or , ?")
	END
UNTIL (sym = rparen) OR (sym >= semicolon)
END
```
A further case of this kind is the declaration sequence. Instead of
```
IF sym = const THEN ... END ;
IF sym = type THEN ... END ;
IF sym = var THEN ... END ;
```
we employ the more liberal formulation
```
REPEAT
	IF sym = const THEN ... END ;
	IF sym = type THEN ... END ;
	IF sym = var THEN ... END ;
	IF (sym >= const) & (sym <= var) THEN Mark("bad declaration sequence") END
UNTIL (sym # const) & (sym # type) & (sym # var)
```
The reason for deviating from the previously given method is that declarations in a wrong order (for example variables before constants) must provoke an error message, but at the same time can be parsed individually without difficulty. A further, similar case can be found in `Type`. In all these cases, it is absolutely mandatory to ensure that the parser can never get caught in the loop. The easiest way to achieve this is to make sure that in each repetition at least one symbol is being read, that is, that each path contains at least one call of `Get`. Thereby, in the worst case, the parser reaches the end of the source text and stops.

It should now have become clear that there is no such thing as a perfect strategy of error handling which would translate all correct sentences with great efficiency and also sensibly diagnose all errors in ill-formed texts. Every strategy will handle certain abstruse sentences in a way that appears unexpected to its author. The essential characteristics of a good compiler, regardless of details, are that (1) no sequence of symbols leads to its crash, and (2) frequently encountered errors are correctly diagnosed and subsequently generate no, or few additional, spurious error messages. 

The strategy presented here operates satisfactorily, albeit with possibilities for improvement. The strategy is remarkable in the sense that the error handling parser is derived according to a few, simple rules from the straight parser. The rules are augmented by the judicious choice of a few parameters which are determined by ample experience in the use of the language.

## 7.4. Exercises

### 7.4.1
The scanner uses a linear search of `array KeyTab` to determine whether or not a sequence of letters is a key word. As this search occurs very frequently, an improved search method would certainly result in increased efficiency. Replace the linear search in the array by
1. A binary search in an ordered array.
2. A search in a binary tree.
3. A search of a hash table. Choose the hash function so that at most two comparisons are necessary to find out whether or not the letter sequence is a key word.

Determine the overall gain in compilation speed for the three solutions.

### 7.4.2

Where is the Oberon syntax not LL(1), that is, where is a lookahead of more than one symbol necessary? Change the syntax in such a way that it satisfies the LL(1) property.

### 7.4.3

Extend the scanner in such a way that it accepts real numbers as specified by the Oberon syntax.
