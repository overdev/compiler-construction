# 8. Consideration of Context Specified by Declarations

## 8.1. Declarations

Although programming languages are based on context-free languages in the sense of Chomsky, they are by no means context free in the ordinary sense of the term. The context sensitivity is manifest in the fact that every identifier in a program must be declared. Thereby it is associated with an object of the computing process which carries certain permanent properties. For example, an identifier is associated with a variable, and this variable has a specific data type as specified in the identifier's declaration. An identifier occurring in a statement refers to the object specified in its declaration, and this declaration lies outside the statement. We say that the declaration lies in the context of the statement.

Consideration of context evidently lies beyond the capability of context-free parsing. In spite of this, it is easily handled. The context is represented by a data structure which contains an entry for every declared identifier. This entry associates the identifier with the denoted object and its properties. The data structure is known by the name symbol table. This term dates back to the times of assemblers, when identifiers were called symbols. Also, the structure is typically more complex than a simple array.

The parser will now be extended in such a way that, when parsing a declaration, the symbol table is suitably augmented. An entry is inserted for every declared identifier. To summarize:

- Every declaration results in a new symbol table entry.
- Every occurrence of an identifier in a statement requires a search of the symbol table in order to determine the attributes (properties) of the object denoted by the identifier.

A typical attribute is the object's class. It indicates whether the identifier denotes a constant, a variable, a type or a procedure. A further attribute in all languages with data types is the object's type.

The simplest form of data structure for representing a set of items is the list. Its major disadvantage is a relatively slow search process, because it has to be traversed from its root to the desired element. For the sake of simplicity - data structures are not the topic of this text - we declare the following data types representing linear lists:
```
Object = POINTER TO ObjDesc;
ObjDesc = RECORD
	name: Ident;
	class: INTEGER;
	type: Type;
	next: Object;
	val: LONGINT
END
```
The following declarations are, for example, represented by the list shown in Figure 8.1.

