# Assignment 4: Typechecking and Code Generation for the JPL Subset

Your third assignment is to build a compiler for the subset of JPL
that you implemented a parser for in Assignment 2. This compiler's job
is to turn an AST into:

- an assembly function, or
- a naming or type error.

You will be implementing this compiler in three steps:

- Name resolution and type checking
- Flattening
- Code generation

**IMPORTANT:** Ideally, your compiler will typecheck the AST,
flatten it, and then typecheck the (now flattened) AST a second time
in order to catch flattener bugs. However, depending on how you have
structured your compiler, it may not be convenient to typecheck
twice. Therefore we are permitting a different option where your
compiler supports two different modes. First, when your compiler gets
the `-t` option, it typechecks the AST and prints it in s-expression
form as describe below. Second, when it gets the `-f` option, it
flattens the AST (without typechecking first) and then typechecks it
before printing s-expressions. If you choose this second option, you
will still need to print s-expressions containing type information
before and after flattening, but no single execution of your compiler
will need to run the typechecker twice.

Recall that the subset of JPL that we are working with contains values
of just four types: `bool`, `int`, `float`, and
`{float,float,float,float}[,]`. In this assignment we will refer to
the last of these as `pict`. These correspond to the following C types
(you'll need `<stdint.h>`):

- `bool` corresponds to `int32_t` and takes up 4 bytes
- `int` corresponds to `int64_t` and takes up 8 bytes
- `float` corresponds to `double` and takes up 8 bytes
- `pict` corresponds to `struct { int64_t; int64_t; double *; }`
  and takes up 24 bytes (since all three components of the struct
  have 8-byte alignment, there is no padding)

Expressions in this subset are either constants, or they are a call to
one of the following functions:

- `sub_ints(int, int) : int`
- `sub_floats(float, float) : float`
- `has_size(pict, int, int) : bool`
- `sepia(pict) : pict`
- `blur(pict, float) : pict`
- `resize(pict, int, int) : pict`
- `crop(pict, int, int, int, int) : pict`

A runtime with implementations (in C) of all of these functions will
be provided to you, with the following signatures:

```
#include <stdint.h>

struct pict {
    int rows;
    int cols;
    double *data;
};

int64_t sub_ints(int64_t a, int64_t b);
double sub_floats(double a, double b);
int32_t has_size(struct pict input, int64_t rows, int64_t cols);
struct pict sepia(struct pict input);
struct pict blur(struct pict input, double radius);
struct pict resize(struct pict input, int64_t rows, int64_t cols);
struct pict crop(struct pict input, int64_t top, int64_t left, int64_t bottom, int64_t right);
```

Your compiler's output will call these provided implementations.

Your compiler must output code for the [NASM assembler][nasm]. That
assembly code must compile cleanly to an object file that defines the
function `main`; after compilation, the user will run NASM to assemble
the generated code, and link it with the provided runtime to produce a
finished executable.

[nasm]: https://www.nasm.us/

Besides the functions provided above, the runtime also provides these
functions:

```
struct pict read_image(char *filename);
void print(char *text);
void write_image(struct pict input, char *filename);
void show(char *typestring, void *datum);
void fail_assertion(char *text);
double get_time(void);
```

You are expected to use them to implement JPL's various commands and
statements. Note that to use `show` you need the type, as a string in
JPL syntax, and a *pointer to* the object you want to print. (In other
words, you must call `show("...", &data)` for it to work.) The
`print` and `fail_assertion` commands expect that their
string argument ends in a newline, which you need to add when
generating assembly code.

## Type Checking in JPL subset

With only four types, type checking JPL is pretty simple. You will
need to define a class for types, containing the four types above plus
function types; a function type stores a list of types for the
function's arguments, plus a return type. This will allow you to
construct a *symbol table*, which maps variable and function names (as
strings) to types.

Next, you must define two functions:

- `type_expr` takes an `Expr` AST node and a symbol table, and either
  terminates the program (on a type error) or returns the type of the
  expression. Type errors only occur if the arguments to a function
  have the incorrect type, or if an undefined variable or function is
  used, or if a normal variable is used as a function.

- `type_command` takes a `Command` AST node and a symbol table, and
  either terminates the program (on a type error) or returns nothing
  after modifying the symbol table. Type errors occur for `assert`
  with a non-`bool` argument, `write image` with a non-`pict`
  argument, and `return` with a non-`int` argument. They also occur
  for `let` and `read image` when they try to redefine a variable
  name that is already bound.

Note that `show` works on any type so it never raises a type error.

Modify your `Expr` AST node to add a field for types. Modify
`type_expr` to save the type it returns to the AST node it was type
checking.

Your compiler should implement a `-t` command line option which causes
it to stop after typechecking and print s-expressions (similar to
those you printed for the handin part of Assignment 2) that include
type information.

For example, consider this JPL program (which is the same as `007.jpl`
in the `tests` subdirectory under the directory where this `README.md`
file is stored:

```
let z = sub_ints(7, 2)
return z
```

Your existing compiler from before starting this assignment, when
given the `-p` flag, should be producing something like this:

```
(StmtCmd (LetStmt (ArgLValue (VarArgument z)) (CallExpr sub_ints (IntExpr 7) (IntExpr 2))))
(StmtCmd (ReturnStmt (VarExpr z)))

Compilation succeeded
```

Then, after your output is run through `pp.rkt` (which strips off
the final line) you should end up with this exact text:

```
(StmtCmd
 (LetStmt
  (ArgLValue (VarArgument z))
  (CallExpr sub_ints (IntExpr 7) (IntExpr 2))))
(StmtCmd (ReturnStmt (VarExpr z)))
#<eof>
```

When given the `-t` flag, your compiler for this assignment should
produce output that, when run through `pp.rkt`, looks exactly like
this:

```
(StmtCmd
 (LetStmt
  (ArgLValue (VarArgument int z))
  (CallExpr int sub_ints (IntExpr int 7) (IntExpr int 2))))
(StmtCmd (ReturnStmt (VarExpr int z)))
#<eof>
```

The difference is that all now all AST nodes that have types include
the name of those types as annotations in the output. You can find
more examples in `tests/*.jpl.typechecked`.


## Flattening the JPL subset

Flattening refers to replacing deeply-nested ASTs with shallow ones by
adding more `let` statements. This is an extremely common technique
that is implemented in almost all compilers. Its purpose is to take
complex syntactical forms that appear in the source language, and to
turn them into simpler (but functionally equivalent) forms that are
easier for the rest of the compiler to process. For example,
flattening the following expression:

    read image "in.png" to img
    time write image resize(crop(sepia(img), 50, 250, 650, 650), 300, 200) to "out.png"

would produce:

    read image "in.png" to img
    let t.1 = get_time()
    let t.2 = sepia(img)
    let t.3 = 50
    let t.4 = 250
    let t.5 = 650
    let t.6 = 650
    let t.7 = crop(t.2, t.3, t.4, t.5, t.6)
    let t.8 = 300
    let t.9 = 200
    let t.10 = resize(t.7, t.8, t.9)
    write image t.10 to "out.png"
    let t.11 = get_time()
    let t.12 = sub_floats(t.11, t.1)
    print "time:"
    show t.12
    let t.13 = 0
    return t.13

Specifically, flattening a type-correct JPL program from this
assignment's subset must result in a type-correct JPL program, also in
the same subset, which satisfies these additional constraints:

- Every argument to a function is a variable reference
- Every argument to `assert`, `return`, `show`, or `write image` is a
  variable reference.

- There are no `time` commands; those are expanded to two `get_time`
  calls (one before the code whose execution is being timed and one
  after), `sub_floats`, `print`, and `show`

- There is exactly one `return` command, and it is the last command in
  the list. If the input program had more than one `return`, drop all
  commands that come after it; if it did not have a `return`, add `let
  t.N = 0; return t.N` to the end of the program.

Note that this requires introducing new variable names. You should
name your new variables `t.N`, where `N` is the value of a global
counter that counts up from 0. This guarantees that these variables
will not clash either with each other or with variable names chosen by
the user. It also allows us to match your flattened output against
ours (since those are the variable names that our compilers use too).

Make sure that the flattened JPL you generate is type-correct. We
recommend re-type-checking the flattened output and crashing your
compiler if it does not type check. This will catch a lot of bugs.

Your flattening function should take a list of commands and a symbol
table as input. As output, it produces a list of commands and updates
the symbol table to reflect the new variables that it
introduced. Flattening is not allowed to fail: once a JPL program in
this assignment's subset has been typechecked, flattening should
always work.

Note that `time return` is slightly awkward because the `time` command
is flattened to multiple commands, some of which come before the
`return` command and some of which come after. If your compiler runs
into this code, all parts of the `time` command that come before the
`return` should appear in the final flattened output, but none of the
commands after the `return` should appear.

Your compiler must implement a `-f` command line option which causes
it to stop after flattening and print s-expressions. Again, these
should be annotated with types. Here is the flattened, typechecked
version of `007.jpl` after being run through `pp.rkt`:

```
(StmtCmd (LetStmt (ArgLValue (VarArgument int t.0)) (IntExpr int 7)))
(StmtCmd (LetStmt (ArgLValue (VarArgument int t.1)) (IntExpr int 2)))
(StmtCmd
 (LetStmt
  (ArgLValue (VarArgument int z))
  (CallExpr int sub_ints (VarExpr int t.0) (VarExpr int t.1))))
(StmtCmd (ReturnStmt (VarExpr int z)))
#<eof>
```

The expected flattened output for every test case can be found
in `tests/*.jpl.flattened`.

**IMPORTANT:** Since there are many different ways in which you might
choose to traverse the AST during the flattening process, we are not
going to insist that you produce exactly the output above. Your output
may differ in the order in which flattened statements appear, and it
may differ in choices of temporary variables. In other words, your
output must meet the requirements for flattened code that are listed
in this part of the assignment, but it does not need to be
syntactically identical to our output.


**For a small amount of extra credit:** Instead of printing `time:`
before printing an elapsed time, print `time: cmd` where `cmd` is the
text of the command being timed, as it appeared literally in the
original source code, including any comments or newlines that appeared
inside that command. When doing this, you'll run into a small problem
because in JPL the string argument to `print` cannot contain double
quotes. You should replace double quotes with single quotes to
circumvent this. Printing this output will cause your compiler to fail
some test cases. This is fine as long as the only failures are the
ones triggered by this extra string.


## Planning the Stack Frame

The first stage of code generation is planning the stack frame.

Compute the total size of all variables in the symbol table; treat
functions in the symbol table as taking up 0 bytes. This total is the
size of the stack frame; store this in a variable.

Create a data structure to map variable names (as strings) to
locations in the stack frame (as integers). Iterate through your
symbol table and assign a location to each variable, in order, making
sure to give each variable as much space as its type requires. For
example, the the flattened code above defines:

| Var      | img  | t.0    | t.1  | t.2 | t.3 | ... |
|----------|------|--------|------|-----|-----|-----|
| Type     | pict | double | pict | int | int | ... |
| Size     | 24   | 8      | 24   | 8   | 8   | ... |
| Location | 24   | 32     | 56   | 64  | 72  | ... |

The total size of the stack frame in this case is 176 bytes. Note that
locations do not start at 0. This is because the stack grows down.

The stack frame must be a multiple of 16 bytes in size. In this
example, it already is, but if it weren't, you need to pad the stack
frame to be a multiple of 16 bytes.

## Storing Constants

Let's begin generating assembly code. In the [Assembly
Handbook](../assembly.md), go to the "Syntax of Assembly" section and
output the various parts required for your assembly code.

Next you must gather all the integer, float, and string constants in
your program. These are placed in the "data section" of the assembly,
and later on when the program actually uses those constants they're
loaded from the data section.

Each constant must be given a name; store that name somewhere so you
can recall it later, when the constant is used. One way to do that is
to have a hash table that maps AST nodes to names.

Then output a constant definition for each constant, as described in
our [Assembly Handbook](../assembly.md).

## Generating code

In the "text section" of your assembly code, you must generate the
code for the `main` function. That means generating the function
prologue, then outputting code for each commands in the program in
order, and then generating the function epilogue. The [Assembly
Handbook](../assembly.md) has templates for the assembly code for each
of these steps.

Luckily, after flattening, the JPL program is a shallow list of
commands, with one of the following forms:

- A `let` statement with an integer or floating point constant
- A `let` statement with a variable reference
- A `let` statement with a function call where all arguments are variables
- An `assert` statement with a variable reference
- A `read image` command
- A `write image` command with a variable reference
- A `print` command
- A `show` command with a variable reference
- A `return` statement with a variable reference

Each of the types of commands has its own assembly snippet that must
be produced, described in the [Assembly Handbook](../assembly.md) in
the root of this directory.

## Testing correctness

If you follow the instructions in our [Assembly
Handbook](../assembly.md) carefully, your code should work. What that
means is a little involved. Your compiler must ensure that:

- The compiler, run with the `-s` flag, returns successfully and
  outputs a bunch of assembly code and a single line with the words
  `Compilation successful` at the beginning.
- The NASM assembler then successfully assembles the code into an
  object file and produces no console output.
- The linker then successfully links the object file and the runtime
  into an executable and produces no console output.
- The resulting executable runs and produces the expected output and
  return code, without triggering any kind of fault condition (like a
  segmentation fault). It also does not trigger any unexpected effects
  (accessing the wrong files, for example).

Because there are lots of ways to go wrong, you'll be doing a lot of
debugging, at several different levels. To start, try to assemble,
link, and run the provided [hello.s](hello.s) file, which should print
"Hello, World!" with a successful return code and no fault conditions
when properly assembled, linked, and run.

## Debugging Assembly

*This will be a challenging assignment.* One of the biggest challenges
during code generation is debugging the emitted assembly language. We
offer the following suggestions:

**Time**: Plan on spending a lot of time debugging. Start very early
on this assignment.

**Incrementalize**: Practice incremental development to the maximum
extent possible. Get something small and simple to work, and test it
thoroughly, before moving on. The code generation tests are ordered by
difficulty for this reason; it is a good idea to start at `001.jpl`
and not to move on until it works.

**Bisect**: Always start by narrowing the issue to the relevant part
of the compiler. Check the type-checked tree, the flattened output,
and the assembly code, so you know which stage went wrong. Keep stages
strictly separated (keep all assembly logic separate from all
flattening logic) so it's easy to debug. You can add extra compiler
flags to output additional information (like the stack layout).

**Common Issues**: The [Assembly Handbook](../assembly.md) lists some
common issues and how to fix them. Often the best way to understand a
problem is to narrow it down to the relevant assembly snippet (easy if
you are working incrementally!) and then compare it to the recommended
output.

**Tools**: Use a tool for interactive debugging of assembly language
programs. The important features are inspecting the machine state
(memory, registers, and processor flags) and executing a single
instruction at a time. The default tool for this job is the
interactive debugger `gdb`. The [Assembly Handbook](../assembly.md)
has links to some debugger tutorials.

**References**: Keep a couple of references for both x86-64
instruction set and the NASM assembler specifically handy while you
are working. You can find lists of both in the [Assembly
Handbook](../assembly.md).

## CHECKIN Due March 5

The checkin part of this assignment is intended to assist you in
eliminating as many bugs as possible in your name resolution, type
checking, and flattening code before you move on to code generation.

For this part of the assignment you should implement typechecking and
flattening as described above. Also, add two new targets to your
makefile called `run-a4t` and `run-a4f`. If we go to your compiler's
root directory and type `make run-a4t TEST=007.jpl` then your makefile
must run your compiler with the `-t` command line option on `007.jpl`.
If we go to your compiler's root directory and type `make run-a4f
TEST=007.jpl` then your makefile must run your compiler with the `-f`
command line option on `007.jpl`.

To test your typechecker, go to your compiler's root directory and run
`BLA/jpl/assignment4/test-subset-typechecker` where `BLA` is wherever
you checked out the `jpl` repository. This script will invoke your
makefile using commands such as `make run-a4t TEST=007.jpl` and
compare your output against the reference output.

As mentioned above, we are not going to perform automated testing of
your flattened code because there are many subtleties in how you
traverse the AST that change the order of statements and temporary
variables. However, we ask that you check (by hand) that your
flattened output is equivalent to ours, by looking at the
`tests/*.jpl.flattened` files. We will also look at your output
by hand.


## HANDIN Due March 19

The handin expects you to have a complete compiler for the JPL subset
of interest, which emits NASM assembly code. Specifically your
compiler should implement a `-s` command line option which causes it
to generate assembly code and print it to the standard output. That
assembly code must be correct as described above---it must pass NASM,
the linker, and run cleanly, producing the expected output.

Also, make sure that your makefile supports `make run TEST=foo.jpl`.
In this case, the makefile should supply the `-s` flag.

To help you in this task, we are providing a number of test inputs as
well as suggested assembly output and also what should be output when
the generated code runs. These can all be found in the `tests`
subdirectory. The tests are approximately ordered by difficulty, so we
suggest going through them in numerical order, getting each one to
work before moving on to the next one. Comparing your generated
assembly to the recommended assembly is likely the fastest way for you
to find issues in your compiler, but you are not required to match it
exactly (since this could be difficult if your flattened code looks
different from ours). We strongly recommend trying to get the assembly
right *before* running it: running malformed assembly is dangerous in
a way looking at it isn't, and comparing diffs of the assembly code is
way easier than using a debugger to step through a bad executable.

Please make sure your compiler works on CADE machines. As usual, this
is a hard requirement, so test it out early so you have time to fix
issues that may come up. On CADE, you will want to use the `elf64`
format for NASM.
