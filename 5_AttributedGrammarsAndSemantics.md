# 5. Attributed Grammars and Semantics

In attributed grammars certain attributes are associated with individual constructs, that is, with nonterminal symbols. The symbols are parameterized and represent whole classes of variants. This serves to simplify the syntax, but is, in practice, indispensible for extending a parser into a genuine translator (Rechenberg and Mössenböck, 1985). The translation process is characterized by the association of a (possibly empty) output with every recognition of a sentential construct. Each syntactic equation (production) is accompanied by additional rules defining the relationship between the attribute values of the symbols which are reduced, the attribute values for the resulting nonterminal symbol, and the issued output. We present three applications for attributes and attribute rules.

## 5.1. Type rules 

As a simple example we shall consider a language featuring several data types. Instead of specifying separate syntax rules for expressions of each type (as was done in Algol 60), we define expressions exactly once, and associate the data type T as attribute with every construct involved. For example, an expression of type T is denoted as exp(T), that is, as exp with attribute value T. Rules about type compatibility are then regarded as additions to the individual syntactic equations. For instance, the requirements that both operands of addition and subtraction must be of the same type, and that the result type is the same as that of the operands, are specified by such additional attribute rules:

| Syntax                     | Attribute rule | Context condition |
|:--------------------------:|:--------------:|:-----------------:|
| `exp(T0) = term(T1) | `    | T0 := T1       |                   |
| `exp(T1) "+" term(T2) | `  | T0 := T1       | T1 = T2           |
| `exp(T1) "-" term(T2). `   | T0 := T1       | T1 = T2           |

If operands of the types INTEGER and REAL are to be admissible in mixed expressions, the rules become more relaxed, but also more complicated:
```
T0 := if (T1 = INTEGER) & (T2 = INTEGER) then INTEGER else REAL,
T1 = INTEGER or T1 = REAL
T2 = INTEGER or T2 = REAL
```
Rules about type compatibility are indeed also static in the sense that they can be verified without execution of the program. Hence, their separation from purely syntactic rules appears quite arbitrary, and their integration into the syntax in the form of attribute rules is entirely appropriate. However, we note that attributed grammars obtain a new dimension, if the possible attribute values (here, types) and their number are not known a priori.

If a syntactic equation contains a repetition, then it is appropriate with regard to attribute rules to express it with the aid of recursion. In the case of an option, it is best to express the two cases separately. This is shown by the following example where the two expressions
```
exp(T0) = term(T1) {"+" term(T2)}.       exp(T0) = ["-"] term(T1).
```
are split into pairs of terms, namely
```
exp(T0) = term(T1)                 |     exp(T0) = term(T1)
        | exp(T1) "+" term(T2).    |             | "-" term(T1).
```
The type rules associated with a production come into effect whenever a construct corresponding to the production is recognized. This association is simple to implement in the case of a recursive descent parser: program statements implementing the attribute rules are simply interspersed within the parsing statements, and the attributes occur as parameters to the parser procedures standing for the syntactic constructs (nonterminal symbols). The procedure for recognizing expressions may serve as a first example to demonstrate this extension process, where the original parsing procedure serves as the scaffolding:
```
PROCEDURE expression;
BEGIN term;
	WHILE (sym = "+") OR (sym = "-") DO
		GetSym; term
	END
END expression
```
is extended to implement its attribute (type) rules:
```
PROCEDURE expression(VAR typ0: Type);
	VAR typ1, typ2: Type;
BEGIN term(typ1);
	WHILE (sym = "+") OR (sym = "-") DO
		GetSym; term(typ2);
		typ1 := ResType(typ1, typ2)
	END ;
	typ0 := typ1
END expression
```

## 5.2. Evaluation rules

As our second example we consider a language consisting of expressions whose factors are numbers only. It is a short step to extend the parser into a program not only recognizing, but at the same time also evaluating expressions. We associate with each construct its value through an attribute called val. In analogy to the type compatibility rules in our previous example, we now must process evaluation rules while parsing. Thereby we have implicitly introduced the notion of semantics:
```
Syntax                                   Attribute rule (semantics)
exp(v0)    = term(v1)                    v0 := v1
           | exp(v1) "+" term(v2)        v0 := v1 + v2
           | exp(v1) "-" term(v2).       v0 := v1 - v2

term(v0)   = factor(v1)                  v0 := v1
           | term(v1) "*" factor(v2)     v0 := v1 * v2
           | term(v1) "/" factor(v2).    v0 := v1 / v2

factor(v0) = number(v1)                  v0 := v1
           | "(" exp(v1) ")".            v0 := v1
```
Here, the attribute is the computed, numeric value of the recognized construct. The necessary extension of the corresponding parsing procedure leads to the following procedure for expressions:
```
PROCEDURE expression(VAR val0: INTEGER);
	VAR val1, val2: INTEGER; op: CHAR;
BEGIN term(val1);
	WHILE (sym = "+") OR (sym = "-") DO
		op : = sym; GetSym; term(val2);
		IF op = "+" THEN val1 : = val1 + val2 ELSE val1 := val1 - val2 END
	END ;
	val0 := val1
END expression
```

