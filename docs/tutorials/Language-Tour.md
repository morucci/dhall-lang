# Language Tour

Walk through all language features

```eval_rst
.. contents:: Table of Contents
   :backlinks: none
```

This tutorial covers all of the Dhall configuration language features, in a
way that is not specific to an integration (e.g. JSON / YAML) or language binding
(e.g. Go / Rust).  In other words, this tutorial assumes that you've already
figured out use Dhall within a project and you want to dive more comprehensively
into Dhall.

## Prerequisites

This tutorial assumes that you have already completed the JSON/YAML tutorial:

* [Getting started: Generate JSON or YAML](./Getting-started_Generate-JSON-or-YAML.md)

This tutorial will use the command-line `dhall` tool to illustrate language
features independent of any specific integration.  You can install a prebuilt
executable for Windows / OS X / Linux from the following release page:

* [`dhall-lang/dhall-haskell` - Releases](https://github.com/dhall-lang/dhall-haskell/releases)

> **Exercise**: Verify that you have correctly installed the tool by running:
>
> ```bash
> $ dhall <<< '2 + 2'
> ```
> ... and verifying that you get:

> ```dhall
> 4
> ```

## Introduction

All Dhall integrations support some way to load "plain" data into the desired
file format or language binding.  These "plain" values include:

* `Bool` values
* `Double` values
* `Natural` values
* `Integer` values
* `Text` values
* `Optional` values
* `List`s
* Records

... and possibly also unions, depending on the integration.

Some languages can even load Dhall functions, meaning that they are dynamically
translated to functions in the host language.  However, this tutorial will not
cover that.

Here is an example of a "plain" expression that can likely be loaded into any
such integration:

```dhall
[ { name = "Alice"
  , age = 24
  , admin = True
  }
, { name = "Bob"
  , age = 49
  , admin = True
  }
]
```

For example, the above Dhall expression corresponds to the following JSON
expression:

```json
[
  {
    "name": "Alice",
    "age": 24,
    "admin": true
  },
  {
    "name": "Bob",
    "age": 49,
    "admin": true
  }
]
```

> **Exercise:** Save the above Dhall expression to a file named `plain.dhall`
> and interpret the file by running:
>
> ```bash
> $ dhall --file plain.dhall
> ```
>
> The only change you should see is that the interpreter sorts the record
> fields.

All integrations can go a step further and interpret the Dhall expression before
converting the expression into the desired format or language.  For example,
the following expression is equivalent to our original example (albeit more
indirect, just to illustrate a point):

```dhall
let List/filter = https://prelude.dhall-lang.org/List/filter

let Person = { name : Text, age : Natural, admin : Bool }

let alice : Person =
      { name = "Alice"
      , age = 24
      , admin = True
      }

let bob : Person =
      { name = "Bob"
      , age = 49
      , admin = True
      }

let carlo : Person =
      { name = "Carlo"
      , age = 20
      , admin = False
      }

let isAdmin = \(person : Person) -> person.admin

in  List/filter Person isAdmin [ alice, bob, carlo ]
```

This is because every Dhall integration includes a built-in interpreter capable
of evaluating Dhall expressions, reducing them to plain data before further
conversion.

> **Exercise**: Save the above Dhall expression to a file named `example.dhall`
> and run:
>
> ```bash
> $ dhall --file example.dhall
> ```
>
> ... and verify that you get the same result as interpreting `plain.dhall`.

A Dhall interpreter processes expressions in six phases:

* Desugaring

  Some higher-level language features are "syntactic sugar" for lower-level
  language-features.  "Desugaring" is the process of translating higher-level
  features to lower-level features.

  Example: `{ x.y = 1 }` desugars to `{ x = { y = 1 } }`

* Import resolution

  This phase replaces URLs, file paths, and environment variables with the
  expressions that they refer to.

  Example: `https://prelude.dhall-lang.org/v15.0.0/Bool/not False` resolves to
  `(\(b : Bool) -> b == False) False`

* Type checking

  This phase ensures that the code is safe to evaluate by detecting and
  forbidding expressions that might lead to crashes, loops, or internal errors.

  Example: `Natural/even False` will fail to type-check

* Normalization (a.k.a. "Evaluation")

  This phase eliminates all indirection in the expression by evaluating all
  remaining programming language features that were not already covered by one
  of the preceding phases.  The result is an expression in a canonical "normal
  form".

  Example: `\(x : Natural) -> x + 0` will normalize to `\(x : Natural) -> x`

* Marshalling

  The Dhall expression is converted a file in the desired format or an
  expression within the desired language.

  Example (converting to Bash):

  ```bash
  $ dhall-to-bash --declare FOO <<< '{ x = 1, y = 2 }'
  declare -r -A FOO=([x]=1 [y]=2)
  ```

Integrations will typically perform all of these steps in one go, but you can
use the `dhall` command-line tool to separate out some of these steps.

> **Exercise:** Using the same `./example.dhall` file as before, run the
> following commands:
>
> ```bash
> $ # `dhall resolve` performs only import resolution
> $ dhall resolve --file ./example.dhall | tee ./resolved.dhall
> ```
>
> ```bash
> $ # `dhall type` type-checks the expression and displays the inferred type
> $ dhall type --file ./resolved.dhall
> ```
>
> ```bash
> $ # `dhall normalize` normalizes the resolved expression
> $ dhall normalize --file ./resolved.dhall | tee ./normalized.dhall
> [ { admin = True, age = 24, name = "Alice" }
> , { admin = True, age = 49, name = "Bob" }
> ]
> ```
>
> ```bash
> $ # `dhall-to-json` marshals the normalized expression into JSON
> $ dhall-to-json --file ./normalized.dhall 
> [
>   {
>     "admin": true,
>     "age": 24,
>     "name": "Alice"
>   },
>   {
>     "admin": true,
>     "age": 49,
>     "name": "Bob"
>   }
> ]
> ```
> 
> ... and study how the Dhall expression evolves between phases.

Note that some of these commands can actually perform all of the preceding
steps.  For example, `dhall-to-json` could have taken the original
`./example.dhall` as input and produced the same result.  However, splitting
things into explicit phases can sometimes help better understand how the
interpreter processes the code.

> **Exercise:** There are two URL imports in the following Dhall expression:
>
> ```dhall
> if True
> then https://prelude.dhall-lang.org/Bool/not True
> else https://prelude.dhall-lang.org/Natural/sum [ 2, 3, 5 ]
> ```
>
> Which imports are resolved by the interpreter?
>
> Does the expression type-check?

## `Bool` values

The `Bool` type is one of the simplest types that the language provides
built-in support for.

The only two valid `Bool` constants are `False` and `True` and the language
provides the following logical operators which work on `Bool`s:

* `&&` - logical "and"

  Example: `True && False` evaluates to `False`

* `||` - logical "or"

  Example: `True || False` evaluates to `True`

* `==` - equality

  Example: `True == False` evaluates to `False`

* `!=` - inequality

  Example: `True != False` evaluates to `True`

Carefully note that the `==` and `!=` operators only work on values of type
`Bool`.  This is one important way that Dhall differs from many other languages.
For example, you cannot compare `Text` or `Natural` numbers for equality or
inequality using these operators.

> **Exercise:** Try to compare two numbers for equality and see what happens:
>
> ```bash
> $ dhall <<< '1 == 1'
> ```
>
> Which interpreter phase do you think rejected the expression?

Additionally the language provides built-in support for `if` expressions.

> **Exercise:** What do you think is the result of interpreting the following
> expression:
>
> ```dhall
> "${if True then "Hello" else "Goodbye"}, world!"
> ```
>
> Test your guess!

## Numbers

`Natural` numbers are non-negative integers.  In other words:

```
0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, …
```

All `Natural` number literals are unsigned.  You can also use hexadecimal
notation if you prefer:

```
0x0, 0x1, 0x2, 0x3, 0x4, 0x5, 0x6, 0x7, 0x8, 0x9, 0xA, 0xB, 0xC, 0xD, 0xE, …
```

`Natural` numbers are the "fundamental" numeric type for the language, since
many useful functions will prefer to use `Natural` numbers in their types to
forbid negative inputs or output.

For example, the type of the built-in `List/length` function guarantees that the
function can never return a negative length:

```dhall
List/length : forall (a : Type) -> List a -> Natural
```

The `Natural` number type is also a good default choice for many configuration
options where negative values are not sensible, like:

* A person's age
* The number of CPUs to provision for a machine
* The maximum number of permitted retries for a failing service

`Integer`s are a different numeric type which is not the same as `Natural`
numbers and all `Integer` literals require an explicit sign:

```
…, -7, -6, -5, -4, -3, -2, -1, +0, +1, +2, +3, +4, +5, +6, +7, …
```

`Integer`s also permit hexadecimal notation, like `Natural` numbers.

Both `Natural` numbers and `Integer`s are unlimited precision, which means that
there is no maximum value for either type.

`Double`s represent IEEE 754 double-precision floating point numbers, such as:

```
-1.0, 3.14159265359, 6.0221409e+23, 1e6
```

> **Exercise:** What do you think will happen if you input a `Double` literal
> that is out of the valid range for a double-precision floating point number?
>
> ```bash
> $ dhall <<< '1e10000'
> ```
>
> Run the above command to find out!

`Double` literals require an explicit decimal point or an exponent (if using
scientific notation).  The `Double` type is also a distinct numeric type from
`Natural` numbers and `Integer`s.

The language provides the following arithmetic operators which only work on
`Natural` numbers:

* `+` - addition

  Example: `2 + 3` evaluates to `5`

* `*` - multiplication

  Example: `2 * 3` evaluates to `6`

These operators do not work on `Integer`s or `Double` values, although you can
convert between `Natural` numbers and `Integer`s using two built-in functions:

* `Natural/toInteger : Natural -> Integer` - Convert a `Natural` number to an
  `Integer`

* `Integer/clamp : Integer -> Natural` - Convert an `Integer` to a `Natural`
  number, clamping all negative `Integer`s to `0`

In general, the language design encourages the use of `Natural` numbers as much
as possible.

## `Text`

`Text` is the most complex of the primitive types because:

* the language supports `Text` interpolation
* the language supports multi-line `Text` literals

In the simple case a `Text` literal is surrounded by double quotes:

```dhall
"Hello, world!"
```

... and these literals permit escape sequences similar to JSON escape sequences
using backslash:

```dhall
"Line 1\nLine 2\n"
```

The full set of escape sequences are:

* `\"` - Escape a quotation mark
* `\\` - Escape a backslash
* `\/` - Escape a forward slash
* `\b` - Escape a backspace
* `\f` - Escape a form feed
* `\n` - Escape a line feed
* `\r` - Escape a carriage return
* `\t` - Escape a tab
* `\$` - Escape a dollar sign
* `\uXXXX` / `\u{XXXX}` - Escape a Unicode sequence (specified using hex)

The language also permits Unicode characters in `Text` literals, too.

> **Exercise:** Run the following command to test Dhall's support for special
> characters:
>
> ```bash
> $ dhall <<< '"🍋\t🍓\t🍍\t🍉\t🍌\n\u{1F60B} \"Yum!\"\n"'
> ```

### Multi-line `Text` literals

Dhall supports multi-line `Text` literals surrounded by two single quotes
on both side, like this:

```bash
$ dhall <<< '"Line 1\nLine 2\nLine 3\n"'
```
```dhall
''
Line 1
Line 2
Line 3
''
```

You can think of the two single quotes on each side as "big double quotes" if
you need a convenient mnemonic for this feature.

Multi-line `Text` literals automatically strip leading indentation for all
lines after the opening quotes, meaning that this expression

```dhall
''
ABC
DEF
'''
```

... is the same as this expression

```dhall
    ''
    ABC
    DEF
    ''
```

... which is also the same as this expression:

```dhall
                   ''
    ABC
    DEF
    ''
```

... all three of which are syntactic sugar for the following plain `Text`
literal:

```dhall
"ABC\nDEF\n"
```

However, the preceding multi-line literals are **NOT** the same as this
expression:

```dhall
''
    ABC
    DEF
''
```

... which desugars to:

```dhall
"    ABC\n    DEF\n"
```

This latter multi-line string literal does not strip the leading four-space
prefix because there are not four spaces before the closing quotes.

However, blank lines within the interior of the multi-line string literal
are ignored for the purposes of computing the shared indentation to strip.  For
example, this expression does not have leading spaces in the middle line:

```dhall
␠␠␠␠␠␠''
␠␠␠␠␠␠foo

␠␠␠␠␠␠bar
␠␠␠␠␠␠''
```

... but the indentation is still stripped, making the expression equivalent to:

```dhall
''
foo

bar
''
```

Both leading tabs and leading spaces are stripped in this way, so long as the
pattern of tabs and spaces match for each line that isn't blank.

All multi-line string literals begin with an obligatory newline character (which
is not included in the final string).  For example, this is not valid:

```dhall
''ABC''
```

... but this is valid:

```dhall
''
ABC''
```

... which desugars to:

```dhall
"ABC"
```

> **Exercise:** What does this multi-line `Text` literal desugar to?
>
> ```dhall
>       ''
>     ABC
>   DEF
> ''
> ```
>
> Test your guess by interpreting the equivalent plain `Text` literal (which th

### `Text` interpolation

You can also interpolate Dhall expressions into `Text` using `${…}`, like this:

```dhall
let greeting = "Hello"

in  "${greeting}, world!"
```

The expression that you interpolate must have type `Text` and the language will
not automatically convert non-`Text` values to `Text`.  For example, this will
not type-check:

```dhall
let answer = 42

in  "The answer to life, the universe, and everything: ${answer}"
```

You have to instead render values as `Text` using explicit conversion functions,
like this:

```dhall
let answer = 42

in  "The answer to life, the universe, and everything: ${Natural/show answer}"
```
