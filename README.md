# MicroJava Compiler

A compiler for the MicroJava programming language, made as a project for the course Compiler Construction 1, School of Electrical Engineering, University of Belgrade (year 2025).

The compiler translates syntactically and semantically correct MicroJava programs into MicroJava bytecode, which is executed on the MicroJava virtual machine (MJVM).

The compiler has four basic parts: lexical analysis, syntax analysis, semantic analysis and code generation.

---

## Compiler phases

### 1. Lexical analysis

The lexer reads the source code character by character and groups the characters into tokens: identifiers, numbers, char and bool constants, keywords (`program`, `print`, `new`, ...) and operators. Spaces, tabs, line breaks and `//` comments are skipped, because the rest of the compiler doesn't need them.

If the lexer meets a character that doesn't belong to the language, it prints the character and the line number to standard error (the console) and continues, so one bad character doesn't stop the whole compilation.

The token rules are defined in `spec/mjlexer.lex`. JFlex generates the lexer class `Yylex.java` from that file.

### 2. Syntax analysis

The parser takes the tokens from the lexer and checks that they are in the right order according to the grammar of the language. For example, it checks that a declaration looks like `type name;` and that a statement ends with `;`. While doing that, it builds the syntax tree (AST) of the program, which the next phases work on.

If the program breaks a grammar rule, the parser logs an error with the line number. It can also recover from some errors and continue parsing, so more than one error can be found in a single run. Here, recovery is done for assignment statements: after an error the parser skips tokens until the next `;`.

The grammar is in `spec/mjparser.cup`. The parser and the syntax tree classes (`rs.ac.bg.etf.pp1.ast`) are generated with AST-CUP.

### 3. Semantic analysis

The grammar can't check everything. For example, `x = 5;` is correct syntax, but it only makes sense if `x` was declared and is an `int`. The semantic analyzer (`SemAnalyzer`) walks through the syntax tree and checks this kind of thing, using the symbol table, where all declared names are stored with their types.

It checks, among other things, that:
- every name is declared before it is used, and not declared twice in the same scope
- types match in assignments and constant declarations
- `int` is used for arithmetic, `++`/`--` and array indexes
- `read` and `print` get valid types, and `union` gets sets
- method calls have the right number and types of arguments
- the program has a `main` method

Every use of a constant, global variable or local variable is logged (line, name and symbol table info), and the symbol table is printed at the end.

### 4. Code generation

If the program has no syntax or semantic errors, the code generator (`CodeGenerator`) walks through the syntax tree once more and writes MJVM instructions for each node: loading values, arithmetic, storing to variables, array access, `read`, `print` and so on. The result is an executable `.obj` file, which can be disassembled and run on the MicroJava virtual machine.

---

## What compiler supports

The grammar below is the subset of MicroJava that the compiler handles (the full grammar is in `spec/mjparser.cup`).

Program structure:

```
Program      = "program" ident { ConstDecl | VarDecl } "{" { MethodDecl } "}"
ConstDecl    = "const" Type ident "=" (numConst | charConst | boolConst)
               { "," ident "=" (numConst | charConst | boolConst) } ";"
VarDecl      = Type ident [ "[" "]" ] { "," ident [ "[" "]" ] } ";"
MethodDecl   = ( Type | "void" ) ident "(" ")" { VarDecl } "{" { Statement } "}"
Type         = ident
```

Statements:

```
Statement           = DesignatorStatement ";"
                    | "read" "(" Designator ")" ";"
                    | "print" "(" Expr [ "," numConst ] ")" ";"
                    | "return" [ Expr ] ";"
DesignatorStatement = Designator "=" Expr
                    | Designator "=" Designator Setop Designator
                    | Designator "++"
                    | Designator "--"
                    | Designator "(" [ ActPars ] ")"
ActPars             = Expr { "," Expr }
```

Expressions:

```
Expr       = [ "-" ] Term { Addop Term }
Term       = Factor { Mulop Factor }
Factor     = numConst | charConst | boolConst
           | "(" Expr ")"
           | Designator
           | "new" Type "[" Expr "]"
Designator = ident [ "[" Expr "]" ]
Addop      = "+" | "-"
Mulop      = "*" | "/" | "%"
Setop      = "union"
```

Also:
- `void main()` with no arguments, global and local variables (single values and arrays), global constants
- arrays of `int`, `char` and `bool`
- sets of integers: `new set[n]`, `union`, printing
- built-in methods: `chr`, `ord`, `len`, `add`, `addAll`

## Tools

- JDK 1.8 (Java 8)
- Apache Ant
- JFlex
- AST-CUP
- Symbol table library
- MicroJava runtime
- log4j

**Tool usage:**

- **Generated code:** The lexer and the parser are not written by hand, they are generated from description files. JFlex builds the lexer from the token rules (`spec/mjlexer.lex`), and AST-CUP builds the parser and the syntax tree classes from the grammar (`spec/mjparser.cup`).
- **Hand-written code:** The semantic analysis and the code generation are written by hand. The semantic analysis keeps the declared names and their types in the symbol table library. The code generation uses the MicroJava runtime to write the bytecode.
- **Helper tools:** Ant runs the build, and log4j is used to log the output of the compiler.

---

## Notes

- A set is an int array with one extra element at the end that holds the number of elements. `new set[n]` creates an array of `n + 1` elements.
- `add`, `addAll`, `len`, `chr`, `ord`, set printing and `union` are written as bytecode in the `CodeGenerator` constructor. They are called like normal methods.
- `union` adds all elements of both sets into the result set.
- `char` arrays are byte arrays. `int`, `bool` and `set` use words.
- Output (syntax tree, symbol usage, errors, symbol table) goes through log4j.
- The `test/` folder has a test program, its input and the generated `.obj` file.

---
