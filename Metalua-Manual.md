# Meta-programming in Metalua
## Concepts
### Lua
Lua is a very clean and powerful language, with everything the discriminating hacker will love: advanced data structures, true function closures, coroutines (a.k.a collaborative multithreading), powerful runtime introspection and metaprogramming abilities, and ultra-easy integration with C.

The general approach in Lua's design is to implement a small number of very powerful concepts, and use them to easily offer particular services. For instance, objects can be implemented through metatables (which allows you to customize the behavior of data structures), or through function closures. It's quite easy to develop a class based system with single or multiple inheritance, or a prototype based system à la Self, or the kind of more advanced and baroque things that only CLOS users could dream of...

Basically, Lua could be thought of as Scheme, with:
- a conventional syntax (similar to Pascal's or Ruby's);
- the associative table as basic datatype instead of the list;
- no full continuations (although coroutines are actually one-shot semi-continuations);
- no macro system.
### Metalua
Metalua is an extension of Lua, which essentially addresses the lack of a macro system by providing compile-time metaprogramming (CTMP) and the ability for a user to extend the syntax from within Lua.

Runtime metaprogramming (RTMP) allows a program to inspect itself while running: an object can thus enumerate its fields and methods, their properties, maybe dump its source code; it can be modified on-the-fly by adding a method, changing its class, etc. But this doesn't allow changing the shape of the language itself: you cannot use this to add exceptions to a language that lacks them, nor call-by-need (a.k.a. "lazy") evaluation to a traditional language, nor continuations, nor new control structures, new operators... To do this, you need to modify the compiler itself. It can be done, if you have the sources of the compiler, but that's generally not worth it, given the complexity of a compiler program and the portability and maintenance issues that ensue.
### Metaprogramming
A compiler is essentially a system which takes sources (generally as a set of ASCII files), turns them into a practical-to-play-with data structure, does stuff on it, then feeds it to a bytecode or machine code producer. The source and byte-code stages are bad abstraction levels to do anything practical: the sensible way to represent code, when you want to manipulate it with programs, is the abstract syntax tree (AST). This is the practical-to-play-with abstraction level mentioned above: a tree in which each node corresponds to a control structure, where the inclusion relationship is respected (e.g. if  instruction I is in a loop's body B, then the node representing I is a subtree of the tree representing B)...

CTMP is possible if the compiler allows its user to read, generate and modify AST, and to splice these generated AST back into programs. This is done by Lisp and Scheme by making the programmer write programs directly in AST (hence the large amount of parentheses in Lisp sources), and by offering a magic instruction that executes during compilation a piece of code which generates an AST, and inserts this AST into the source AST: that magic couple of instructions is the macro system.

Metalua has a similar execute-and-splice-the-result magic construct; the main difference is that it doesn't force the programmer to directly write in AST (although he's allowed to if he finds it most suitable for a specific task). However, supporting "real language syntax" adds a couple of issues to CTMP: there is a need for transformation from real syntax to AST and the other way around, as well as a need for a way to extend the syntax.

This manual won't try to teach you Lua as there's a wealth of excellent tutorials on the web for this. I highly recommend Roberto Ierusalimschy's "Programming in Lua" book, a.k.a. "the blue PiL", probably one of the best programming books since K&R's "The C Language". Suffice to say that a seasoned programmer will be able to program in Lua in a couple of hours, although some advanced features (coroutines, function environments, function closures, metatables, runtime introspection) might take longer to master if you don't already know a language supporting them.