![Figure 8.1. Symbol table representing objects with names and attributes.](https://github.com/overdev/compiler-construction/blob/master/images/cc_figure_8_1.png)

For the generation of new entries we introduce the procedure `NewObj` with the explicit parameter class, the implied parameter id and the result `obj`. The procedure checks whether the new identifier (`id`) is already present in the list. This would signify a multiple definition and constitute a programming error. The new entry is appended at the end of the list, so that the list mirrors the order of the declarations in the source text.
```
PROCEDURE NewObj(VAR obj: Object; class: INTEGER);
	VAR new, x: Object;
BEGIN x := topScope;
	WHILE (x.next # NIL) & (x.next.name # id) DO x := x.next END ;
	IF x.next = NIL THEN
		NEW(new); new.name := id;
		new.class := class;
		new.next := NIL;
		x.next := new;
		obj := new
	ELSE obj := x.next; Mark("multiple declaration")
	END
END NewObj;
```
In order to speed up the search process, the list is often replaced by a tree structure. Its advantage becomes noticeable only with a fairly large number of entries. For structured languages with local scopes, that is, ranges of visibility of identifiers, the symbol table must be structured accordingly, and the number of entries in each scope becomes relatively small. Experience shows that as a result the tree structure yields no substantial benefit over the list, although it requires a more complicated search process and the presence of three successor pointers per entry instead of one.
Note that the linear ordering of entries must also be recorded, because it is significant in the case of procedure parameters.

A procedure find serves to access the object with name id. It represents a simple linear search, proceeding through the list of scopes, and in each scope through the list of objects.
```
PROCEDURE find(VAR obj: OSG.Object);
	VAR s, x: Object;
BEGIN s := topScope;
	REPEAT x := s.next;
		WHILE (x # NIL) & (x.name # id) DO x := x.next END ;
		s := s.dsc
	UNTIL (x # NIL) OR (s = NIL);
	IF x = NIL THEN x := dummy; OSS.Mark("undef") END ;
	obj := x
END find;
```

## 8.2. Entries for data types

In languages featuring data types, their consistency checking is one of the most important tasks of a compiler. The checks are based on the type attribute recorded in every symbol table entry. Since data types themselves can be declared, a pointer to the respective type entry appears to be the obvious solution. However, types may also be specified anonymously, as exemplified by the following declaration:
```
VAR a: ARRAY 10 OF INTEGER
```
The type of variable `a` has no name. An easy solution to the problem is to introduce a proper data type in the compiler to represent types as such. Named types then are represented in the symbol table by an entry of type `Object`, which in turn refers to an element of type `Type`.
```
Type = POINTER TO TypDesc;
TypDesc = RECORD
	form, len: INTEGER;
	fields: Object;
	base: Type
END
```
The attribute form differentiates between elementary types (`INTEGER`, `BOOLEAN`) and structured types (arrays, records). Further attributes are added according to the individual forms.

Characteristic for arrays are their length (number of elements) and the element type (base). For records, a list representing the fields must be provided. Its elements are of the class Field. As an example, Figure 8.2. shows the symbol table resulting from the following declarations:

![Figure 8.2. Symbol table representing declared objects.](https://github.com/overdev/compiler-construction/blob/master/images/cc_figure_8_2.png)

As far as programming methodology is concerned, it would be preferable to introduce an extended data type for each class of objects, using a base type with the fields id, type and next only. We refrain from doing so, not least because all such types would be declared within the same module, and because the use of a numeric discrimination value (class) instead of individual types avoids the need for numerous, redundant type guards and thereby increases efficiency. After all, we do not wish to promote an undue proliferation of data types.

## 8.3. Data representation at run-time

So far, all aspects of the target computer and its architecture, that is, of the computer for which code is to be generated, have been ignored, because our sole task was to recognize source text and to check its compliance with the syntax. However, as soon as the parser is extended into a compiler, knowledge about the target computer becomes mandatory.

First, we must determine the format in which data are to be represented at run-time in the store. The choice inherently depends on the target architecture, although this fact is less apparent because of the similarity of virtually all computers in this respect. Here, we refer to the generally accepted form of the store as a sequence of individually addressable byte cells, that is, of byteoriented memories. Consecutively declared variables are then allocated with monotonically increasing or decreasing addresses. This is called sequential allocation.

Every computer features certain elementary data types together with corresponding instructions, such as integer addition and floating-point addition. These types are invariably scalar types, and they occupy a small number of consecutive memory locations (bytes). In the present language Oberon-0, there exist only the two basic, scalar data types: `INTEGER` and `BOOLEAN`. In the computer used here, the former occupies 4 bytes, the latter a single byte. However, in general every type has a size, and every variable has an address.

These attributes, type.size and obj.adr, are determined when the compiler processes declarations. The sizes of the elementary types are given by the machine architecture, and corresponding entries are generated when the compiler is loaded and initialized. For structured, declared types, their size has to be computed.

The size of an array is its element size multiplied by the number of its elements. The address of an element is the sum of the array's address and the element's index multiplied by the element size.
Let the following general declarations be given:
```
TYPE T = ARRAY n OF T0
VAR a: T
```
Then type size and element address are obtained by the following equations:
```
size(T) = n * size(T0)
adr(a[x]) = adr(a) + x * size(T0)
```
For multi-dimensional arrays, the corresponding formulas (see Figure 8.3) are:
![](https://github.com/overdev/compiler-construction/blob/master/images/cc_mathexpr_8.png)

Note that for the computation of the size the array's lengths in all dimensions are known, because they occur as constants in the program text. However, the index values needed for the computation of an element's address are typically not known before program execution.

![Figure 8.3. Representation of a matrix.](https://github.com/overdev/compiler-construction/blob/master/images/cc_figure_8_3.png)

In contrast, for record structures, both type size and field address are known at compile time. Let us consider the following declarations:
```
TYPE T = RECORD f0: T0; f1: T1; ... ; fk-1: Tk-1 END
VAR r: T
```
Then the type's size and the field addresses are computed according to the following formulas:

![](https://github.com/overdev/compiler-construction/blob/master/images/cc_mathexpr_8_2.png)

Absolute addresses of variables are usually unknown at the time of compilation. All generated addresses must be considered as relative to a common base address which is given at run-time. The effective address is then the sum of this base address and the address determined by the compiler. If a computer's store is byte-addressed, as is fairly common, a further point must be considered. Although bytes can be accessed individually, typically a small number of bytes (say 4 or 8) are transferred from or to memory as a packet, a so-called word. If allocation occurs strictly in sequential order it is possible that a variable may occupy (parts of) several words (see Figure 8.4), assuming a size of 2 for integers, 4 for real numbers. But this should definitely be avoided, because otherwise a variable access would involve several memory accesses, resulting in an appreciable slowdown. A simple method of overcoming this problem is to round up (or down) each variable's address to the next multiple of its size. This process is called alignment. The rule holds for elementary data types. For arrays, the size of their element type is relevant, and for records we simply round up to the computer's word size. The price of alignment is the loss of some bytes in memory, which is quite negligible.

![Figure 8.4. Alignment in address computation.](https://github.com/overdev/compiler-construction/blob/master/images/cc_figure_8_4.png)

The following additions to the parsing procedure for declarations are necessary to generate the required symbol table entries:
```
IF sym = type THEN (* "TYPE" ident "=" type *)
	Get(sym);
	WHILE sym = ident DO
		NewObj(obj, Typ); Get(sym);
		IF sym = eql THEN Get(sym) ELSE Mark("= ?") END ;
		Type1(obj.type);
		IF sym = semicolon THEN Get(sym) ELSE Mark("; ?") END
	END
END ;
IF sym = var THEN (* "VAR" ident {"," ident} ":" type *)
	Get(sym);
	WHILE sym = ident DO
		IdentList(Var, first); Type1(tp); obj := first;
		WHILE obj # NIL DO
			obj.type := tp; INC(adr, obj.type.size);
			obj.val := adr; obj := obj.next
		END ;
		IF sym = semicolon THEN Get(sym) ELSE Mark("; ?") END
	END
END ;
```
Here, procedure IdentList is used to process an identifier list, and the recursive procedure `Type1` serves to compile a type declaration.
```
PROCEDURE IdentList(class: INTEGER; VAR first: Object);
	VAR obj: Object;
BEGIN
	IF sym = ident THEN
		NewObj(first, class); Get(sym);
		WHILE sym = comma DO
			Get(sym);
			IF sym = ident THEN NewObj(obj, class); Get(sym) ELSE Mark("ident?") END
		END;
		IF sym = colon THEN Get(sym) ELSE Mark("no :") END
	END
END IdentList;

PROCEDURE Type1(VAR type: Type);
	VAR n: INTEGER;
		obj, first: Object; tp: Type;
BEGIN type := intType; (*sync*)
	IF (sym # ident) & (sym < array) THEN Mark("ident?");
		REPEAT Get(sym) UNTIL (sym = ident) OR (sym >= array)
	END ;
	IF sym = ident THEN
		find(obj); Get(sym);
		IF obj.class = Typ THEN type := obj.type ELSE Mark("type?") END
	ELSIF sym = array THEN
		Get(sym);
		IF sym = number THEN n := val; Get(sym) ELSE Mark("number?"); n := 1 END ;
		IF sym = of THEN Get(sym) ELSE Mark("OF?") END ;
		Type1(tp); NEW(type); type.form := Array; type.base := tp;
		type.len := n; type.size := type.len * tp.size
	ELSIF sym = record THEN
		Get(sym); NEW(type); type.form := Record; type.size := 0; OpenScope;
		REPEAT
			IF sym = ident THEN
				IdentList(Fld, first); Type1(tp);
				obj := first;
				WHILE obj # NIL DO
					obj.type := tp; obj.val := type.size;
					INC(type.size, obj.type.size);
					obj := obj.next
				END
			END ;
			IF sym = semicolon THEN Get(sym)
			ELSIF sym = ident THEN Mark("no ;")
			END
		UNTIL sym # ident;
		type.fields := topScope.next; CloseScope;
		IF sym = end THEN Get(sym) ELSE Mark("END?") END
	ELSE Mark("ident ?")
	END
END Type1;
```
The auxiliary procedures OpenScope and CloseScope ensure that the list of record fields is not intermixed with the list of variables. Every record declaration establishes a new scope of visibility of field identifiers, as required by the definition of the language Oberon. Note that the list into which new entries are inserted is rooted in the global variable topScope.

## 8.4. Exercises

### 8.4.1
The scope of identifiers is defined to extend from the place of declaration to the end of the procedure in which the declaration occurs. What would be necessary to let this range extend from the beginning to the end of the procedure?

### 8.4.2
Consider pointer declarations as defined in Oberon. They specify a type to which the declared pointer is bound, and this type may occur later in the text. What is necessary to accommodate this relaxation of the rule that all referenced entities must be declared prior to their use?
