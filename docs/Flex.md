## Disclaimer
This is a "draft". It has not been reviewed, it is unfinished, it might contain inaccuracies, and I have not verified that any of the code runs or that the terminology is technically precise. I have used Flex long time ago, and I started writing this as a review for myself, but decided to share with the world. Nonetheless, it might be an useful summary in a relatively simplified language if you just want to quickly learn something new.
## Overview
`Flex` is a GNU tool for generating _scanners_ from `.flex` (or `.lex`) files (it seems `lex` is the predecessor of `lex`, but they are similar in many ways). These _scanners_ will read a stream of text, match regexes, and, upon match, run some C code. `Flex` is incredibly useful for making _lexers_, or programs which performs [lexical analysis](https://en.wikipedia.org/wiki/Lexical_analysis). In short, this is the step of finding the _lexical tokens_ of a program. 

As a scanner generator, it will read a `.lex` file and output a `.yy.c` file, which can finally be compiled to an object or a binary.
## Theory Review
The _lexical tokens_ (or just tokens for short) are everything that is in a program. E.g., in the following stupid C program:
```C

int main(int argc, char* argv[]){
  int n = 2;

  if(n % 2 == 0) {
	  // Oh, wow! The number 2 is even! All is good!
	  return 0;
  } else {
	  // Oh no! We have an odd number! Error!
	  return 1;
  }   
}
```

The following are examples of tokens:
```
int
main
(
int
argc
,
char*
argv
[]
)
{
=
2
%
else
return
1
// Oh wow! The number 2 is even! All is good!
}
```

These do not necessarily correspond to how C compilers' actual lexical analysis, but it is a possible one nevertheless. Tokens are not necessarily a set of characters delimited by whitespace, but they can be. Each token is a set of character that correspond to a pattern. For instance, when the lexer finds a "//", then it will match these backlashes and every character that follows it until the end of the line. All these characters comprise a single `comment` token. A token may have one or multiple possible values. E.g. both comments in the code above are examples of `comment` tokens, but the token `right_curly_bracket`'s only value is likely "}". 

Tokens in a specific order specify a _rule_ of a _grammar_. E.g. a rule `binary_op` could be defined as follows:
```
binary_op -> expression binary_operator expression
```

Where `expression` is a rule defining, well, expressions that can be evaluated, and `binary_operator` is also a rule defined by a `binary_op` token, such as "+", "-", "\*", "%", "&&", "||", "<<", ">>", "&", "|". Note that rules can be comprised of other rules or of tokens, and **order matters**. And a _grammar_ is a set of _rules_.

In that way, in the big picture of a compiler, `flex` will allow for finding tokens, which may be used by _parsers_ (such as `bison`) for matching rules, and in `bison`, whenever a rule gets matched, code gets generated somehow.
# Flex File
## File Structure
The structure of a Lexer file is generally as follows:
```c
%{
// C headers, includes, and other definitions
#include <stdio.h>

typedef enum {
  T_EXCLAMATION
  T_LETTER
  // Other definitions
} t_token;
//...
%}
  // Regex definitions
letter    [a-zA-Z]
 //...

%s state1 
%x state2
%%
 // Rules
 // A rule matching a literal
\!             { return T_EXCLAMATION; }
 // A rule matching a regex definition
letter         { return T_LETTER; }
 // A rule that takes effect when the lexer is on state <initial>
<state1>\!     { BEGIN(state2); return T_EXCLAMATION_BUT_DIFFERENT; }
%%
// Plain-old C code

// Lots of code in here

int main() {
	// Do stuff
	return 0;
}
```

As you can see, a Flex file is divided into three sections by the `%%` as follows:

```
definitions
%%
rules
%%
code
```

## Definitions
This is where you want to include some C code in between the `%{` and the `%}`. Some code that might be useful there include headers, `include` directives, macro definitions, global variables, and so forth. In the `rules` section, some code might be run, and we can use anything that has been defined in that section of the code.

Outside the `%{` and `%}` enclosed C code, we can include pattern definitions to make our code more readable. For instance, rather than having a rule as follows:
```C
[a-zA-Z]                   { return T_LETTER; }
```

We can have the following:
```c
// ...
letter       [a-zA-Z]
%%

letter                     { return T_LETTER; }
```

## Rules
Here we define the rules. Rules are a relation between patterns and code, where, when scanning, if the pattern is matched, the code is run. The code run may or may not return. If it does not return, the code will get executed and then the scanner will continue looking for matches. If it returns, it must return an `int`, and scanning stops until `yylex` is called again.

Here is one examples of some rules:
```c
[0-9]                   { printf("Found a digit!\n"); }
[a-zA-Z]                { return T_LETTER; }
```

On the left, we have a regex pattern the scanner will look for. Upon finding a match, the code on the right gets run. In the example above, if a digit is found, it will print "Found a digit!" line. If it finds a character, it will stop scanning and return a `T_LETTER` token, where `T_LETTER` is actually an enum encoding some integer value.

## User Code
Here you can have regular C code. You may or may not have a `main` here. You may or may not call `yylex` to do scanning here. You may or may not have any code at all here. But this section allows you to have a single Flex file with all the code that you need for writing a simple program, although, for larger programs I believe it is best to use that section only for lexer-related function definitions.
## Patterns
As probably previously stated, flex matches through regex. Regex patterns can either be defined in a way similar to how macros get defined, like above, `letter [a-zA-Z]`, and later be used to make other definitions (e.g. `alphanumeric ({digit} | {letter} )+` or it can be used in rules:
```C 
// Instead of
letter         { return T_LETTER; }
// We could have
[a-zA-Z]       { return T_LETTER; }
```

For reference on the patterns made available by `Flex`, see [this section](https://ftp.gnu.org/old-gnu/Manuals/flex-2.5.4/html_mono/flex.html#SEC7) in the manual.
## Comments
Comments gotta have spacing or else it will give off an error. E.g.:
```
// Comment
```
Will give off an error like the following:
```
flex -o build/lexer.yy.c src/lexer.lex
src/lexer.lex:5: bad character: /
src/lexer.lex:5: bad character: /
src/lexer.lex:5: unknown error processing section 1
src/lexer.lex:5: unknown error processing section 1
make: *** [Makefile:22: build/lexer.yy.c] Error 1
```

Instead, do this:
```
 // Comment
```

## States
Flex scanner always run in a certain **state**. The state at which it begins is the **INITIAL** state. Other states can be defined. States can be **inclusive** or **exclusive**:
- **Inclusive:** Both rules of this state and the **INITIAL** statGenerating C File Scannerse will be matched/applied.
- **Exclusive**: Onle the rules of this state, not including of the **INITIAL** state will be matched.

To define rules, in the `definition` section of the flex file, include `%s state1` for **inclusive** states and `%x state2` for **exclusive** states:
```
%s inclusive_state
%x exclusive_state
```

To associate a rule with a state, prefix it with `<STATE_NAME>`. E.g.
```
<STATE1>\!           { return T_EXCL_STATE1;  }
<STATE2>\!           { return T_EXCL_STATE2;  }
<STATE3,STATE4>\!    { return T_EXCL_STATE34; }
<INITIAL>\!          { return T_EXCL_INITIAL; }
\?                   { return T_INTR_INITIAL; }
```

Rules without a prefixed state are automatically associated with the **INITIAL** state.

# Functions and Data Structures
## Running the Scanner
The function `yylex()` is the "heart" of the scanner. Calling this function will effectively commence scanning until end of input (EOF) or until a pattern is matched and the scanner runs some code that returns something.

The signature of this function will depend on some factors. If you're writing a stand-alone scanner (not linking with Bison), then the function signature will likely be `int yylex(void)`. If you're integrating with Bison, then it might be something like `int yylex(YYSTYPE* yylval)`, where, in both cases, the `int` is usually some number encoding the token which is usually defined in an enum. The `YYSTYPE` is a data structured used by Bison and will not be discussed in this reference. When EOF is reached, `yylex()` will return 0.

In that manner, a code that uses `yylex()` might be something along those lines:
```C
%{
// Headers, includes and definitions

// We gotta either define a token for EOF with value 0, or start assigning values to enum starting from 1 as done below.
typedef enum {T_AND = 1, T_OR, T_NOT, T_VAR} t_token;

// ...
%}
  /* REGEX DEFINITIONS */
letter        [a-zA-Z]
digit         [0-9]
identifier    {letter}({letter}|{digit})*
%%
	/* Code on REGEX matching */
\&             { return T_AND; } 
\|             { return T_OR; }
\~             { return T_NOT; }
{identifier}   { yylval.varname = new std::string(yytext); return T_VAR; }
[ \t\n]         ;
%%

int main(int argc, char* argv[]) {
	// Stuff
	while ((token = yylex()) {
		switch(token) {
			case T_AND:
				// Something...
				break;
			case T_OR:
				// Something...
				break;
			case T_NOT:
				// Something
				break;
			case T_VAR:
				// Something
				break;	
		}	
		// More stuff...
	}
	return 0;
}
```
## Input/Output Control
`FILE* yyin` and `FILE* yyout` are two global variables defined in the Flex scanner file which point to the input and output stream for the lexer. They allow you to control to which file the scanner will read from, and to which file it will output, aside from allowing you to do other operations you can do over a `FILE*`.

Here is some ChatGPT-generated sample usage:
```C
yyin = fopen("input.txt", "r");
yyout = fopen("output.txt", "w");
if (!yyin || !yyout) {
    perror("fopen");
    exit(EXIT_FAILURE);
}
yylex();
fclose(yyin);
fclose(yyout);
```

You can also use `void yyrestart(FILE* new_file)` to reset Flex's internal state and start scanning over the `new_file`.
## Backtracking and Control
There are two functions that allow `yylex` to "re-scan" after finding a match: `yyless` and `yyunput`.

### yyless
The function `void yyless(int n)`, when called after matching, will "trim" the first `n` characters of the match and have these first `n` characters as the new match for the rest of the code from where it was called and, after returning from the code called upon that match, resume scanning from the beginning of the match + `n`.

To illustrate that, let's use this simple, but great example from [codedost](https://codedost.com/flex/flex-programs/flex-program-check-use-yyless-function/). Suppose we have the following `.lex` file as provided by the article:

```c
%{
/* Implementation of yyless() */
#undef yywrap
#define yywrap() 1
%}

%%

[a-z]+ {
	printf("\nLowercase word = ");
	ECHO;
	yyless(3);
	printf("\nThe word after yyless() = ");
	ECHO;
}

[a-zA-Z]+ {

	printf("\nMixed letter is = ");	
	ECHO;

}

%%

main()
{
yylex();
}
```

We would get the following output given input `codedost`:
```
Lowercase word = codedost
The word after yyless() = cod
Lowercase word = edost
The word after yyless() = edo
Lowercase word = st
The word after yyless() = st
```

Thus, the internal state of the scanner would be something like this:
```
1st Match:
codedostEOF
^^^^^^^^
match

After yyless(3):
codedostEOF
^^^
match

Scanner pointer after returning:
codedostEOF
   ^

2nd Match:
codedostEOF
   ^^^^^
   match

After yyless(3):
codedostEOF
   ^^^
   match

Scanner pointer after returning:
codedostEOF
      ^

3rd Match:
codedostEOF
      ^^
      match

After yyless(3):
codedostEOF
      ^^
      match

Scanner pointer after returning
codedostEOF
        ^
```

## yyunput
The function `void yyunput(int c, char* ptr)` causes the scanner to insert the character c at `ptr`.

For example, suppose we want to read a sequence of digits and print the digits in groups of at most 3 digits, not including any digits for the following sequence.

For example, if given the digits `123456 7890 123 45`, then we want to print:
```
123
456
789
0
123
45
```

Then we can use unput as follows to get this result:
```C
%{
#include <stdio.h>
%}

%%

[0-9]{4,}  {
    int len = strlen(yytext);
    int i;
    for (i = len - 1; i >= 3; i--) {
        yyunput(yytext[i], &yytext[i]);
    }
    yytext[3] = '\0';  // Truncate the string to the first 3 characters
    printf("Matched Rule 1: %s\n", yytext);
    return 1;
}

[0-9]+     {
    printf("Matched Rule 2: %s\n", yytext);
    return 1;
}

%%

int main() {
    yylex();
    return 0;
}
```

The output produced would be as follows:
```
Matched Rule 1: 123
Matched Rule 2: 456
Matched Rule 1: 789
Matched Rule 2: 0
Matched Rule 2: 123
Matched Rule 2: 45
```

Note that, when matching with the 1st rule which will always match when there is a sequence of 4 or more digits, the `for` loop will return to the input stream every digit but the first 3 digits from the match and keep only the remaining digits as part of the match. 
## Global Variables with Useful Metadata
Flex also makes available a few variables which hold some metadata regarding the match and the internal state of the scanner:
- `char* yytext`: A pointer to a null-terminated character array of the match.
- `int yyleng`: An `int` with the size of the match.
- `int yylineno`: An `int` with the number line in which the scanner is currently in. You can either manually manage `yylineno` OR use `%option yylineno` for it to be automatically managed by the scanner (incremented for every new line encountered in the input).

Hint: These are very useful variables for error reporting. Make sure to use them if you want to indicate where an error has happened.
## Error-handling
The function `void yyerror(char* msg)` is the function called by the scanner whenever an error is encountered. It is not defined -- you get to define it.

The scanner by itself won't ever call this function (unless you explicitly code it to do so) since everything that is not matched will simply be ignored. Nonetheless, if you use it with Bison, then Bison will call it whenever it encounters a syntax error.

## Appending to the Match
The function `void yymore(void)` allows one to append the following match to the current match.

For example, you can use it to match a multi-line comment as follows:
```C
%{
#include <stdio.h>
%}

%%

"/*"          { yymore(); }
[^*]*\*+[^*/]*  { yymore(); }  /* Match content within the comment, but not the ending "*/" */
\*/           { printf("Found comment: %s\n", yytext); }

%%

int main() {
    yylex();
    return 0;
}
```

So, for instance, running the scanner generated from this Flex file on the input 
```
/* This is a 
multiline comment */
```

Will cause the following to be printed:
```
Found comment: /* This is a 
multiline comment */
```

(Although, IMO, a state-based solution to handling comments might result in clearer code and logic).

## Terminate
The function `void yyterminate(void)`, as the name suggests,  terminates the scanning. Although obvious-sounding, here are some important claficiations to be made

- It does not stop program execution, only the input scanning.
- It is equivalent to returning 0 on a code run upon matching with a pattern, which is what is returned on EOF.
- It does not alter the internal state of the scanner.
- Scanning can be returned by calling yylex again.
## Text Editors for Flex/Lex files
- **VSCode** - Has plugins available for these files. I don't use it, so I can't recommend any.
- **Vim/nvim** - Provide native support and syntax highlighting for flex files. Good choice.
- **Helix** - This text-editor is relevant to me because I use it, and if you use it like me, you're out of luck because it doesn't support flex/lex files.
- **Notepad** - Why do you do this to yourself?
## Calling Flex Functions Externally
If you want to call Flex functions from another source file (e.g. another file implements the scanner function), then you have two choices:
1. You can generate a Flex header file by calling `flex --header-file=header-file source-file`. This will generate the `header-file`, which you can `include` in a C file. This is not a good choice if you want to include these functions in a C++ files. If you want to do that, then the option below might be a better alternative.
2. Write your own header file with the function signatures of the functions you want to call externally. If you want to call these functions from a C++ file, you need to wrap them inside a `extern "C"` as follows:

```cpp
extern "C"
{
	int yyparse(void);
	int yylex(void);
	int yywrap(void);
}
```

## Compiling
```
flex -o out src
```

The naming conventions of Flex `src` files include file names with a `.l`, `.lex`, or `.flex` extension (I recommend using `.lex` since it is more easily recognised by text editors while `.flex` might not be), and `out` files usually have the extension `.lex.c`.

To compile a Flex-generated scanner, you compile it like you'd compile any other C/C++ file, but you have to make sure to include the `-lfl` flag to link the Flex library.
## Important Functions and Data Structures
**This section needs improvement**. In short, these are some relevant functions that I was too lazy to look in greater detail in the documentation, so I just asked ChatGPT to summarize them for me. For now, look up the documentation for more info.

1. **Basic Functions**:
    - `yylex()`: The primary scanning function generated by Flex. When called, it scans the input for tokens.
2. **Input/Output Control**:
    - `yyin`: A global `FILE*` variable that points to the input stream. By default, it's set to `stdin`, but you can redirect it to read from a file.
    - `yyout`: A global `FILE*` variable for the lexer's output. By default, it's `stdout`.
    - `yyrestart(FILE* new_file)`: This function can be used to prepare Flex to scan a new file.
3. **Buffer Management**:
    - `YY_BUFFER_STATE yy_create_buffer(FILE* file, int size)`: Creates and returns a new buffer state object for the given file. The size specifies the size of the buffer.
    - `void yy_switch_to_buffer(YY_BUFFER_STATE new_buffer)`: Switches the scanner's current buffer to the new buffer.
    - `void yy_delete_buffer(YY_BUFFER_STATE buffer)`: Deletes a buffer created by `yy_create_buffer()`.
    - `void yy_flush_buffer(YY_BUFFER_STATE buffer)`: Flushes the contents of the specified buffer.
4. **Backtracking and Text Control**:
    - `yyless(int n)`: Useful for backtracking. Retains the first `n` characters of `yytext` and returns the rest back to the input stream.
    - `yyunput(char c, char* buf_ptr)`: Puts a single character back into the input stream.
5. **State Management (Start Conditions)**:
    - `BEGIN(state_name)`: Changes the current start condition to the specified state. This is helpful when defining patterns that should only match in specific contexts.
    - `INITIAL`: The default start condition.
6. **Text and Match Data**:
    - `char* yytext`: A pointer to the current matched text.
    - `int yyleng`: The length of the matched text (i.e., the length of `yytext`).
    - `int yylineno`: Keeps track of the number of lines read. If `%option yylineno` is used in the lexer definition, `yylineno` will automatically be incremented for each newline (`\n`) encountered in the input.
7. **Error Handling**:
    - `yyerror(char* msg)`: A function that can be defined by the user. It's called by Bison (or other Yacc-compatible parser generators) when a syntax error is encountered.
8. **Miscellaneous**:
    - `yymore()`: Tells Flex that the next time it matches a rule, the corresponding text should be appended to the current value of `yytext` instead of replacing it.
    - `yyterminate()`: Used to indicate the end of scanning, typically returning a zero value to indicate the end of input.
## Resoures
**Manual**: https://ftp.gnu.org/old-gnu/Manuals/flex-2.5.4/

**Decaf Flex**: https://anoopsarkar.github.io/compilers-class/lex-practice.html (I wrote my first compiler in this course)