Among resources available online, my personal favorites would be:
- The reference manual: http://www.lua.org/manual/5.1
- The first edition of PiL, kindly put online by its author at http://www.lua.org/pil
- A compact reference sheet (grammar and standard libraries) by Enrico Colombini: http://lua-users.org/wiki/LuaShortReference
- Generally speaking, the Lua community wiki (http://lua-users.org/wiki) is invaluable.
- The mailing list (http://www.lua.org/lua-l.html) and the IRC channel (irc://irc.freenode.net/#lua) are populated with a very helpful community.
- You will also find a lot of useful programs and libraries for Lua hosted at http://luaforge.net: various protocol parsers, bindings to 2D/3D native/portable GUI, sound, database drivers...
- A compilation of the community's wisdom will eventually be plubished as "Lua Gems"; you can already check its ToC at http://www.lua.org/gems

So, instead of including yet another Lua tutorial, this manual will rather focus on the features specific to Metalua, that is mainly:
- The couple of syntax extensions offered by Metalua over Lua;
- The two CTMP magic constructs `+{...}` and `-{...}`;
- The libraries which support CTMP (mainly for syntax extension).
### Metalua design philosophy
Metalua has been designed to occupy a vacant spot in the space of CTMP-enabled languages:
- Lisp offers a lot of flexibility, at the price of macro-friendly syntax, rather than user-friendly. Besides the overrated problem of getting used to those lots of parentheses, it's all too tempting to mix macros and normal code in Lisp, in a way that doesn't visually stand out; this really doesn't encourage the writing of reusable, mutually compatible libraries. As a result of this extreme flexibility, large scale collaboration doesn't seem to happen, and Lisps lack of a de facto comprehensive set of standard libs, besides those included in Common Lisp's specification. Comparisons have been drawn between getting Lispers to work together and herding cats...

- Macro-systems bolted on existing languages (Template Haskell, CamlP5, MetaML...) tend to be hard to use: the syntax and semantics of these target languages are complex, and make macro writing much harder than necessary. Moreover, for some reason, most of these projects target statically typed languages: although static inference type systems à la Hindley-Milner are extremely powerful tools in many contexts, my intuition is that static types are more of a burden than a help for many macro-friendly problems.

- Languages built from scratch, such as Converge or Logix, have to bear with the very long (often decade) maturing time required by a programming language. Moreover, they lack the existing libraries and developers that come with an already successful language.

Lua presents many features that beg for a real macro system:
- Its compact, clear, orthogonal, powerful semantics, and its approach of giving powerful generic tools rather than ready-made closed features to its users.
- Its excellent support for runtime metaprogramming.
- Its syntax, despite (or due to its) being very readable and easy to learn, is also extremely simple to parse. This means no extra technology gets in the way of handling syntax (no BNF-like specialized language, no byzantine rules and exceptions). Even more importantly, provided that developers respect a couple of common-sense rules, cohabitation of multiple syntax extensions in a single project is made surprisingly easy.

Upon this powerful and sane base, Metalua adds CTMP with the following design goals:
- Simple things should be easy and look clean: writing simple macros shouldn't require an advanced knowledge of the language's internals. And since we spend 95% of our time not writing macros, the syntax should be optimized for regular code rather than for code generation.
- Good coding practices should be encouraged. Among others, separation between meta-levels must be obvious, so that it stands out when something interesting is going on. Ideally, good code must look clean, and messy code should look ugly.
- However, the language must be an enabler, not handcuffs: it should ensure that users know what they're doing, but it must provide them with all the power they're willing to handle.

Finally, it's difficult to talk about a macro-enabled language without making Lisp comparisons. Metalua borrows a lot to Scheme's love for empowering minimalism, through Lua. However, in many other respects, it's closer to Common Lisp: where Scheme insists on doing The Right Thing, CL and Metalua assume that the programmer knows better than the compiler. Therefore, when a powerful but potentially dangerous feature is considered, metalua generally tries to warn the user that he's entering the twilight zone, but will let him proceed. The most prominent example is probably macro hygiene. Scheme pretty much constraints macro writing into a term rewriting system: it allows the compiler to enforce macro hygiene automatically, but is sometimes crippling when writing complex macros (although it is, of course, Turing-complete). Metalua opts to offer CL style, non-hygienic macros, so that AST is regular data manipulated by regular code. Hygienic safety is provided by an optional library, which makes it easy but not mandatory to do hygienic macros.
## Metalua syntax extensions over Lua
Metalua is essentially Lua + code generation at compile time + extensible syntax. However, there are a couple of additional constructs, considered of general interest, which have been added to Lua's original syntax. These are presented in this section.
### Anonymous Functions
Lua lets you use anonymous functions. However, when programming in a functional style, where there are a lot of short anonymous functions simply returning an expression, the default syntax becomes cumbersome. Metalua, being functional-style friendly, offers a terser idiom: `function(arg1, arg2, argn) return some_expr end` can be written: `|arg1,arg2,argn| some_exp`.

Notice that this notation is currying-friendly, i.e. one can easily write functions that return functions: `function(x) return function(y) return x+y end end` is simply written `|x||y| x+y`.

Lua functions can return several values, but it appeared that supporting multiple return values in metalua's short lambda notation caused more harm than good. If you need multiple returns, use the traditional long syntax.

Finally, it's perfectly legal to define a parameterless function, as in `|| 42`. This makes a convenient way to pass values around in a lazy way.
### Functions as infix operators
In many cases, people would like to extend syntax simply to create infix binary operators. Haskell offers a nice compromise to satisfy this need without causing any mess, and metalua incorporated it: when a function is put between backquotes, it becomes infix. for instance, let's consider the plus function `plus = |x,y| x+y`; this function can be called the classic way, as in `plus(20, 22)`; but if you want to use it in an infix context, you can also write ``20 `plus` 22``.
### Algebraic Datatypes
This syntax for datatypes is of special importance to metalua, as it's used to represent source code being manipulated. Therefore, it has its dedicated section later in this manual.
### Metalevel Shifters
These two dual notations are the core of metaprogramming: one transforms code into a manipulatable representation, and the other transforms the representation back into code. They are noted `+{...}` and `-{...}`, and due to their central role in metalua, their use can't be summed up adequately here: they are fully described in the subsequent sections about metaprogramming.
## Data Structures
### Algebraic Datatypes
(ADT is also the usual accronym for Abstract DataType. However, I'll never talk about abstract datatypes in this manual, so there's no reason to get confused about it. ADT always refers to algebraic datatypes).

Metalua's distinctive feature is its ability to easily work on program source codes as trees, and this include a proper syntax for tree manipulation. The generic table structure offered by Lua is definitely good enough to represent trees, but since we're going to manipulate them a lot, we give them a specific syntax which makes them easier to read and write.

So, a tree is basically a node, with:
- a tag (a string, stored in the table field named `tag`)
- some children, which are either sub-trees, or atomic values (generally strings, numbers or booleans). These children are stored in the array-part of the table, i.e. with consecutive integers as keys.
#### Example 1
The most canonical example of ADT is probably the inductive list. Such a list is described either as the empty list `Nil`, or a pair (called a `cons` in Lisp) of the first element on one side (`car` in Lisp), and the list of remaining elements on the other side (`cdr` in Lisp). These will be represented in Lua as `{ tag = "Nil" }` and `{ tag = "Cons", car, cdr }`. The list `(1, 2, 3)` will be represented as:
```
{ tag="Cons", 1, 
  { tag="Cons", 2, 
    { tag="Cons", 3, 
      { tag="Nil" } } } }
```
#### Example 2
Here is a more programming language oriented example: imagine that we are working on a symbolic calculator. We will have to work with this:
- literal numbers, represented as integers;
- symbolic variables, represented by the string of their symbol;
- formulae, i.e. numbers, variables and/or sub-formulae combined by operators. Such a formula is represented by the symbol of its operator, and the sub-formulae/numbers/variables it operates on.

Most operations, e.g. evaluation or simplification, will do different things depending on whether it is applied on a number, a variable, or a formula. Moreover, the meaning of the fields in data structures depends on that data type. The datatype is given by the name put in the tag field. In this example, tag can be one of Number, Var or Formula. The formula $e^{ip}+1$ would be encoded as:
```
{ tag="Formula", "Addition", 
  { tag="Formula", "Exponent", 
    { tag="Variable", "e" },
    { tag="Formula", "Multiplication", 
      { tag="Variable", "i" },
      { tag="Variable", "pi" } } },
  { tag="Number", 1 } }
```
#### Syntax
The simple data above already has a quite ugly representation, so here are the syntax extensions we provide to represent trees in a more readable way:
- The tag can be put in front of the table, prefixed with a backquote. For instance, `{ tag = "Cons", car, cdr }` can be abbreviated as `` `Cons{ car, cdr }``.
- If the table contains nothing but a tag, the braces can be omitted. Therefore, `{ tag = "Nil" }` can be abbreviated as `` `Nil`` (although `` `Nil{ }`` is also legal).
- If there is only one element in the table besides the tag, and this element is a literal number or a literal string, braces can be omitted. Therefore `{ tag = "Foo", "Bar" }` can be abbreviated as `` `Foo "bar"``.

With this syntax sugar, the $e^{ip}+1$ example above would read:
```
`Formula{ "Addition", 
   `Formula { "Exponent", 
      `Variable "e",
      `Formula { "Multiplication", 
         `Variable "i",
         `Variable "pi" } },
   `Number 1 }
```
Notice that this is a valid description of some tree structure in metalua, but it's not a representation of metalua code: metalua code is represented as tree structures indeed, but a structure different from this example's one. In other words, this is an ADT, but not an AST.

For the record, the metalua (AST) representation of the code $1+e(i*pi)$ is:
```
`Op { "add", `Number 1,
   `Op { "pow", `Id "e", 
      `Op { "mul", `Id "i", `Id "pi" } } }
```
After reading more about AST definition and manipulation tools, you'll hopefully be convinced that the latter representation is more powerful.
## Abstract Syntax Trees
An AST is an Abstract Syntax Tree, a data representation of source code suitable for easy manipulation. AST is just a particular usage of ADT, and we will represent them with the ADT syntax described above.
### Example
this is the tree representing the source code `print(foo, "bar"):`
```
`Call{ `Id "print", `Id "foo", `String "bar" }
```
Metalua tries, as much as possible, to shield users from direct AST manipulation, and a thorough knowledge of them is generally not needed. Metaprogrammers should know their general form, but it is reasonable to rely on a cheat-sheet to remember the exact details of AST structures. Such a summary is provided in the appendix of this tutorial, as a reference when dealing with them.

In the rest of this section, we will present the translation from Lua source to their corresponding AST.
## AST <-> Lua source translation
This subsection explains how to translate a piece of lua source code into the corresponding AST, and conversely. Most of time, users will rely on a mechanism called quasi-quotes to produce the AST they will work with, but it is sometimes necessary to directly deal with AST, and therefore to have at least a superficial knowledge of their structure.
### Expressions
Expressions are pieces of Lua code which can be evaluated to give a value. This includes constants, variable identifiers, table constructors, expressions based on unary or binary operators, function definitions, function calls, method invocations, and index selection from a table.

Expressions should not be confused with statements: an expression has a value with can be returned through evaluation, whereas statements just execute themselves and change the computer state (mainly memory and IO). For instance, `2+2` is an expression which evaluates to `4`, but `four=2+2` is a statement, which sets the value of variable four but has no value itself.
### Number constants
A number is represented by an AST with the tag `Number` and the number value as its sole child. For instance, 6 is represented by `` `Number 6``.
### String constants
A string is represented by an AST with the tag `String` and the string as its sole child. For instance, "foobar" is represented by:
`` `String "foobar"``.
### Other constant values
Here are the translations of other keyword-based atomic values:
- `nil` is encoded as `` `Nil``;
- `false` is encoded as `` `False``;
- `true` is encoded as `` `True``;
- `...` is encoded as `` `Dots``.
### Table constructors
A table constructor is encoded as:
```
`Table{ ( `Pair{ expr, expr } | expr )* }
```
This is a list, tagged with `Table`, whose elements are either:
- the AST of an expression, for array-part entries without an explicit associated key;
- a pair of expression AST, tagged with `Pair`: the first expression AST represents a key, and the second represents the value associated to this key.
#### Examples
The empty table `{ }` is represented as `` `Table{ }``;

- `{1, 2, "a"}` is represented as: `` `Table{ `Number 1, `Number 2, `String "a" };``

- `{x=1, y=2}` is syntax sugar for `{["x"]=1, ["y"]=2}`, and is represented by `` `Table{ `Pair{ `String "x", `Number 1 }, `Pair{ `String "y", `Number 2} };``

- indexed and non-indexed entries can be mixed: `{ 1, [100]="foo", 3}` is represented as `` `Table{ `Number 1, `Pair{ `Number 100, `String "foo"}, `Number 3 } ``;
### Binary Operators
Binary operations are represented by `` `Op{ operator, left, right}``, where operator is the operator's name as one of the strings below, left is the AST of the left operand, and right is the AST of the right operand.

The following table associates a Lua operator to its AST name:
| Op. | AST   | Op. | AST   | Op. | AST      | Op. | AST   |
|-----|-------|-----|-------|-----|----------|-----|-------|
| +   | "add" | -   | "sub" | *   | "mul"    | /   | "div" |
| %   | "mod" | ^   | "pow" | ..  | "concat" | ==  | "eq"  |
| <   | "lt"  | <=  | "le"  | and | "and"    | or  | "or"  |

Operator names are the sames as the corresponding Lua metatable entry, without the prefix "__". There is no operator for the operators `~=`, `>=` and `>`: they can be simulated by swapping the arguments of `<=` and `<`, or adding a `not` to the operator `==`.
#### Examples
- `2+2` is represented as `` `Op{ 'add', `Number 2, `Number 2 }``;
- `1+2*3` is represented as:
```
`Op { 'add', `Number 1, 
   `Op { 'mul', `Number 2, `Number 3 } }
```
- `(1+2)*3` is represented as:
```
`Op{ 'mul, `Op{ 'add', `Number 1, `Number 2 },
           `Number 3 } }
```
- `x>=1 and x<42` is represented as:
```
`Op{ 'and', `Op{ 'le', `Number  1, `Id "x" },
            `Op{ 'lt', `Id "x", `Number 42 } }
```
### Unary Operators
Unary operators are similar to binary operators, except that they only take the AST of one subexpression. The following table associates a Lua unary operator to its AST:
| Op. | AST   |
|-----|-------|
| -   | "sub" |
| #   | "len" |
| not | "not" |
#### Examples
- `-x` is represented as
	
	`` `Op{ `Sub, `Id "x" }``;
- `-(1+2)` is represented as: 

	`` `Op{ `Sub, `Op{ `Add, `Number 1, `Number 2 } } ``
- `#x` is represented as 
	
	`` `Op{ `Len, `Id "x" }``
### Indexed access
They are represented by an AST with tag `Index`, the table's AST as the first child, and the key's AST as the second child.
#### Examples
- `x[3]` is represented as 
	
	`` `Index{ `Id "x", `Number 3 }``;
- `x[3][5]` is represented as:

	`` `Index{ `Index{ `Id "x", `Number 3 }, `Number 5 } ``;
- `x.y` is syntax sugar for `x["y"]`, and is represented as:

	`` `Index{ `Id "x", `String "y" }``;

Notice that index AST can also appear as the left-hand side of assignments, as shall be shown in the subsection dedicated to statements.
### Function call
Function call AST have the tag `Call`, the called function's AST as the first child, and its arguments as the remaining children.
#### Examples
- `f()` is represented as `` `Call{ `Id "f" }``;
- `f(x, 1)` is represented as `` `Call{ `Id "f", `Id "x", `Number 1 }``;
- `f(x, ...)` is represented as `` `Call{ `Id "f", `Id "x", `Dots }``.

Notice that function calls can be used as expressions, but also as statements. 
### Method invocation
Method invocation AST have the tag `Invoke`, the object's AST as the first child, the string name of the method as a second child, and the arguments as the remaining children.

#### Examples
- `o:f()` is represented as `` `Invoke{ `Id "o", `String "f" }``;
- `o:f(x, 1)` is represented as: `` `Invoke{ `Id "o", `String "f", `Id "x", `Number 1 }``;
- `o:f(x, ...)` is represented as: `` `Invoke{ `Id "o", `String "f", `Id "x", `Dots }``;

Notice that method invocations can be used as expressions, but also as statements. Notice also that `function o:m(x) return x end` is not a method invocation, but syntax sugar for the statement `o["f"] = function(self, x) return x end`. See the paragraph about assignment in the statements subsection for its AST representation.
### Function definition
A function definition consists of a list of parameters and a block of statements. The parameter list, which can be empty, contains only variable names, represented by their `` `Id{...}`` AST, except for the last element of the list, which can also be dots AST `` `Dots`` (to indicate that the function is a variable-argument function).

The block is a list of statement AST, optionaly terminated with a `` `Return{...}`` or `` `Break`` pseudo-statement. These pseudo-statements will be described in the statements subsection.

The function definition is encoded as `` `Function{ parameters, block }``

#### Examples
- `function(x) return x end` is represented as:

	`` `Function{ { `Id x } { `Return{ `Id "x" } } }``;

- `function(x, y) foo(x) bar(y) end` is represented as:

```
`Function{ { `Id x, `Id y } 
           { `Call{ `Id "foo", `Id "x" },
             `Call{ `Id "bar", `Id "y" } } }
