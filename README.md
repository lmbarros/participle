# A dead simple parser package for Go [![](https://godoc.org/github.com/alecthomas/participle?status.svg)](http://godoc.org/github.com/alecthomas/participle) [![Build Status](https://travis-ci.org/alecthomas/participle.svg?branch=master)](https://travis-ci.org/alecthomas/participle) [![Go Report Card](https://goreportcard.com/badge/github.com/alecthomas/participle)](https://goreportcard.com/report/github.com/alecthomas/participle)

The goals of this package are:

1. Provide a simple, idiomatic and elegant way to define parsers.
2. Allow generation of very fast parsers from this definition.

Conceptually, Participle operates similarly to how the JSON package works;
annotations on the struct define how this mapping occurs.

## Overview

A grammar is an annotated Go structure used to both define the parser grammar,
and be the AST output by the parser:


```go
type Grammar struct {
  Hello string `@Ident`
}
```

A parser is constructed from a grammar and a lexer:

```go
parser, err := participle.Build(&Grammar{}, nil)
```

Once constructed, the parser is applied to input to produce an AST:

```go
ast := &Grammar{}
err := parser.ParseString("world", ast)
// ast == &Grammar{Hello: "world"}
```

## Annotation syntax

- `@<expr>` Capture expression into the field.
- `@@` Recursively capture using the fields own type.
- `<identifier>` Match named lexer token.
- `{ ... }` Match 0 or more times.
- `( ... )` Group.
- `[ ... ]` Optional.
- `"..."` Match the literal.
- `<expr> <expr> ...` Match expressions.
- `<expr> | <expr>` Match one of the alternatives.

Notes:

- Each struct is a single production, with each field applied in sequence.
- `@<expr>` is the mechanism for capturing matches into the field.
- if a struct field is not keyed with "parser", the entire struct tag
  will be used as the grammar fragment. This allows the grammar syntax to remain
  clear and simple to maintain.


## Capturing

Prefixing any expression in the grammar with `@` will capture matching values
for that expression into the corresponding field.

For example:

```go
// The grammar definition.
type Grammar struct {
  Hello string `@Ident`
}

// The source text to parse.
source := "world"

// After parsing, the resulting AST.
result == &Grammar{
  Hello: "world",
}
```


For slice and string fields, each instance of `@` will accumulate into the
field (including repeated patterns). Accumulation into other types is not
supported.

A successful capture match into a boolean field will set the field to true.

For integer and floating point types, a successful capture will be parsed
with `strconv.ParseInt()` and `strconv.ParseBool()` respectively.

Custom control of how values are captured into fields can be achieved by a
field type implementing the `Capture` interface (`Capture(values []string)
error`).

## Lexing

Participle operates on tokens and thus relies on a lexer to convert character
streams to tokens.

Three lexers are provided, varying in speed and flexibility. The fastest lexer
is based on the [text/scanner](https://golang.org/pkg/text/scanner/) package
but only allows tokens provided by that package. Next fastest is the regexp
lexer (`lexer.Regexp()`). The slowest is currently the EBNF based lexer, but it has a large potential for optimisation through code generation.

To use your own Lexer you will need to implement two interfaces:
[Definition](https://godoc.org/github.com/alecthomas/participle/lexer#Definition)
and [Lexer](https://godoc.org/github.com/alecthomas/participle/lexer#Lexer).

## Example

There are several [examples](_examples) included in the source, but here is an
example of defining a parser for the form of EBNF used by `exp/ebnf`:

```go
package main

import (
  "fmt"
  "os"

  "github.com/alecthomas/participle"
)

type Group struct {
  Expression *Expression `'(' @@ ')'`
}

type Option struct {
  Expression *Expression `'[' @@ ']'`
}

type Repetition struct {
  Expression *Expression `'{' @@ '}'`
}

type Literal struct {
  Start string `@String`
  End   string `[ '…' @String ]`
}

type Term struct {
  Name       string      `@Ident |`
  Literal    *Literal    `@@ |`
  Group      *Group      `@@ |`
  Option     *Option     `@@ |`
  Repetition *Repetition `@@`
}

type Sequence struct {
  Terms []*Term `@@ { @@ }`
}

type Expression struct {
  Alternatives []*Sequence `@@ { '|' @@ }`
}

type Expressions []*Expression

type Production struct {
  Name        string      `@Ident '='`
  Expressions Expressions `@@ { @@ } '.'`
}

type EBNF struct {
  Productions []*Production `{ @@ }`
}

func main() {
  parser, err := participle.Build(&EBNF{}, nil)
  if err != nil { panic(err) }

  ebnf := &EBNF{}
  err = parser.Parse(os.Stdin, ebnf)
  if err != nil { panic(err) }

  json.NewEncoder(os.Stdout).Encode(ebnf)
}
```

## Performance

One of the included examples is a complete Thrift parser
(shell-style comments are not supported). This gives
a convenient baseline for comparing to the PEG based
[pigeon](https://github.com/PuerkitoBio/pigeon), which is the parser used by
[go-thrift](https://github.com/samuel/go-thrift). Additionally, the pigeon
parser is utilising a generated parser, while the participle parser is built at
run time.

You can run the benchmarks yourself, but here's the output on my machine:

```
BenchmarkParticipleThrift-4        10000      221818 ns/op     48880 B/op     1240 allocs/op
BenchmarkGoThriftParser-4           2000      804709 ns/op    170301 B/op     3086 allocs/op
```

On a real life codebase of 47K lines of Thrift, Participle takes 200ms and go-
thrift takes 630ms, which aligns quite closely with the benchmarks.
