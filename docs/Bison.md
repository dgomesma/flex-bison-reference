# Introduction
## Overview
Bison is a LALR (Look-Ahead, Left-to-Right) [parser](https://en.wikipedia.org/wiki/Parsing) and is able to perform syntax analysis given a string of symbols. Bison is the successor of Yacc and is part of the GNU Project. It works very well in conjunction with Lex/Flex, and usually you want to use it in conjunction with it.

Bison works by shifting a pointer along the string and performing reductions. For each reduction, which takes place when a production is applicable, some code may get executed. In the context of a compiler, that code usually generates the target code.
## Lex/Flex
Bison usually works in conjunction with Flex for identifying terminal symbols (in Flex terminology, the tokens), and a reference/tutorial for it is available as well [in here](https://github.com/dgomesma/flex-bison-reference/blob/main/docs/Flex.md). Given the similarities between both file structures and how Bison relies on Flex for lexical analysis, I recommend first understanding how Flex works before moving on to this reference.
## Theory Review
Bison works by shifting from left to right and reducing the symbols until it eventually reaches the initial symbol of the context-free grammar (CFG).

A context-free grammar is defined by:
1. A set of terminals (in the context of Bison, the tokens)
2. A set of non-terminals
3. A start symbol
4. Rules of production

For instance, suppose we have the following CFG:
```
Terminals: NUMBER, '+', '*'
Non-terminals: expr, term
Start Symbol: expr
Rules of Production:
expr -> expr '+' expr 
| expr '*' expr
| term
```

Then the LALR will run a pointer through the input, **shifting** from **left-to-right** and **reducing** the terminal symbols to non-terminal symbols until it eventually reaches the start symbol.

Suppose, for instance, we want a LALR parser to parse 2 + 3 (and suppose `^`) is the pointer:
```
^ 2 + 3
2 ^ + 3 (shift)
expr ^ + 3 (reduce)
expr + ^ 3 (shift)
expr + 3 ^ (shift)
expr + expr ^ (reduce)
expr (reduce)
```

Note, however, that if we give a string like 2 + 3 + 4 to the parser, it will enter in a conflict at the step `expr + expr ^ + 3`. That is: should it reduce `expr + expr -> expr` or shift to the right? This is called a **shift-reduce conflict**. To solve it, associativity rules must be defined:
```
%left '+'
%left '*'
```

In this way, we are defining both operations to have left-associativity, giving higher precedence to '\*' (in Bison, higher precedence is given to the symbols whose associativity have last been defined).

Now, since '\*' has a higher precedence than '+', the parser will favour shift over reducing, and thus the parsing will proceed as follows:
```
expr + expr ^ * 7
expr + expr * ^ 7
expr + expr * 7 ^
expr + expr * expr ^
expr + expr ^
expr
```

In that way the parser should be able to resolve shift-reduce conflicts and reduce a string of symbols to its initial symbol. In Bison, however, if a lack of specification causes a shift-reduce conflict to be undecidable, then Bison should crash.
# Bison File
A Bison file structure is similar to a Flex file:
```c
/* Prologue: C code and includes */
%{
#include <stdio.h>
#include <stdlib.h>
%}

/* Definitions: Bison Directives and Type Declarations */
%union {
    int intValue;
}

%token <intValue> INTEGER
%left '+'
%left '*'
%type <intValue> expr

%%

/* Grammar Rules */

expr:
      INTEGER                { $$ = $1; }
    | expr '+' expr          { $$ = $1 + $3; }
    | expr '*' expr          { $$ = $1 * $3; }
    | '(' expr ')'           { $$ = $2; }
    ;

%%

/* Epilogue: Additional C code */

int main() {
    printf("Enter an arithmetic expression: ");
    yyparse();
    return 0;
}

void yyerror(const char* s) {
    fprintf(stderr, "Error: %s\n", s);
}
```
## Prologue
The prologue is where you keep C code headers, definitions, macros, and everything else that you need to declare or initialise to be used later, including in the Grammar Rules section. It includes everything enclosed by the `%{` and the `%}`.
## Definitions
In the definitions section of the Bison file is where you keep the Bison directives defining the tokens (terminal symbols), semantic values of rules, associativity rules, and semantic values data types in a union.
## Grammar Rules
This is where the actual grammar rules go. This is the syntax that Bison expects the input to conform to. When Bison matches the input with any of the right-hand side productions, it executes the code associated with it. If unable to match with any rule, then it encounters an error.
##  Epilogue
This is where you keep additional C/C++ code. Usually this code won't be sufficient to have a stand-alone running program like you are able to do in Flex since Bison relies on Flex for lexical analysis.
# Bison Directives
## Tokens
Tokens comprise the primitives of the grammar. They are the _terminal symbols_ since they are not comprised of other symbols. Tokens can be declared with the bison directive `%token`:

```c
%token <type> TOKEN_NAME1 TOKEN_NAME2 ... "descriptive"
```
The `%token` syntax is comprised of the following fields:
- `%token`: Determines that one or multiple tokens will be defined.
- `<type>` (optional): The optional semantic value type. For instance, when Flex scans a number, it might return a token `T_NUM`, but you might also want to know the value of what was scanned. Flex can do it (as will be described later).
- `TOKEN_NAME1`, `TOKEN_NAME2` ...: One or more tokens with the semantic value `type`. Having more than one token is optional.
- `"descriptive"` (optional): The descriptive string. May be used for providing a more descriptive to the tokens when, for instance, reporting errors.
## Left/Right
The `%left` and `%right` directives specify the associativity of the grammar tokens. The syntax is as follows:

```
%left TOKEN1 TOKEN2 ...
```

The same syntax applies for `%right`. 

These tokens help resolve shift-reduce conflicts by enforcing a decision to be taken. For instance, if we have
~~~C
%left '+'
~~~

Then if given the expression `1 + 2 + 3` and the CFG described in the Theory Review section, then when the parser reaches the state `expr + expr ^ + 3`, rather than shifting to the right, it will evaluate `expr + expr -> expr`, resulting in `expr ^ + 3`. In other words, this makes it so that the expression is evaluated in that order: `(1 + 2) + 3`.

On the other hand, if we had
~~~C
%right '+'
~~~

Then at `expr + expr + 3`, the parser would shift right, and these following steps would ensue:
~~~
expr + expr ^ + 3
expr + expr + ^ 3 (shift) 
expr + expr + 3 ^ (shift)
expr + expr + expr ^ (reduce)
expr + expr ^ (reduce)
expr ^ (reduce)
~~~

Thus, it would make it equivalent to evaluating `1 + (2 + 3)`.
## Noassoc
The `%nonassoc` token tells Bison that the terminal has no associativity, causing a syntax error when a decision regarding associativity is required.

For instance, the comparison operator `<` is one such example where associativity probably doesn't make much sense. For instance, if given `2 < 3 < 4`, what should be reduced first? Then, by stating
~~~
%noassoc '<'
~~~

Bison will raise a syntax error when it encounters such an issue. Note that, by default, Bison will prefer to shift to the right in such scenarios and raise a warning, so having `%noassoc` may be helpful in raising syntax errors.

## Precedence
Precedence is another factor that helps determine the resolution of a shift-reduce conflict. Suppose you wanna assign the left associativity to `+` and to `*`, so you do the following:

~~~c
%left '+' '*'
~~~

Then what will be the result of `1 + 2 * 3`? Will it be 7 or 9? If you guessed 9, then you guessed it correctly. Because both terminals are declared left-associative in the same directive, they will have the same precedence.

Precedence is determined by their order in which their associativity is determined, where the terminals whose associativity have been first declared get a lower precedence than terminals whose associativity have been last declared. Thus, to correct the precedence for these operations to follow standard order of operations in mathematics, we would need the following:
```C
%left '+'
%left '*'
```

In that manner, when the parser reaches the state `expr + expr ^ * 3`, it will decide to shift to the right rather than reducing `expr + expr`.