```
- `function(fmt, ...) print(string.format(fmt, ...)) end` is represented as:
```
`Function{ { `Id "fmt", `Dots } 
           { `Call{ `Id "print",
                    `Call{ `Index{ `Id "string",
                                   `String "format" }, 
                           `Id "fmt", 
                           `Dots } } } }
```

- `function f (x) return x end` is not an expression, but a statement: it is actually syntax sugar for the assignment `f = function (x) return x end`, and as such, is represented as:
```
`Let{ { `Id "f" }, 
      { `Function{ {`Id 'x'} {`Return{`Id 'x'} } } } }
```
(see assignment in the statements subsection for more details);
### Parentheses
In Lua, parentheses are sometimes semantically meaningful: when the parenthesised expression returns multiple values, putting it between parentheses forces it to return only one value. For instance, `local function f() return 1, 2, 3 end; return { f() }` will return `{1, 2, 3}`, whereas `local function f() return 1, 2, 3 end; return { (f()) }` will return `{ 1 }` (notice the parentheses around the function call).

Parentheses are represented in AST as the node `` `Paren{ }``. The second example above has the following AST:
```
{ `Localrec{ { `Id "f" },  
             { `Function{ { },
                          `Return{ `Number 1,
                                   `Number 2,
                                   `Number 3 } } } },
  `Return{ `Table{ `Paren{ `Call{ `Id "f" } } } } }
```
### Statements
Statements are instructions which modify the state of the computer. There are simple statements, such as variable assignment, local variable declaration, function calls and method invocation; there are also control structure statements, which take simpler statement and modify their action: these are if/then/else, repeat/until, while/do/end, for/do/end and do/end statements.
### Assignment
Variable assignment `a, b, c = foo, bar` is represetned by the AST `` `Set{ lhs, rhs }``, with lhs being a list of variables or table indexes, and rhs the list of values assigned to them.

#### Examples
- `x[1]=2` is represented as: `` `Set{ { `Index{ `Id "x", `Number 1 } }, { `Number 2 } }``;

- `a, b = 1, 2` is represented as:
`` `Set{ { `Id "a",`Id "b" }, { `Number 1, `Number 2 } }``;

- `a = 1, 2, 3` is represented as:
`` `Set{ { `Id "a" }, { `Number 1, `Number 2, `Number 3 } }``;

- `function f(x) return x end` is syntax sugar for:
`f = function (x) return x end`. As such, is represented as:
```
`Set{ { `Id "f" }, 
      { `Function{ {`Id 'x'}  {`Return{ `Id "x" } } } } }
```

- `function o:m(x) return x end` is syntax sugar for:
`o["f"] = function (self, x) return x end`, and as such, is represented as:
```
`Set{ { `Index{ `Id "o", `String "f" } }, 
      { `Function{ { `Id "self, "`Id x } 
                   { `Return{ `Id "x" } } } } }
```
### Local declaration
Local declaration `local a, b, c = foo`, bar works just as assignment, except that the tag is `Local`, and it is allowed to have an empty list as values.

#### Examples
- `local x=2` is represented as:
`` `Local{ { `Id "x" }, { `Number 2 } }``;

- `local a, b` is represented as:
`` `Local{ { `Id "a",`Id "b" }, { } }``;
### Recursive local declaration
In a local declaration, the scope of local variables starts after the statement. Therefore, it is not possible to refer to a variable inside the value it receives, and `local function f(x) f(x) end` is not equivalent to `local f = function (x) f(x) end`: in the latter, the f call inside the function definition probably refers to some global variable, whereas in the former, it refers to the local variable currently being defined (f this therefore a forever looping function).

To handle this, the AST syntax defines a special `` `Localrec`` local declaration statement, in which the variables enter in scope before their content is evaluated. Therefore, the AST corresponding to `local function f(x) f(x)` end is:
```
`Localrec{ { `Id "f" }, 
           { `Function{ { `Id x } 
                        { `Call{ `Id "f", `Id "x" } } } } }
```
Caveat: In the current implementation, both the variable names list and values list have to be of length 1. This is enough to represent `local function ... end`, but should be generalized in the final version of Metalua.
### Function calls and method invocations
They are represented the same way as their expression counterparts, see the subsection above for details.
### Blocks and pseudo-statements
Control statements generally take a block of instructions as parameters, e.g. as the body of a for loop. Such statement blocks are represented as the list of the instructions they contain. As a list, the block itself has no tag field.

#### Example
`foo(x); bar(y); return x,y` is represented as:
```
{ `Call{ `Id "foo", `Id "x" },
  `Call{ `Id "bar", `Id "y" },
  `Return{ `Id "x", `Id "y" } }
```
### Do statement
These represent `do ... end` statements, which limit local variables scopes. They are represented as blocks with a `Do` tag.

#### Example
`do foo(x); bar(y); return x,y end` is represented as:
```
`Do{ `Call{ `Id "foo", `Id "x" },
     `Call{ `Id "bar", `Id "y" },
     `Return{ `Id "x", `Id "y" } }
```
### While statement
`while <foo> do <bar1>; <bar2>; ... end` is represented as 
`` `While{ <foo>, { <bar1>, <bar2>, ... } }``.
### Repeat statement
`repeat <bar1>; <bar2>; ... until <foo>` is represented as 
`` `Repeat{ { <bar1>, <bar2>, ... }, <foo> }``.
### For statements
`for x=<first>,<last>,<step> do <foo>; <bar>; ... end` is represented as `` `Fornum{ `Id "x", <first>, <last>, <step>, { <foo>, <bar>, ... } }``.

The step parameter can be omitted if equal to 1.
```Lua
for x1, x2... in e1, e2... do
  <foo>;
  <bar>;
  ...
end
```
is represented as:
```
`Forin{ {`Id "x1",`Id "x2",...},
        { <e1>, <e2>,... },
        { <foo>, <bar>, ... } }.
```
### If statements
"If" statements are composed of a series of (condition, block) pairs, and optionally of a last default "else" block. The conditions and blocks are simply listed in an `` `If{ ... }`` ADT. Notice that an "if" statement without a final "else" block will have an even number of children, whereas a statement with a final ``else'' block will have an odd number of children.

#### Examples
`if <foo> then <bar>; <baz> end` is represented as:
`` `If{ <foo>, { <bar>, <baz> } }``;

`if <foo> then <bar1> else <bar2>; <baz2> end` is represented as: `` `If{ <foo>, { <bar1> }, { <bar2>, <baz2> } }``;

`if <foo1> then <bar1>; <baz1> elseif <foo2> then <bar2>; <baz2> end`
is represented as:
`` `If{ <foo1>, { <bar1>, <baz1> }, <foo2>,{ <bar2>, <baz2> } }``;

```
if     <foo1> then <bar1>; <baz1> 
elseif <foo2> then <bar2>; <baz2> 
else               <bar3>; <baz3> end
```
is represented as:
```
`If{ <foo1>, { <bar1>, <baz1> }, 
     <foo2>, { <bar2>, <baz2> },
             { <bar3>, <baz3> } }
```
### Breaks and returns
Breaks are represented by the childless `` `Break`` AST. Returns are represented by the (possibly empty) list of returned values.

#### Example
`return 1, 2, 3` is represented as:
`` `Return{ `Number 1, `Number 2, `Number 3 }``.
### Extensions with no syntax
A couple of AST nodes do not exist in Lua, nor in Metalua native syntax, but are provided because they are particularly useful for writing macros. They are presented here.
#### Goto and Labels
Labels can be string AST, identifier AST, or simply string; they indicate a target for goto statements. A very common idiom is ``local x = mlp.gensym(); ... `Label{ x }``. You just jump to that label with `` `Goto{ x }``.

Identifiers, string AST or plain strings are equivalent: `` `Label{ `Id "foo"}`` is synonymous for `` `Label{ `String "foo" }`` and `` `Label "foo"'``. The same equivalences apply for gotos, of course.

Labels are local to a function; you can safely jump out of a block, but if you jump inside a block, you're likely to get into unspecified trouble, as local variables will be in a random state.

#### Statements in expressions
A common need when writing a macro is to insert a statement in the middle of an expression. It can be done by using an anonymous function closure, but that would be expensive, so Metalua offers a better solution. The `Stat node evaluates a statement block in the middle of an expression, then returns an arbitrary expression as its result. Notice one important point: the expression is evaluated in the block's context, i.e. if there are some local variables declared in the block, the expression can use them.

For instance, `` `Stat{ +{local x=3}, +{x}}`` evaluates to 3.
## Splicing and quoting
As the previous section shows, AST are not extremely readable, and as promised, Metalua offer a way to avoid dealing with them directly. Well, rarely dealing with them anyway.

In this section, we will deal a lot with `+{...}` and `-{...}`; the only (but real) difficulty is not to get lost between meta-levels, i.e. not getting confused between a piece of code, the AST representing that piece of code, some code returning an AST that shall be executed during compilation, etc.
### Quasi-quoting
Quoting an expression is extremely easy: just put it between quasi-quotes. For instance, to get the AST representing `2+2`, just type `+{expr: 2+2}`. Actually, since most of quotes are actually expression quotes, you are even allowed to skip the `expr:` part: `+{2+2}` works just as well.

If you want to quote a statement, just substitute `expr:` with `stat:`: `+{stat: if x>3 then foo(bar) end}`.

Finally, you might wish to quote a block of code. As you can guess, just type: ``+{block: y = 7; x = y+1; if x>3 then foo(bar) end}``. A block is just a list of statements. That means that `+{block: x=1}` is the same as `{ +{stat: x=1} }` (a single-element list of statements).

However, quoting alone is not really useful: if it's just about pasting pieces of code verbatim, there is little point in meta-programming. We want to be able to poke "holes" in quasi-quotes (hence the "quasi"), and fill them with bits of AST coming from outside. Such holes are marked with a `-{...}` construct, called a splice, inside the quote. For instance, the following piece of Metalua will put the AST of `2+2` in variable `X`, then insert it in the AST an assignement in `Y`:
```Lua
X = +{ 2 + 2 }
Y = +{ four = -{ X } }
```
After this, `Y` will contain the AST representing `four = 2+2`. Because of this, a splice inside a quasi-quote is often called an anti-quote (as we shall see, splices also make sense, although a different one, outside quotes).

Of course, quotes and antiquotes can be mixed with explicit AST. The following lines all put the same value in Y, although often in a contrived way:

```Lua
-- As a single quote:
Y = +{stat: four = 2+2 }
-- Without any quote, directly as an AST:
Y = `Let{ { `Id "four" }, { `Op{ `Add, `Number 2, `Number 2 } } }
-- Various mixes of direct AST and quotes:
X = +{ 2+2 };                          Y = +{stat: four = -{ X } }
X = `Op{ `Add, +{2}, +{2} };           Y = +{stat: four = -{ X } }
X = `Op{ `Add, `Number 2, `Number 2 }; Y = +{stat: four = -{ X } }
Y = +{stat: four = -{ `Op{ `Add, `Number 2, `Number 2 } } }
Y = +{stat: four = -{ +{ 2+2 } } }
Y = `Let{ { `Id "four" }, { +{ 2+2 } } }
-- Nested quotes and splices cancel each other:
Y = +{stat: four = -{ +{ -{ +{ -{ +{ -{ +{ 2+2 } } } } } } } } }
```
The content of an anti-quote is expected to be an expression by default. However, it is legal to put a statement or a block of statements in it, provided that it returns an AST through a return statement. To do this, just add a `block:` (or `stat:`) markup at the beginning of the antiquote. The following line is (also) equivalent to the previous ones:
```
Y = +{stat: four = -{ block: 
                      local two=`Number 2
                      return `Op{ 'add', two, two } } }
```
Notice that in a block, where a statement is expected, a sub-block is also accepted, and is simply combined with the upper-level one. Unlike `` `Do{ }`` statements, it doesn't create its own scope. For instance, you can write `-{block: f(); g()}` instead of `-{stat:f()}; -{stat:g()}`.
### Splicing
Splicing is used in two, rather different contexts. First, as seen above, it's used to poke holes into quotations. But it is also used to execute code at compile time.

