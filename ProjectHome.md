# Introduction #

Lua Checker is a program that analyzes [Lua](http://www.lua.org) source for common programming errors, much as the "lint" program does for C. The following problems are currently identified:
  * Use of variables that have not been declared.
  * Multiple declarations of variables.
  * Attempting to change the value of constant variables.
  * Other things are planned, see the 'future' section below.

Lua Checker was developed within the Street View team at Google to validate Lua scripts. [Russ Smith](http://www.q12.org) is the primary author.
Lua Checker is licensed under the standard Lua 5.0 license (an MIT-style license).

Lua Checker contains a bison compatible LALR(1) parser for the [Lua](http://www.lua.org) language (see `lua.y`) which may be useful to Lua developers in its own right.

## Background ##

Languages such as C/C++ and Java are strongly typed. This means that each variable has only one type (such as a number, string or object), and the compiler will give an error when the variable is used in an incorrect way. This means that many problems are caught early.

In contrast, the script language [Lua](http://www.lua.org) is dynamically typed, with a simple type model.
Variables can be assigned values of any type.
This makes development of small scripts somewhat easier, but it means that in larger programs many common programming errors are not detected until run time. For example:

  * Referencing a variable that has not been declared will return `nil`. This is no different from referencing a 'declared' variable that has the value `nil`. Thus spelling mistakes in Lua variable names can go undetected and lead to program misbehavior.
  * Tables (Lua's main data structure) are not typed, so any table can contain any keys. When tables are used in a manner similar to C structures, spelling mistakes in field names can go undetected and lead to program misbehavior.
  * Function arguments and return values are also untyped and have similar problems.

In practice, Lua programs over (say) 1000 lines tend to accumulate these kinds of problems, making debugging difficult.
A standard Lua practice to deal with undefined global variables is to install a special 'get-value' handler on the global variable table that warns when undefined globals are accessed. This is of limited benefit because problems are still only detected at run-time.

Lua Checker is designed to solve these problems. It performs a static analysis of Lua source code prior to it being run, and prints out warnings about any problems that it finds.

## Usage ##

To help Lua Checker to do its job, Lua source code must be written in a slightly more restricted way.
  * All global variables must be declared in the global scope before they are used. For example do the following:
```
  foo = 123             -- Foo declared
  print(foo)            -- Foo used
```
> but don't do this, even though it is functionally the same:
```
  function FooSetter()
    foo = 123           -- Lua Checker doesn't know that foo is allowed to be a global
  end
  FooSetter()
  print(foo)
```
  * Global variables access via the `_G` table will bypass the checking mechanism, so don't do this unless you really have to.
  * Other source files included via `dofile` will also be scanned, as long as dofile is called at the outer scope with a single string argument:
```
  dofile 'foo.lua'      -- foo.lua will also be scanned
  dofile('bar.lua')     -- bar.lua will be ignored
  if true then
    dofile 'baz.lua'    -- Inner scope: baz.lua will be ignored
  end
```
  * Variables can be declared as constant with the following addition to Lua syntax:
```
  foo = 123             --@ const
```
> the `--@` marker indicates that special Lua Checker keywords will follow on the same line.
> The `const` keyword means that the previous variable declared is to be a constant.
> Any further attempt to set that variable will be an error.
> Note that special keywords can be followed by regular Lua comments as follows:
```
  foo = 123             --@ const  -- A comment
```

### Command Line Arguments ###

**The `CHECK_LUA.sh` script is currently used to invoke lua\_checker. It is clunky and will be replaced with something better.**

The `lua_checker` program is invoked as:
```
  lua_checker ...flags... filename.lua
```
The available flags are
  * -no\_reuse\_varnames: Variables names can't be reused in inner scopes. This is more strict than most strongly typed languages, but it may catch more errors.
  * -const\_functions: All function variables are constant, so for example the following is an error:
```
  function Foo() ... end     -- Foo is now a constant function variable
  Foo = 123                  -- Error, attempt to assign to a constant
```

## Implementation ##

The interesting parts of Lua Checker are implemented within bison parsers.
Two separate parsers are used, that are run by two separate programs:
  1. `lua_simplifier` takes the original Lua source code and rewrites it into a simpler form. The simplified source has most syntactic sugar expanded out and has fewer syntactic ambiguities (mostly achieved by putting semicolon at the end of every line). See the comments at the start of `lua.y` for a detailed description. The point of simplification is to allow the actual Lua checking code to live within a much simpler parser that does not have to also deal with complex Lua syntax issues.
  1. `lua_checker` takes the simplified Lua source and actually performs the checks.

## Future ##

Features that are in the works:
  * Allow tables to be assigned a type, similar to C structures. Warn when unexpected tables fields are accessed.
  * Allow functions to be assigned argument and return value types. Warn when argument or return value types mismatch.
  * Detect variables that are never assigned to (useless variables).
  * Detect variables that unexpectedly change type (e.g. from number to table).
  * Warn about defining new global functions in local scope.
  * Warn about using ellipses in functions that don't have an ellipses argument.
  * Allow user-defined Lua extensions (e.g. global functions defined by the embedded Lua environment).
  * Also check modules pulled in with 'require'.