## 5.3 Translation rules

A third example of the application of attributed grammars exhibits the basic structure of a compiler. The additional rules associated with a production here do not govern attributes of symbols, but specify the output (code) issued when the production is applied in the parsing process. The generation of output may be considered as a side-effect of parsing. Typically, the output is a sequence of instructions. In this example, the instructions are replaced by abstract symbols, and their output is specified by the operator put.
```
Syntax                      Output rule (semantics)
exp    = term               -
         exp "+" term       put("+")
         exp "-" term.      put("-")
term   = factor             -
         term "*" factor    put("*")
         term "/" factor.   put("/")
factor = number             put(number)
         "(" exp ")".       -
```
As can easily be verified, the sequence of output symbols corresponds to the parsed expression in postfix notation. The parser has been extended into a translator.

|    Infix notation | Postfix notation |
|:-----------------:|:----------------:|
|             2 + 3 |            2 3 + |
|         2 * 3 + 4 |        2 3 * 4 + |
|         2 + 3 * 4 |        2 3 4 * + |
| (5 - 4) * (3 + 2) |    5 4 - 3 2 + * |

The procedure parsing and translating expressions is as follows:
```
PROCEDURE expression;
	VAR op: CHAR;
BEGIN term;
	WHILE (sym = "+") OR (sym = "-") DO
		op := sym; GetSym; term; put(op)
	END
END expression
```
When using a table-driven parser, the tables expressing the syntax may easily be extended also to represent the attribute rules. If the evaluation and translation rules are also contained in associated tables, one is tempted to speak about a formal definition of the language. The general, table-driven parser grows into a general, table-driven compiler. This, however, has so far remained a utopia, but the idea goes back to the 1960s. It is represented schematically by Figure 5.1.

![Figure 5.1. Schema of a general, parametrized compiler.](https://github.com/overdev/compiler-construction/blob/master/images/cc_figure_5_1.png)

Ultimately, the basic idea behind every language is that it should serve as a means for communication. This means that partners must use and understand the same language. Promoting the ease by which a language can be modified and extended may therefore be rather counterproductive. Nevertheless, it has become customary to build compilers using table-driven parsers, and to derive these tables from the syntax automatically with the help of tools. The semantics are expressed by procedures whose calls are also integrated automatically into the parser. Compilers thereby not only become bulkier and less efficient than is warranted, but also much less transparent. The latter property remains one of our principal concerns, and therefore we shall not pursue this course any further.

## 5.4. Exercise

### 5.4.1

Extend the program for syntactic analysis of EBNF texts in such a way that it generates (1) a list of terminal symbols, (2) a list of nonterminal symbols, and (3) for each nonterminal symbol the sets of its start and follow symbols. Based on these sets, the program is then to determine whether the given syntax can be parsed top-down with a lookahead of a single symbol. If this is not so, the program displays the conflicting productions in a suitable way.
**Hint**: Use Warshall's algorithm (R. W. Floyd, Algorithm 96, Comm. ACM, June 1962).
```
TYPE matrix = ARRAY [1..n],[1..n] OF BOOLEAN;

PROCEDURE ancestor(VAR m: matrix; n: INTEGER);
(* Initially m[i,j] is TRUE, if individual i is a parent of individual j.
At completion, m[i,j] is TRUE, if i is an ancestor of j *)
	VAR i, j, k: INTEGER;
BEGIN
	FOR i := 1 TO n DO
		FOR j := 1 TO n DO
			IF m[j, i] THEN
				FOR k := 1 TO n DO
					IF m[i, k] THEN m[j, k] := TRUE END
				END
			END
		END
	END
END ancestor
```
It may be assumed that the numbers of terminal and nonterminal symbols of the analysed languages do not exceed a given limit (for example, 32).