As can be expected from their syntaxes, `-{...}` undoes what `+{...} `does: quotes change a piece of code into the AST representing it, and splices cancel the quotation of a piece of code, including it directly in the AST (that piece of code therefore has to either be an AST, or evaluate to an AST. If not, the result of the surrounding quote won't be an AST).

But what happens when a splice is put outside of any quote? There is no explicit quotation to cancel, but actually, there is hidden AST generation. The process of compiling a Metalua source file consists in the following steps:
```
                  ______               ________
+-----------+    /      \    +---+    /        \    +--------+
|SOURCE FILE|-->< Parser >-->|AST|-->< Compiler >-->|BYTECODE|  
+-----------+    \______/    +---+    \________/    +--------+
```
So in reality, the source file is translated into an AST; when a splice is found, instead of just turning that AST into bytecode, we will execute the corresponding program, and put the AST it must return in the source code. This computed AST is the one which will be turned into bytecode in the resulting program. Of course, that means locally compiling the piece of code in the splice, in order to execute it:
```
                                                     +--------+
                  ______               ________   +->|BYTECODE|  
+-----------+    /      \    +---+    /        \  |  +--------+
|SOURCE FILE|-->< Parser >-->|AST|-->< Compiler >-+
+-----------+    \______/    +-^-+    \________/  |  +--------+
                              /|\      ________   +->|BYTECODE|  
                               |      /        \     +---+----+
                               +-----<   Eval   ><-------+
                                      \________/
```
As an example, consider the following source code, its compilation and its execution:

```
fabien@macfabien$ cat sample.mlua
-{block: print "META HELLO"
         return +{ print "GENERATED HELLO" } }
print "NORMAL HELLO"

fabien@macfabien$ metalua -v sample.mlua -o sample.luac
[ Param "sample.mlua" considered as a source file ]
[ Compiling `File "sample.mlua" ]
META HELLO
[ Saving to file "sample.luac" ]
[ Done ]
fabien@macfabien$ lua sample.luac
GENERATED HELLO
NORMAL HELLO
```
 
Thanks to the print statement in the splice, we see that the code it contains is actually executed during evaluation. More in detail, what happens is that:
- The code inside the splice is parsed and compiled separately;
- it is executed: the call to `print "META HELLO"` is performed, and the AST representing `print "GENERATED HELLO"` is generated and returned;
- in the AST generated from the source code, the splice is replaced by the AST representing `print "GENERATED HELLO"`. Therefore, what is passed to the compiler is the AST representing `print "GENERATED HELLO"; print "NORMAL HELLO"`.
Take time to read, re-read, play and re-play with the manipulation described above: understanding the transitions between meta-levels is the essence of meta-programming, and you must be comfortable with such transitions in order to make the best use of Metalua.

Notice that it is admissible, for a splice outside a quote, not to return anything. This allows to execute code at compile time without adding anything in the AST, typically to load syntax extensions. For instance, this source will just print "META HELLO" at compile time, and "NORMAL HELLO" at runtime: `-{print "META HELLO"}; print "NORMAL HELLO"`.
### A couple of simple, concrete examples
#### ternary choice operator
Let's build something more useful. As an example, we will build a ternary choice operator, equivalent to the `_ ? _ : _` from C. Here, we will not deal yet with syntax sugar: our operator will have to be put inside splices. Extending the syntax will be dealt with in the next section, and then, we will coat it with a sweet syntax.

Here is the problem: in Lua, choices are made by using `if _ then _ else _ end` statements. It is a statement, not an expression, which means that we can't use it in, for instance:
```
local hi = if lang=="fr" then "Bonjour" 
           else "hello" end -- illegal!
```
This won't compile. So, how do we turn the `if` statement into an expression? The simplest solution is to put it inside a function definition. Then, to actually execute it, we need to evaluate that function. Which means that our pseudo-code `local hi = (lang == "fr" ? "Bonjour" : "Hello")` will actually be compiled into:
```
local hi = 
  (function ()
     if lang == "fr" then return "Bonjour"
                     else return "Hello" end end) ()
```
We are going to define a function building the AST above, filling holes with parameters. Then we are going to use it in the actual code, through splices.

```
fabien@macfabien$ cat sample.lua
-{stat:
  -- Declaring the [ternary] metafunction. As a 
  -- metafunction, it only exists within -{...}, 
  -- i.e. not in the program itself.
  function ternary (cond, b1, b2)
     return +{ (function() 
                    if -{cond} then
                       return -{b1} 
                    else
                       return -{b2}
                    end
                 end)() }
  end }

lang = "en"
hi = -{ ternary (+{lang=="fr"}, +{"Bonjour"}, +{"Hello"}) }
print (hi)

lang = "fr"
hi = -{ ternary (+{lang=="fr"}, +{"Bonjour"}, +{"Hello"}) }
print (hi)

