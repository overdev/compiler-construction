# 6. The Programming Language Oberon-0

In order to avoid getting lost in generalities and abstract theories, we shall build a specific, concrete compiler, and we explain the various problems that arise during the project. In order to do this, we must postulate a specific source language.

Of course we must keep this compiler, and therefore also the language, sufficiently simple in order to remain within the scope of an introductory tutorial. On the other hand, we wish to explain as many of the fundamental constructs of languages and compilation techniques as possible. Out of these considerations have grown the boundary conditions for the choice of the language: it must be simple, yet representative. We have chosen a subset of the language Oberon (Reiser and Wirth, 1992), which is a condensation of its ancestors Modula-2 (Wirth, 1982) and Pascal (Wirth, 1971) into their essential features. Oberon may be said to be the latest offspring in the tradition of Algol 60 (Naur, 1960). Our subset is called Oberon-0, and it is sufficiently powerful to teach and exercise the foundations of modern programming methods.

Concerning program structures, Oberon-0 is reasonably well developed. The elementary statement is the assignment. Composite statements incorporate the concepts of the statement sequence and conditional and repetitive execution, the latter in the form of the conventional if-. while-, and repeat statements. Oberon-0 also contains the important concept of the subprogram, represented by the procedure declaration and the procedure call. Its power mainly rests on the possibility of parameterizing procedures. In Oberon, we distinguish between value and variable parameters.

With respect to data types, however, Oberon-0 is rather frugal. The only elementary data types are integers and the logical values, denoted by `INTEGER` and `BOOLEAN`. It is thus possible to declare integer-valued constants and variables, and to construct expressions with arithmetic operators. Comparisons of expressions yield Boolean values, which can be subjected to logical operations.

The available data structures are the array and the record. They can be nested arbitrarily. Pointers, however, are omitted.

Procedures represent functional units of statements. It is therefore appropriate to associate the concept of locality of names with the notion of the procedure. Oberon-0 offers the possibility of declaring identifiers local to a procedure, that is, in such a way that the identifiers are valid (visible) only within the procedure itself.

This very brief overview of Oberon-0 is primarily to provide the reader with the context necessary to understand the subsequent syntax, defined in terms of EBNF.
```
ident
	= letter {letter | digit}.
integer
	= digit {digit}.
selector
	= {"." ident | "[" expression "]"}.
number
	= integer.
factor
	= ident selector
	| number
	| "(" expression ")"
	| "~" factor.
term
	= factor {("*" | "DIV" | "MOD" | "&") factor}.
SimpleExpression
	= ["+"|"-"] term {("+"|"-" | "OR") term}.
expression
	= SimpleExpression [("=" | "#" | "<" | "<=" | ">" | ">=") SimpleExpression].
assignment
	= ident selector ":=" expression.
ActualParameters
	= "(" [expression {"," expression}] ")" .
ProcedureCall
	= ident selector [ActualParameters].
IfStatement
	= "IF" expression "THEN" StatementSequence {"ELSIF" expression "THEN" StatementSequence} ["ELSE" StatementSequence] "END".
WhileStatement
	= "WHILE" expression "DO" StatementSequence "END".
RepeatStatement
	= “REPEAT” Statement Sequence “UNTIL” expression.
statement
	= [assignment | ProcedureCall | IfStatement | WhileStatement].
StatementSequence
	= statement {";" statement}.
IdentList
	= ident {"," ident}.
ArrayType
	= "ARRAY" expression "OF" type.
FieldList
	= [IdentList ":" type].
RecordType
	= "RECORD" FieldList {";" FieldList} "END".
type
	= ident | ArrayType | RecordType.
FPSection
	= ["VAR"] IdentList ":" type.
FormalParameters
	= "(" [FPSection {";" FPSection}] ")".
ProcedureHeading
	= "PROCEDURE" ident [FormalParameters].
ProcedureBody
	= declarations ["BEGIN" StatementSequence] "END" ident.
ProcedureDeclaration
	= ProcedureHeading ";" ProcedureBody.
declarations
	= ["CONST" {ident "=" expression ";"}] ["TYPE" {ident "=" type ";"}] ["VAR" {IdentList ":" type ";"}] {ProcedureDeclaration ";"}.
module
	= "MODULE" ident ";" declarations ["BEGIN" StatementSequence] "END" ident "." .
```
The following example of a module may help the reader to appreciate the character of the language. The module contains various, well-known sample procedures. It also contains calls to specific, predefined procedures `OpenInput`, `ReadInt`, `WriteInt`, `WriteLn`, and `eot()` whose purpose is evident. Note that every command which asks for input, must start with a call to `OpenInput`.
```
MODULE Samples;

PROCEDURE Multiply*;
	VAR x, y, z: INTEGER;
BEGIN OpenInput; ReadInt(x); ReadInt(y); z := 0;
	WHILE x > 0 DO
		IF x MOD 2 = 1 THEN z := z + y END ;
		y := 2*y; x := x DIV 2
	END ;
	WriteInt(x, 4); WriteInt(y, 4); WriteInt(z, 6); WriteLn
END Multiply;

PROCEDURE Divide*;
	VAR x, y, r, q, w: INTEGER;
BEGIN OpenInput; ReadInt(x); ReadInt(y); r := x; q := 0; w := y;
	WHILE w <= r DO w := 2*w END ;
	WHILE w > y DO
		q := 2*q; w := w DIV 2;
		IF w <= r THEN r := r - w; q := q + 1 END
	END ;
	WriteInt(x.4); WriteInt(y, 4); WriteInt(q, 4); WriteInt(r, 4); WriteLn
END Divide;

PROCEDURE Sum*;
	VAR n, s: INTEGER;
BEGIN OpenInput; s:= 0;
	WHILE ~eot() DO ReadInt(n); WriteInt(n, 4); s := s + n END ;
	WriteInt(s, 6); WriteLn
END Sum;

END Samples.
```
Corresponding commands are:
```
Samples.Multiply 7 9
Samples.Divide 65 7
Samples.Sum 1 2 3 4 5~
```
## 6.1 Exercise

### 6.1.1

Determine the code for the computer defined in Chapter 9, generated from the program listed at the end of this Chapter.
