# Introduction
## Overview
Bison is a LALR (Look-Ahead, Left-to-Right) [parser](https://en.wikipedia.org/wiki/Parsing) and is able to perform syntax analysis given a string of symbols, or in the context of something as a compiler, given a string of tokens.Bison is the successor of Yacc and is part of the GNU Project. It works very well in conjunction with Lex/Flex, and usually you want to use it in conjunction with it.

Bison works by reading a string of tokens and whenever it matches with a formal rule (a valid syntax), it executes some code. If you are writing a compiler, this code might involve generating some code.
## Lex/Flex
Bison usually works in conjunction with Flex, and a reference/tutorial for it is available as well [in here](https://github.com/dgomesma/flex-bison-reference/blob/main/docs/Flex.md). Given the similarities between both file structures and how Bison relies on Flex for lexical analysis, I recommend first understanding how Flex works before moving on to this reference./