fabien@macfabien$ mlc sample.lua
Compiling sample.lua...
...Wrote sample.luac
fabien@macfabien$ lua sample.luac
Hello
Bonjour
```
#### Incrementation operator
Now, we will write another simple example, which doesn't use quasi-quotes, just to show that we can. Another operator that C developers might be missing with Lua is the `++` operator. As with the ternary operator, we won't show yet how to put the syntax sugar coating around it, just how to build the backend functionality.

Here, the transformation is really trivial: we want to encode `x++` as `x=x+1`. We will only deal with `++` as statement, not as an expression. However, `++` as an expression is not much more complicated to do. Hint: use the turn-statement-into-expr trick shown in the previous example. The AST corresponding to `x=x+1` is `` `Let{ { `Id x }, { `Op{ `Add, `Id x, `Number 1 } } }``. From here, the code is straightforward:

```
fabien@macfabien$ cat sample.lua
-{stat:
   function plusplus (var) 
      assert (var.tag == "Id")
      return `Let{ { var }, { `Op{ `Add, var, `Number 1 } } }
   end }

x = 1;                  
print ("x = " .. tostring (x))
-{ plusplus ( +{x} ) }; 
print ("Incremented x: x = " .. tostring (x))

fabien@macfabien$ mlc sample.lua
Compiling sample.lua...
...Wrote sample.luac
fabien@macfabien$ lua sample.luac
x = 1
Incremented x: x = 2
```
 
Now, we just put a decent syntax around this, and we are set! This is the subject of the next sections: gg is the generic grammar generator, which allows you to build and grow parsers. It's used to implement mlp, the Metalua parser, which turns Metalua sources into AST.

Therefore, the informations useful to extend Metalua syntax are:
- What are the relevant entry points in mlp, the methods which allow syntax extension.
- How to use these methods: this consists into knowing the classes defined into `gg`, which offer dynamic extension possibilities.
# Meta-programming libraries and extensions
## `gg`, the grammar generator
`gg` is the grammar generator, the library with which Metalua parser is built. Knowing it allows you to easily write your own parsers, and to plug them into mlp, the existing Metalua source parser. It defines a couple of generators, which take parsers as parameters, and return a more complex parser as a result by combining them.

Notice that `gg` sources are thoroughly commented, and try to be readable as a secondary source of documentation.

There are four main classes in gg, which allow you to generate:
- sequences of keywords and subparsers;
- keyword-driven sequence sets, i.e. parsers which select a sequence parser depending on an initial keyword;
- lists, i.e. series of an undetermined number of elements of the same type, optionally separated by a keyword (typically `,`);
- expressions, built with infix, prefix and suffix operators around a primary expression element;
### Sequences
A sequence parser combines sub-parsers together, calling them one after the other. A lot of these sub-parsers will simply read a keyword, and do nothing of it except making sure that it is indeed here: these can be specified by a simple string. For instance, the following declarations create parsers that read function declarations, thanks to some subparsers:
- `func_stat_name` reads a function name (a list of identifiers separated by dots, plus optionally a semicolon and a method name);
- `func_params_content` reads a possibly empty list of identifiers separated with commas, plus an optional `...`;
- `mlp.block` reads a block of statements (here the function body);
- `mlp.id` reads an identifier.
```
-- Read a function definition statement
func_stat = gg.sequence{ "function", func_stat_name, "(",
                         func_params_content, ")", mlp.block, "end" }

-- Read a local function definition statement
func_stat = gg.sequence{ "local", "function", mlp.id, "(",
                         func_params_content, ")", mlp.block, "end" }
```
#### Constructor `gg.sequence (config_table)`
This function returns a sequence parser. `config_table` contains, in its array part, a sequence of strings representing keyword parsers, and arbitrary sub-parsers. Moreover, the following fields are allowed in the hash part of the table:
- `name = <string>`: the parser's name, used to generate error messages;
- `builder = <function>`: if present, whenever the parser is called, the list of the sub-parsers results is passed to this function, and the function's return value is returned as the parser's result. If absent, the list of sub-parser results is simply returned. It can be updated at anytime with:

	`x.builder = <newval>`.
- `builder = <string>`: the string is added as a tag to the list of sub-parser results. `builder = "foobar"` is equivalent to:

	`builder = function(x) x.tag="foobar"; return x end`
- `transformers = <function list>`: applies all the functions of the list, in sequence, to the result of the parsing: these functions must be of type AST->AST. For instance, if the transformers list is `{f1, f2, f3}`, and the builder returns `x`, then the whole parser returns `f3(f2(f1(x)))`.
#### Method `:parse(lexstream)`
Read a sequence from the lexstream. If the sequence can't be entirely read, an error occurs. The result is either the list of results of sub-parsers, or the result of builder if it is non-nil. In the `func_stat` example above, the result would be a list of 3 elements: the results of `func_stat_name`, `func_params_content` and `mlp.block`.

It can also be directly called as simply `x(lexstream)` instead of `x:parse(lexstream)`.

#### Method `.transformers:add(f)`
Adds a function at the end of the transformers list.
### Sequence sets
In many cases, several sequence parsers can be applied at a given point, and the choice of the right parser is determined by the next keyword in the lexstream. This is typically the case for Metalua's statement parser. All the sequence parsers must start with a keyword rather than a sub-parser, and that initial keyword must be different for each sequence parser in the sequence set parser. The sequence set parser takes care of selecting the appropriate sequence parser. Moreover, if the next token in the lexstream is not a keyword, or if it is a keyword but no sequence parser starts with it, the sequence set parser can have a default parser which is used as a fallback.

For instance, the declaration of mlp.stat the Metalua statement parser looks like:
```
mlp.stat = gg.multisequence{
  mlp.do_stat, mlp.while_stat, mlp.repeat_stat, mlp.if_stat... }
```
#### Constructor `gg.multisequence (config_table)`
This function returns a sequence set parser. The array part of `config_table` contains a list of parsers. It also accepts tables instead of sequence parsers: in this case, these tables are supposed to be config tables for the `gg.sequence` constructor, and are converted into sequence parsers on-the-fly by calling `gg.sequence` on them. Each of these sequence parsers has to start with a keyword, distinct from all other initial keywords, e.g. it's illegal for two sequences in the same multisequence to start with keyword `do`; it's also illegal for any parser but the default one to start with a subparser, e.g. `mlp.id`.

It also accepts the following fields in the table's hash part:
- `default = <parser>`: if no sequence can be chosen, because the next token is not a keyword, or no sequence parser in the set starts with that keyword, then the default parser is run instead; if no default parser is provided and no sequence parser can be chosen, an error is generated when parsing.
- `name = <string>`: the parser's name, used to generate arror messages;
- `builder = <function>`: if present, whenever the parser is called, the selected parser's result is passed to this function, and the function's return value is returned as the parser's result. If absent, the selected parser's result is simply returned. It can be updated at any time with `x.builder = <newval>`.
- `builder = <string>`: the string is added as a tag to the list of sub-parser results. `builder = "foobar"` is equivalent to:
`builder = function(x) return { tag = "foobar"; unpack (x) } end`
- `transformers = <function list>`: applies all the functions of the list, in sequence, to the result of the parsing: these functions must be of type AST->AST.
#### Method `:parse(lexstream)`
Read from the lexstream. The result returned is the result of the selected parser (one of the sequence parsers, or the default parser). That result is either returned directly, or passed through `builder` if this field is non-nil.

It can also be directly called as simply `x(lexstream)` instead of `x:parse(lexstream)`.
#### Method `:add(sequence_parser)`
Take a sequence parser, or a config table that would be accepted by `gg.sequence` to build a sequence parser. Add that parser to the set of sequence parsers handled by `x`. Cause an error if the parser doesn't start with a keyword, or if that initial keyword is already reserved by a registered sequence parser, or if the parser is not a sequence parser.
#### Field `.default`
This field contains the default parser, and can be set to another parser at any time.
#### Method `.transformers:add`
Adds a function at the end of the transformers list.
#### Method `:get(keyword)`
Takes a keyword (as a string), and returns the sequence in the set starting with that keyword, or nil if there is no such sequence.
#### Method `:del(keyword)`
Removes the sequence parser starting with keyword `kw`.
### List parser
Sequence parsers allow you to chain several different sub-parsers. Another common need is to read a series of identical elements into a list, but without knowing in advance how many of such elements will be found. This allows you to read lists of arguments in a call, lists of parameters in a function definition, lists of statements in a block, etc.

Another common feature of such lists is that elements of the list are separated by keywords, typically semicolons or commas. These are handled by the list parser generator.

A list parser needs a way to know when the list is finished. When elements are separated by keyword separators, this is easy to determine: the list stops when an element is not followed by a separator. But two cases remain unsolved:
When a list is allowed to be empty, no separator keyword allows the parser to realize that it is in front of an empty list: it would call the element parser, and that parser would fail;
when there are no separator keyword separator specified, they can't be used to determine the end of the list.
For these two cases, a list parser can specify a set of terminator keywords: in separator-less lists, the parser returns as soon as a terminator keyword is found where an element would otherwise have been read. In lists with separators, if terminators are specified, and such a terminator is found at the beginning of the list, then no element is parsed, and an empty list is returned. For instance, for argument lists, `)` would be specified as a terminator, so that empty argument lists `()` are handled properly.

Beware that separators are consumed from the lexstream stream, but terminators are not.

#### Constructor `gg.list (config_table)`
This function returns a list parser. config_table can contain the following fields:
- `primary = <parser>` (mandatory): the parser used to read elemetns of the list;

- `separators = <list>`: list of strings representing the keywords accepted as element separators. If only one separator is allowed, then the string can be passed outside the list:
- `separators = "foo"` is the same as `separators = { "foo" }`.

- `terminators = <list>`: list of strings representing the keywords accepted as list terminators. If only one separator is allowed, then the string can be passed outside the list.

- `name = <string>`: the parser's name, used to generate arror messages;

- `builder = <function>`: if present, whenever the parser is called, the list of primary parser results is passed to this function, and the function's return value is returned as the parser's result. If absent, the list of sub-parser results is simply returned. It can be updated at anytime with `x.builder = <newval>`.

- `builder = <string>`: the string is added as a tag to the list of sub-parser results. `builder = "foobar"` is equivalent to:
`builder = function(x) return { tag = "foobar"; unpack (x) }` end

- `transformers = <function list>`: applies all the functions of the list, in sequence, to the result of the parsing: these functions must be of type AST->AST.

- if keyless element is found in `config_table`, and there is no primary key in the table, then it is expected to be a parser, and it is considered to be the primary parser.
#### Method `:parse (lexstream)`
Read a list from the lexstream. The result is either the list of elements as read by the primary parser, or the result of that list passed through builder if it is specified.

It can also be directly called as simply `x(lexstream)` instead of `x:parse(lexstream)`.
#### Method `.transformers:add(f)`
Adds function `f` at the end of the transformers list.
#### Method .separators:add
Adds a string to the list of separators.
#### Method .terminators:add
Adds a string to the list of terminators.
### Expression parser
This is a very powerful parser generator, but it ensues that its API is quite large. An expression parser relies on a primary parser, and the elements read by this parser can be:
- combined pairwise by infix operators;
- modified by prefix operators;
- modified by suffix operators.
All operators are described in a way analoguous to sequence config tables: a sequence of keywords-as-strings and subparsers in a table, plus the usual builder and transformers fields. Each kind of operator has its own signature for builder functions, and some specific additional information such as precedence or associativity. As in multisequences, the choice among operators is determined by the initial keyword. Therefore, it is illegal to have two operator sequences which start with the same keyword (for instance, two infix sequences starting with keyword "$".

Most of the time, the sequences representing operators will have a single keyword, and no subparser. For instance, the addition operator is represented as: ``{ "+", prec=60, assoc="left", builder= |a, _, b| `Op{ `Add, a, b } }``

#### Infix operators
Infix operators are described by a table whose array-part works as for sequence parsers. Besides this array-part and the usual transformers list, the table accepts the following fields in its hash-part:
- `prec = <number>` is its precedence. The higher the precedence, the tighter the operator bind with respect to other operators. For instance, in Metalua, addition precedence is 60, whereas multiplication precedence is 70.

- `assoc = <string>` is one of "left", "right", "flat" or "none", and specifies how an operator associates. If not specified, the default associativity is "left". 

	Left and right describe how to read sequences of operators with the same precedence, e.g. addition is left associative (1+2+3 reads as (1+2)+3), whereas exponentiation is right-associative (1^2^3 reads as 1^(2^3)).

	If an operator is non-associative and an ambiguity is found, a parsing error occurs. 

	Finally, flat operators get series of them collected in a list, which is passed to the corresponding builder as a single parameter. For instance, if ++ is declared as flat and its builder is f, then whenever 1++2++3++4 is parsed, the result returned is `f{1, 2, 3, 4}`.

- `builder = <function>` is the usual result transformer. The function takes as parameters the left operand, the result of the sequence parser (i.e. `{ }` if the sequence contains no subparser), and the right operand; it must return the resulting AST.
#### Prefix operators
These operators are placed before the sub-expression they modify. They have the same properties as infix operators, except that they don't have an `assoc` field, and `builder` takes `|operator, operand|` instead of `|left_operand, operator, right_operand|`.
#### Suffix operators
Same as prefix operators, except that builder takes `|operand, operator|` instead of `|operator, operand|`.
#### Constructor `gg.expr (config_table)`
This function returns an expression parser. `config_table` is a table of fields which describes the kind of expression to be read by the parser. The following fields can appear in the table:
- `primary` (mandatory): the primary parser, which reads the primary elements linked by operators. In Metalua expression parsers, that would be numbers, strings, and identifiers... It is often a multisequence parser, although that's not mandatory.
- `prefix`: a list of tables representing prefix operator sequences, as described above. It supports a default parser: this parser is considered to have successfully parsed a prefix operator if it returns a non-false result.
- `infix`: a list of tables representing infix operator sequences, as described above. Supports a default parser.
- `suffix`: a list of tables representing suffix operator sequences, as described above. Supports a default parser.
#### Methods `.prefix:add(), .infix:add(), .suffix:add()`
Add an operator in the relevant table. The argument must be an operator sequence table, as described above.

#### Method `:add()`
This is just a shortcut for primary.add. Unspecified behavior if primary doesn't support method add.

#### Method `:parse (lexstream)`
Read a list from the lexstream. The result is built by builder1 calls.

It can also be directly called as simply `x(lexstream)` instead of `x:parse(lexstream)`.

#### Method `:tostring()`
Returns a string representing the parser. Mainly useful for error message generation.
### `onkeyword` parser
Takes a list of keywords and a parser: if the next token is one of the keywords in the list, runs the parser; if not, simply returns false.

Notice that by default, the keyword is consumed by the onkeyword parser. If you want it not to be consumed, but instead passed to the internal parser, add a `peek=true` entry in the config table.

#### Constructor `gg.onkeyword (config_table)`
Create a keyword-conditional parser. `config_table` can contain:
- strings, representing the triggerring keywords (at least one);
- the parser to run if one of the keywords is found (exactly one);
- `peek=<boolean>` to indicate whether recognized keywords must be consumed or passed to the inner parser.

The order of elements in the list is not relevant.

#### Method `:parse (lexstream)`
Run the parser. The result is the internal parser's result, or false if the next token in the lexstream wasn't one of the specified keywords.

It can also be directly called as simply `x(lexstream)` instead of `x:parse(lexstream)`.
### `optkeyword` parser
Watch for optional keywords: an optkeyword parser has a list of keyword strings as a configuration. If such a keyword is found as the next lexstream element upon parsing, the keyword is consumed and that string is returned. If not, false is returned.

#### Constructor `gg.optkeyword (keyword1, keyword2, ...)`
Returns a `gg.optkeyword` parser, which accepts all of the keywords given as parameters, and returns either the found keyword, or false if none is found.
## `mlp`, the metalua parser
The metalua parser is built on top of `gg`, and cannot be understood without some knowledge of it. Basically, `gg` allows not only to build parsers, but to build extensible parsers. Depending on a parser's type (sequence, sequence set, list, expression...), different extension methods are available, which are documented in `gg` reference. The current section will give the information needed to extend Metalua syntax:
- what mlp entries are accessible for extension;
- what do they parse;
- what is the underlying parser type (and therefore, what extension methods are supported)
### Parsing expressions
|name|type|description|
|----|----|-----------|
|mlp.expr|gg.expr|Top-level expression parser, and the main extension point for a Metalua expression. Supports all of the methods defined by gg.expr.|
|mlp.func_val|gg.sequence|Read a function definition, from the arguments' openning parenthesis to the final end, but excluding the initial function keyword, so that it can be used both for anonymous functions, for function some_name(...) end and for local function some_name(...) end.|
|mlp.expr_list|||
|mlp.table_content|gg.list|Read the content of a table, excluding the surrounding braces|
|mlp.table|gg.sequence|Read a litteral table, including the surrounding braces|
|mlp.table_field|custom function|Read a table entry: [foo]=bar, foo=bar or bar.|
|mlp.opt_id|custom function|Try to read an identifier, or an identifier splice. On failure, returns false.|
|mlp.id|custom function|Read an identifier, or an identifier splice. Cause an error if there is no identifier.|
### Parsing statements

|name|type|description|
|----|----|-----------|
|mlp.block|gg.list|Read a sequence of statements, optionally separated by semicolons. When introducing syntax extensions, it's often necessary to add block terminators with mlp.block.terminators:add().|
|mlp.for_header|custom function|Read a for header, from just after the `for` to just before the `do`.|
|mlp.stat|gg.multisequence|Read a single statement.|
Actually, mlp.stat is an extended version of a multisequence: it supports easy addition of new assignment operator. It has field assignments, whose keys are assignment keywords, and values are assignment builders taking left-hand-side and right-hand-side as parameters. for instance, C's `+=` operator could be added as:
```Lua
mlp.lexer:add "+="
mlp.stat.assignments["+="] = function(lhs, rhs)
  assert(#lhs==1 and #rhs==1)
  local a, b = lhs[1], rhs[1]
  return +{stat: (-{a}) = -{a} + -{b} }
end 
```
### Other useful functions and variables
- `mlp.gensym()` generates a unique identifier. The uniqueness is guaranteed, therefore this identifier cannot capture another variable; it is useful in writing hygienic macros.
## Extension `match`: structural pattern matching
Pattern matching is an extremely pleasant and powerful way to handle tree-like structures, such as ASTs. Unsurprisingly, it's a feature found in most ML-inspired languages, which excel at compilation-related tasks. There is a pattern matching extension for metalua, which is extremely useful for most meta-programming purposes.
### Purpose
First, to clear a common misconception: structural pattern matching has absolutely nothing to do with regular expresssions matching on strings: it works on arbitrary structured data. When manipulating trees, you want to check whether they have a certain structure (e.g. a `` `Local`` node with as first child a list of variables whose tags are `` `Id`` ); when you've found a data that matches a certain pattern, you want to name the interesting sub-parts of it, so that you can manipulate them easily; finally, most of the time, you have a series of different possible patterns, and you want to apply the one that matches a given data. These are the needs addressed by pattern matching: it lets you give a list of (pattern -> code_to_execute_if_match) associations, selects the first matching pattern, and executes the corresponding code. Patterns both describe the expected structures and bind local variables to interesting parts of the data. Those variables' scope is obviously the code to execute upon matching success.
#### Match statement
A match statement has the form:
```
match <some_value> with
| <pattern_1> -> <block_1>
| <pattern_2> -> <block_2>
...
| <pattern_n> -> <block_n>
end
```
The first vertical bar after the "with" is optional; moreover, a pattern can actually be a list of patterns, separated by bars. In this case, it's enough for one of them to match, to get the block to be executed:
```
match <some_value> with
| <pattern_1> | <pattern_1_bis> -> <block_1>
...
end
```
When the match statement is executed, the first pattern which matches `<some_value>` is selected, the corresponding block is executed, and all other patterns and blocks are ignored. If no pattern matches, an error "mismatch" is raised. However, we'll see it's easy to add a catch-all pattern at the end of the match, when we want it to be failproof.
### Pattern definitions
#### Atomic literals
Syntactically, a pattern is mostly identical to the values it matches: numbers, booleans and strings, when used as patterns, match identical values.
```Lua
match x with
| 1 -> print 'x is one'
| 2 -> print 'x is two'
end
```
#### Tables
Tables as patterns match tables with the same number of array-part elements, if each pattern field matches the corresponding value field. For instance, {1, 2, 3} as a pattern matches {1, 2, 3} as a value. Pattern {1, 2, 3} matches value {1, 2, 3, foo=4}, but pattern {1, 2, 3, foo=4} doesn't match value {1, 2, 3}: there can be extra hash-part fields in the value, not in the pattern. Notice that field 'tag' is a regular hash-part field, therefore {1, 2, 3} matches `` `Foo{1, 2, 3}`` (but not the other way around). Of course, table patterns can be nested. The table keys must currently be integers or strings. It's not difficult to add more, but the need hasn't yet emerged. If you want to match tables of arbitrary array-part size, you can add a "..." as the pattern's final element. For instance, pattern {1, 2, ...} will match all table with at least two array-part elements whose two first elements are 1 and 2.
#### Identifiers
The other fundamental kind of patterns are identifiers: they match everything, and bind whatever they match to themselves. For instance, pattern 1, 2, x will match value 1, 2, 3, and in the corresponding block, local variable x will be set to 3. By mixing tables and identifiers, we can already do interesting things, such as getting the identifiers list out of a local statement, as mentionned above:
```
match stat with
| `Local{ identifiers, values } ->
   table.foreach(identifiers, |x| print(x[1])
... -- other cases
end
```
When a variable appears several times in a single pattern, all the elements they match must be equal, in the sense of the "==" operator. Fore instance, pattern { x, x } will match value { 1, 1 }, but not { 1, 2 }. Both values would be matched by pattern { x, y }, though. A special identifier is "\_", which doesn't bind its content. Even if it appears more than once in the pattern, metched value parts aren't required to be equal. The pattern "\_" is therefore the simplest catch-all one, and a match statement with a "| \_ ->" final statement will never throw a "mismatch" error.
#### Guards
Some tests can't be performed by pattern matching. For these cases, the pattern can be followed by an "if" keyword, followed by a condition.
```
match x with
| n if n%2 == 0 -> print 'odd'
| _ -> print 'even'
end
```
Notice that the identifiers bound by the pattern are available in the guard condition. Moreover, the guard can apply to several patterns:
```
match x with
| n | {n} if n%2 == 0 -> print 'odd'
| _ -> print 'even'
end
```
#### Multi-match
If you want to match several values, let's say 'a' and 'b', there's an easy way:
```
match {a,b} with
| {pattern_for_a, pattern_for_b} -> block
...
end
```
However, it introduces quite a lot of useless tables and checks. Since this kind of multiple matches are fairly common, they are supported natively:
```
match a, b with
| pattern_for_a, pattern_for_b -> block
...
end
```
This will save some useless tests and computation, and the compiler will complain if the number of patterns doesn't match the number of values.
#### String regular expressions
There is a way to use Lua's regular expressions with match, through the division operator `/`: the left operand is expected to be a literal string, interpreted as a regular expression. The variables it captures are stored in a table, which is matched as a value against the right-hand-side operand. For instance, the following case succeeds when foo is a string composed of 3 words separated by spaces. In case of success, these words are bound to variables w1, w2 and w3 in the executed block:
```
match foo with
| "^(%w+) +(%w+) +(%w+)$"/{ w1, w2, w3 } -> 
   do_stuff (w1, w2, w3)
end
```
### Examples
There are quite a lot of samples using match in the metalua distribution, and more will come in the next one. Dig in the samples for fairly simple usages, or in the standard libs for more advanced ones. Look for instance at examples provided with the `walk` library.
## `walk`, the code walker
When you write advanced macros, or when you're analyzing some code to check for some property, you often need to design a function that walks through an arbitrary piece of code, and does complex stuff on it. Such a function is called a code walker. Code walkers can be used for some punctual adjustments, e.g. changing a function's return statements into something else, or checking that a loop performs no break, up to pretty advanced transformations, such as CPS transformation (a way to encode full continuations into a language that doesn't support them natively; see Paul Graham's On Lisp for an accessible description of how it works), lazy semantics... Anyway, code walkers are tricky to write, can involve a lot of boilerplate code, and are generally brittle. To ease things as much as possible, Metalua comes with a walk library, which intends to accelerate code walker implementation. Since code walking is intrinsically tricky, the lib won't magically make it trivial, but at least it will save you a lot of time and code, when compared to writing all walkers from scratch. Moreover, other people who took the time to learn the walker generator's API will enter into your code much faster.
### Principles
Code walking is about traversing a tree, first from root to leaves, then from leaves back to the root. This tree is not uniform: some nodes are expressions, some others statements, some others blocks; and each of these node kinds is subdivided in several sub-cases (addition, numeric for loop...). The basic code walker just goes from root to leaves and back to root without doing anything. Then it's up to you to plug some action callbacks in that walker, so that it does interesting things for you. Without entering into the details of AST structure, here is a simplified version of the walking algorithm, pretending to work on a generic tree:
```Lua
function traverse(node)
   local down_result = down(node)
   if down_result ~= 'break' then 
      for c in children(node) do
         traverse(node)
      end
   end
   up(node)
end
```
The algorithm behind 'walk' is almost as simple as the one above, except that it's specialized for AST trees. You can essentially specialize it by providing your own up() and down() functions. These visitor functions perform whatever action you want on the AST; moreover, down() has the option to return 'break': in that case, the sub-nodes are not traversed. It allows the user to shun parts of the tree, or to provide his own special traversal method in the down() visitor. The real walker generator is only marginally more complex than that:
- It lets you define your up() and down(), and down() can return 'break' to cut the tree traversal; however, these up() and down() functions are specialized by node kind: there can be separate callbacks for expr nodes, stat nodes, block nodes.
- There's also a binder() visitor for identifier binders. Binders are variables which declare a new local variable; you'll find them in nodes `` `Local``, `` `Localrec``, `` `Forin``, `` `Fornum``, `` `Function``. The binders are visited just before the variable's scope begins, i.e. before traversing a loop or a function's body, after traversing a `` `Local``'s right-hand side, before traversing a `` `Localrec``'s right-hand side.
- Notice that separate up() and down() visitors wouldn't make sense for binders, since they're leave nodes.
- Visitor functions don't only take the visited node as parameter: they also take the list of all expr, stat and block nodes above it, up to the AST's root. This allows a child node to act on its parent nodes.
### API
There are 3 main tree walkers: `walk.expr()`, `walk.stat()` and `walk.block()`, to walk through the corresponding kinds of ASTs. Each of these walker take as parameters a table cfg containing the various visitor functions, and the AST to walk throuhg. the configuration table cfg can contain fields:
- `cfg.stat.down(node, parent, grandparent...)`, which applies when traversing a statement down, i.e. before its children nodes are parsed, and can modify the tree, and return nil or 'break'. The way children are traversed is decided after the down() visitor has been run: this point matters when the visitor modifies its children nodes.
- `cfg.stat.up(node, parent, grandparent...)`, which is applies on the way back up. It is applied even if `cfg.stat.down()` returned 'break', but in that case, the children have not been (and will not be) traversed.
- `cfg.expr.down()` and `cfg.expr.up()`, which work just as their stat equivalent, but apply to expression nodes.
	
	Notice that in Lua, function calls and method invocations can be used as statements as well as as espressions: in such cases, they are visited only by the statement visitor, not by the expression visitor.
- `cfg.block.down()` and `cfg.block.up()` do the same for statements blocks: loops, conditional and function bodies.
- `cfg.binder(identifier, id_parent, id_grandparent...)`: this is run on identifiers which create a new local variable, just before that variable's scope begins.

Moreover, there is a `walk.guess(cfg, ast)` walker which tries to guess the type of the AST it receives, and applies the appropriate walker. When an AST can be either an expression or a statement (nodes `` `Call`` and `` `Invoke``), it is interpreted as an expression.
### Examples
A bit of practice now. Let's build the simplest walker possible, that does nothing:
```Lua
cfg = { }
walker = |ast| walk.block(cfg, ast)
```
Now, let's say we want to catch and remove all statement calls to function assert(). This can be done by removing its tag and content: an empty list is simply ignored in an AST. So we're only interested by `` `Call`` nodes, and within these nodes, we want the function to be `` `Id 'assert'``. All of this is only relevant to stat nodes:
```Lua
function cfg.stat.down (x)
   match x with
   | `Call{ `Id 'assert', ... } -> x.tag=nil; x <- { }
   | _ -> -- not interested by this node, do nothing
   end
end
```
You'll almost always want to use the 'match' extension to implement visitors. The imperative table overrider (x <- y a.k.a. table.override(x, y) also often comes handy to modify an AST. We'll now remove assert() calls in non-statement; we cannot replace an expression by nothing, so we'll replace these nodes by these will simply be replaced by nil:
```Lua
function cfg.expr.down (x)
   match x with
   | `Call{ `Id 'assert', ... } -> x <- `Nil
   | _ -> -- not interested by this node, do nothing
   end
end
```
Here's a remark for functional programmers: this API is very imperative; you might cringe at seeing the `` `Call`` nodes transformed in-place. Well, I tend to agree but it's generally counter-productive to work against the grain of the wood: Lua is imperative at heart, and design attempts at doing this functionally sucked more than approaches that embraced imperativeness.
### Cuts
By making down() return 'break', you can prevent the traversal to go further down. This might be either because you're not interested by the subtrees, or because you want to traverse them in a special way. In that later case, just do the traversal by yourself in the down() function, and cut the walking by returning 'break', so that nodes aren't re-traversed by the default walking algorithm. We'll see that in the next, more complex example, listing of free variables. This example is exclusively there for demonstration purposes. For actual work on identifiers that require awareness of an identifier's binder of freedom, there is a dedicated walk.id library. We'll progressively build a walker that gathers all global variables used in a given AST. This involves keeping, at all times, a set of the identifiers currently bound by a "local" declaration, by function parameters, as for loop variables etc. Then, every time an identifier is found in the AST, its presence is checked in the current set of bound variables. If it isn't in it, then it's a free (global) identifier. The first thing we'll need is a scope handling system: something that keeps track of what identifiers are currently in scope. It must also allow to save the current scope (e.g. before we enter a new block) and restore it afterwards (e.g. after we leave the block). This is quite straightforward and unrelated to code walking; here is the code:
```Lua
require 'std'
require 'walk'

-{ extension 'match' }

--------------------------------------------------------------------------------
-- Scope handling: ':push()' saves the current scope, ':pop()'
-- restores the previously saved one. ':add(identifiers_list)' adds
-- identifiers to the current scope. Current scope is stored in
-- '.current', as a string->boolean hashtable.
--------------------------------------------------------------------------------

local scope = { }
scope.__index = scope

function scope:new()
   local ret = { current = { } }
   ret.stack = { ret.current }
   setmetatable (ret, self)
   return ret
end

function scope:push()
   table.insert (self.stack, table.shallow_copy (self.current))
end

function scope:pop()
   self.current = table.remove (self.stack)
end

function scope:add (vars)
   for id in values (vars) do
      match id with `Id{ x } -> self.current[x] = true end
   end
end
```
(There is an improved version of that class in library walk.scope; cf. its documentation for details). Now let's start designing the walker. We'll keep a scope object up to date, as well as a set of found free identifiers, gathered every time we find an `` `Id`` node. To slightly simplify matter, we'll consider that the AST represent a block.
```Lua
local function fv (term)
   local freevars = { }
   local scope    = scope:new()
   local cfg      = { expr  = { } }

   function cfg.expr.down(x)
      match x with
      | `Id{ name } -> if not scope.current[name] then freevars[name] = true end
      | _ -> -- pass
      end
   end

   walk.guess(cfg, term)
   return freevars
end
```
Since we don't ever add any identifier to the scope, this will just list all the identifiers used in the AST, bound or not. Now let's start doing more interesting things:
We'll save the scope's state before entering a new block, and restore it when we leave it. That will be done by providing functions cfg.block.down() and cfg.block.up(). Saving and restoring will be performed by methods :push() and :pop().
Whenever we find a local declaration, we'll add the list of identifiers to the current scope, so that they won't be gathered when parsing the `Id expression nodes. Notice that the identifiers declared by the 'local' statement only start being valid after the statement, so we'll add them in the cfg.stat.up() function rather than cfg.stat.down().
```Lua
local cfg = { expr  = { },
              stat  = { },
              block = { } }
[...]
function cfg.stat.up(x)
	match x with
	| `Local{ vars, ... } -> scope:add(vars)
	| _ -> -- pass
	end
end

-------------------------------------------------------------------
-- Create a separate scope for each block, close it when leaving.
-------------------------------------------------------------------
function cfg.block.down() scope:push() end
function cfg.block.up()   scope:pop()  end
```
This starts to be useful. We can also easily add the case for `` `Localrec`` nodes (the ones generated by "local function foo() ... end"), where the variable is already bound in the `` `Localrec`` statement's right-hand side; so we do the same as for `` `Local``, but we do it in the down() function rather than in the up() one. We'll also take care of `` `Function``, `` `Forin`` and `` `Fornum`` nodes, which introduce new bound identifiers as function parameters or loop variables. This is quite straightforward; the only thing to take care of is to save the scope before entering the function/loop body (in down()), and restore it when leaving (in up()). The code becomes:
```Lua
local function fv (term)
   local freevars = { }
   local scope    = scope:new()
   local cfg      = { expr  = { },
                      stat  = { },
                      block = { } }

   -- Check identifiers; add functions parameters to newly created scope.
   function cfg.expr.down(x)
      match x with
      | `Id{ name } -> if not scope.current[name] then freevars[name] = true end
      | `Function{ params, _ } -> scope:push(); scope:add (params)
      | _ -> -- pass
      end
   end

   -- Close the function scope opened by 'down()'.
   function cfg.expr.up(x)  
      match x with
      | `Function{ ... } -> scope:pop()
      | _ -> -- pass
      end
   end

   -- Create a new scope and register loop variable[s] in it
   function cfg.stat.down(x)
      match x with
      | `Forin{ vars, ... }    -> scope:push(); scope:add(vars)
      | `Fornum{ var, ... }    -> scope:push(); scope:add{var}
      | `Localrec{ vars, ... } -> scope:add(vars)
      | `Local{ ... }          -> -- pass
      | _ -> -- pass
      end
   end

   -- Close the scopes opened by 'up()'
   function cfg.stat.up(x)
      match x with
      | `Forin{ ... } | `Fornum{ ... } -> scope:pop()
      | `Local{ vars, ... }            -> scope:add(vars)
      | _ -> -- pass
      end
   end

   -- Create a separate scope for each block, close it when leaving.
   function cfg.block.down() scope:push() end
   function cfg.block.up()   scope:pop()  end

   walk.guess(cfg, term)
   return freevars
end
```
This is almost correct now. There's one last tricky point of Lua's semantics that we need to address: in repeat foo until bar loops, "bar" is included in "foo"'s scope. For instance, if we write `repeat local x=foo() until x>3`, the "x" in the condition is the local variable "x" declared inside the body. This violates our way of handling block scopes: the scope must be kept alive after the block is finished. We'll fix this by providing a custom walking for the block inside `` `Repeat``, and preventing the normal walking to happen by returning 'break':
```Lua
-----------------------------------------------------------
-- Create a new scope and register loop variable[s] in it
-----------------------------------------------------------
function cfg.stat.down(x)
	match x with
	| `Forin{ vars, ... }    -> scope:push(); scope:add(vars)
	| `Fornum{ var, ... }    -> scope:push(); scope:add{var}
	| `Localrec{ vars, ... } -> scope:add(vars)
	| `Local{ ... }          -> -- pass
	| `Repeat{ block, cond } -> -- 'cond' is in the scope of 'block'
			scope:push()
			for s in values (block) do walk.stat(cfg)(s) end -- no new scope
			walk.expr(cfg)(cond)
			scope:pop()
			return 'break' -- No automatic walking of subparts 'cond' and 'body'
	| _ -> -- pass
	end
end
```
That's it, we've now got a full free variables lister, and have demonstrated most APIs offered by the basic 'walk' library. If you want to walk through identifiers in a scope-aware way, though, you'll want to look at the walk.id library.
### Library walk.id, the scope-aware walker
This library walks AST to gather information about the identifiers in it. It calls distinct visitor functions depending on whether an identifier is bound or free; moreover, when an identifier is bound, the visitor also receives its binder node as a parameter. For instance, in `+{function(x) print(x) end}`, the bound identifier walker will be called on the `+{x}` in the print call, and the visitor's second parameter will be the `` `Function`` node which created the local variable `x`.
#### API
The library is loaded with require 'walk.id'. The walkers provided are:
- `walk_id.expr()`;
- `walk_id.stat()`;
- `walk_id.block()`;
- `walk_id.guess()`.
They take the same config tables as regular walkers, except that they also recognize the following entries:
- `cfg.id.free(identifier, parent, grandparent...)`, which is run on free variables;
- `cfg.id.bound(identifier, binder, parent, grandparent...)`, which is run on bound variables. The statement or expression which created this bound veriable's scope is passed as a second parameter, before the parent nodes.
#### Examples
Let's rewrite the free variables walker above, with the id walker:
```Lua
function fv (term)
   local cfg = { id = { } }
   local freevars = { }
   function cfg.id.free(id)
      local id_name = id[1]
      freevars[id_name] = true
   end
   walk_id.guess (cfg, term)
   return freevars
end
```
Now, let's rename all bound variables in a term. This is slightly trickier than one could think: we need to first walk the whole tree, then perform all the replacement. If we renamed binders as we went, they would stop binding their variables, and something messy would happen. For instance, if we took `+{function(x) print(x) end}` and renamed the binder `x` into `foo`, we'd then start walking the function body on the tree `+{function(foo) print(x) end}`, where `x` isn't bound anymore.
```Lua
--------------------------------------------------------------------------------
-- bound_vars keeps a binder node -> old_name -> new_name dictionary.
-- It allows to associate all instances of an identifier together,
-- with the binder that created them
--------------------------------------------------------------------------------
local bound_vars    = { }

--------------------------------------------------------------------------------
-- local_renames will keep all identifier nodes to rename as keys,
-- and their new name as values. The renaming must happen after 
-- the whole tree has been visited, in order to avoid breaking captures.
--------------------------------------------------------------------------------
local local_renames = { }

--------------------------------------------------------------------------------
-- Associate a new name in bound_vars when a local variable is created.
--------------------------------------------------------------------------------
function cfg.binder (id, binder)
   local old_name         = id[1]
   local binder_table     = bound_vars[binder]
   if not binder_table then
      -- Create a new entry for this binder:
      binder_table        = { }
      bound_vars[binder]  = binder_table
   end
   local new_name         = mlp.gensym(old_name)[1] -- generate a new name
   binder_table[old_name] = new_name -- remember name for next instances
   local_renames[id]      = new_name -- add to the rename todo-list
end

--------------------------------------------------------------------------------
-- Add a bound variable the the rename todo-list
--------------------------------------------------------------------------------
function cfg.id.bound (id, binder)
   local old_name    = id[1]
   local new_name    = bound_vars[binder][old_name]
   local_renames[id] = new_name
end

-- walk the tree and fill laocal_renames:
walk_id.guess(cfg, ast)

-- perform the renaming actions:
for id, new_name in pairs(local_renames) do id[1] = new_name end
```
### Library `walk.scope`, the scope helper
This library allows to easily store data, in an AST walking function, in a scope aware way. Cf. comments in the sources for details.
## `dollar`: generic syntax for macros
When you write a short-lived macro which takes reasonably short arguments, you generally don't want to write a supporting syntax for it. The dollar library allows you to call it in a generic way: if you store your macro in the table mlp.macros, say as function mlp.macros.foobar, then you can call it in your code as $foobar(arg1, arg2...): it will receive as parameters the ASTs of the pseudo-call's arguments.
### Example
```Lua
-{ block:
   require 'dollar'
   function doller.LOG(id)
      match id with `Id{ id_name } -> 
         return +{ printf("%s = %s", id_name, 
                          table.tostring(-{id})) }
      end
   end }

local x = { 1, 2, 3 }
$LOG(x) -- prints "x = { 1, 2, 3 }" when executed
```
# Samples and tutorials
## Advanced examples
This section will present the implementation of advanced features. The process is often the same:
1. "That language has this nifty feature that would really save my day!"
2. Implement it as a metalua macro.
3. ???
4. Profit!

The other common case for macro implementation is the development of a domain-specific language. I'll eventually write a sample or two demonstrating how to do this.
## Exceptions
As a first non-trivial example of extension, we'll pick exception: there is a mechanism in Lua, pcall(), which essentially provides the raw functionality to catch errors when some code is run, so enhancing it to get full exceptions is not very difficult. We will aim at:
- Having a proper syntax, the kind you get in most exception-enabled languages;
- being able to easily classify exception hierarchically;
- being able to attach additional data to exception (e.g. an error message);
- not interfere with the usual error mechanism;
- support the `finally` feature, which guaranties that a piece of code (most often about resource liberation) will be executed.
## Syntax
There are many variants of syntaxes for exceptions. I'll pick one inspired from OCaml, which you're welcome to dislike. And in case you dislike it, writing one which suits your taste is an excellent exercise. So, the syntax for exceptions will be something like:

```Lua
try
   <protected block of statements>
with
  <exception_1> -> <block of statements handling exception 1>
| <exception_2> -> <block of statements handling exception 2>
  ...
| <exception_n> -> <block of statements handling exception n>
end
```
Notice that OCaml lets you put an optional `|` in front of the first exception case, just to make the whole list look more regular, and we'll accept it as well. Let's write a `gg` grammar parsing this:

```Lua
trywith_parser = 
   gg.sequence{ "try",  mlp.block,  "with",  gg.optkeyword "|",
                gg.list{ gg.sequence{ mlp.expr,  "->",  mlp.block },
                         separators = "|", terminators = "end" },
                "end", 
                builder = trywith_builder }
mlp.stat:add(trywith_parser)
mlp.lexer:add{ "try", "with", "->" }
mlp.block.terminator:add{ "|", "with" }
```
We use `gg.sequence` to chain the various parsers; `gg.optkeyword` lets us allow the optional `|`; `gg.list` lets us read an undetermined series of exception cases, separated by keyword `|`, until we find the terminator `end` keyword. The parser delegates the building of the resulting statement to `trywith_builder`, which will be detailed later. Finally, we have to declare a couple of mundane things:
- that `try`, `with` and `->` are keywords. If we don't do this, the two firsts will be returned by the lexer as identifiers instead of keywords; the later will be read as two separate keywords `-` and `>`. We don't have to declare explicitly `|`, as single-character symbols are automatically considered to be keywords.
- that `|` and `with` can terminate a block of statements. Indeed, metalua needs to know when it reached the end of a block, and introducing new constructions which embed blocks often introduce new block terminators. In our case, `with` marks the end of the block in which exceptions are monitored, and `|` marks the beginning of a new exception handling case, and therefore the end of the previous case's block.

That's it for syntax, at least for now. The next step is to decide what kind of code we will generate.

The fundamental mechanism is `pcall(func, arg1, arg2, ..., argn)`: this call will evaluate `func(arg1, arg2, ..., argn)`, and:
- if everything goes smoothly, return `true`, followed by any value(s) returned by `func()`;
- if an error occurs, return `false`, and the error object, most often a string describing the error encountered.

We'll exploit this mechanism, by enclosing the guarded code in a function, calling it inside a `pcall()`, and using special error objects to represent exceptions.
## Exception objects
We want to be able to classify exceptions hierarchically: each exception will inherit from a more generic exception, the most generic one being simply called `exception`. We'll therefore design a system which allows to specialize an exception into a sub-exception, and to compare two exceptions, to know whether one is a special case of another. Comparison will be handled by the usual `< > <= >=` operators, which we'll overload through metatables. Here is an implementation of the base exception exception, with working comparisons, and a `new()` method which allow to specialize an exception. Three exceptions are derived as an example, so that `exception > exn_invalid > exn_nullarg and exception > exn_nomorecoffee`:

```Lua
exception = { } ; exn_mt = { } 
setmetatable (exception, exn_mt)

exn_mt.__le = |a,b| a==b or a<b 
exn_mt.__lt = |a,b| getmetatable(a)==exn_mt and 
                    getmetatable(b)==exn_mt and
                    b.super and a<=b.super

function exception:new()
   local e = { super = self, new = self.new }
   setmetatable(e, getmetatable(self))
   return e
end

exn_invalid     = exception:new()
exn_nullarg     = exn_invalid:new()
exn_nomorecofee = exception:new()
```
To compile a try/with block, after having put the guarded block into a `pcall()` we need to check whether an exception was raised, and if it has been raised, compare it with each case until we find one that fits. If none is found (either it's an uncaught exception, or a genuine error which is not an exception at all), it must be rethrown. 

Notice that throwing an exception simply consists of sending it as an error:

```
throw = error
```
To fix the picture, here is some simple code using try/catch, followed by its translation:

```Lua
-- Original code:
try
   print(1)
   print(2)
   throw(exn_invalid:new("toto"))
   print("You shouldn't see that")
with
| exn_nomorecofee -> print "you shouldn't see that: uncomparable exn"
| exn_nullarg     -> print "you shouldn't see that: too specialized exn"
| exn_invalid     -> print "exception caught correctly"
| exception       -> print "execution should never reach that far"
end 
print("done")


-- Translated version:
local status, exn = pcall (function ()
   print(1)
   print(2)
   throw(exn_invalid)
   print("You shouldn't see that")
   end)

if not status then
   if exn < exn_nomorecoffee then
      print "you shouldn't see that: uncomparable exn"
   elseif exn < exn_nullarg then
      print "you shouldn't see that: too specialized exn"
   elseif exn < exn_invalid then
      print "exception caught correctly"
   elseif exn < exception then
      print "execution should never reach that far"
   else error(exn) end
end 
print("done")
```
In this, the only nontrivial part is the sequence of if/then/elseif tests at the end. If you check the doc about AST representation of such blocks, you'll come up with some generation code which looks like:

```Lua
function trywith_builder(x)
   ---------------------------------------------------------
   -- Get the parts of the sequence:
   ---------------------------------------------------------
   local block, _, handlers = unpack(x)

   ---------------------------------------------------------
   -- [catchers] is the big [if] statement which handles errors
   -- reported by [pcall].
   ---------------------------------------------------------
   local catchers = `If{ }
   for _, x in ipairs (handlers) do
      -- insert the condition:
      table.insert (catchers, +{ -{x[1]} <= exn })
      -- insert the corresponding block to execute on success:
      table.insert (catchers, x[2])
   end

   ---------------------------------------------------------
   -- Finally, put an [else] block to rethrow uncought errors:
   ---------------------------------------------------------
   table.insert (catchers, +{error (exn)})

   ---------------------------------------------------------
   -- Splice the pieces together and return the result:
   ---------------------------------------------------------
   return +{ block:
      local status, exn  = { pcall (function() -{block} end) }
      if not status then
         -{ catchers }
      end }
end
```
## Not getting lost between metalevels
This is the first non-trivial example we see, and it might require a bit of attention in order not to be lost between metalevels. Parts of this library must go at metalevel (i.e. modify the parser itself at compile time), other parts must be included as regular code:
- trywith_parser and trywith_builder are at metalevel: they have to be put between `-{...}`, or to be put in a file which is loaded through `-{ require ... }`.
- the definitions of throw, the root exception and the various derived exceptions are regular code, and must be included normally.
The whole result in a single file would therefore look like:

```Lua
-{ block:
   local trywith_builder = ...
   local trywith_parser  = ...
   mlp.stat:add ...
   mlp.lexer:add ...
   mlp.block.terminator:add ... }

throw     = ...
exception = ...
exn_mt    = ...

exn_invalid     = ...
exn_nullarg     = ...
exn_nomorecofee = ...

-- Test code
try
   ...
with
| ... -> ...
end
```
Better yet, it should be organized into two files:
the parser modifier, i.e. the content of `-{block:...}` above, goes into a file `ext-syntax/exn.lua` of Lua's path;
the library part, i.e. `throw ... exn_nomorecoffee ...` goes into a file `ext-lib/exn.lua` of Lua's path;
the sample calls them both with metalua standard lib's extension function:

```Lua
-{ extension "exn" }
try
  ...
with
| ... -> ...
end
```
## Shortcomings
This first attempt is full of bugs, shortcomings and other traps. Among others:
- Variables exn and status are subject to capture;
- There is no way to put personalized data in an exception. Or, more accurately, there's no practical way to retrieve it in the exception handler.
- What happens if there's a return statement in the guarded block?
- There's no finally block in the construction.
- Coroutines can't yield across a `pcall()`. Therefore, a yield in the guarded code will cause an error.

Refining the example to address these shortcomings is left as an exercise to the reader, we'll just give a couple of design hints. However, a more comprehensive implementation of this exception system is provided in metalua's standard libraries; you can consider its sources as a solution to this exercise!
## Hints
Addressing the variable capture issue is straightforward: use `gg.gensym()` to generate unique identifiers (which cannot capture anything), and put anti-quotes at the appropriate places. Eventually, metalua will include a hygienization library which will automate this dull process. 

Passing parameters to exceptions can be done by adding arbitrary parameters to the `new()` method: these parameters will be stored in the exception, e.g. in its array part. finally, the syntax has to be extended so that the caught exception can be given a name. Code such as the one which follows should be accepted:
```Lua
try
   ...
   throw (exn_invalid:new "I'm sorry Dave, I'm afraid I can't do that.")
with
| exn_invalid e -> printf ("The computer choked: %s", e[1])
end
```
The simplest way to detect user-caused returns is to create a unique object (typically an empty table), and return it at the end of the block. when no exception has been thrown, test whether that object was returned: if anything else than it was returned, then propagate it (by returning it again). If not, do nothing. Think about the case when multiple values have been returned.

The finally block poses no special problem: just go through it, whether an exception occured or not. Think also about going through it even if there's a return to propagate.

As for yielding from within the guarded code, there is a solution, which you can find by searching Lua's mailing list archives. The idea is to run the guarded code inside a coroutine, and check what's returned by the coroutine run:
- if it's an error, treat it as a pcall() returning false;
- if it's a normal termination, treat it as a pcall() returning true;
- if it's a yield, propagate it to the upper level; when resumed, propagate the resume to the guarded code which yielded.
