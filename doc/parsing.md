Interpreting the output of tree-sitter
==

Introduction to the problem
--

Ideally, the result of tree-sitter parsing should be a CST that can
be used directly. It is not the case because the json (or C) tree we get
only has nodes for named rules, lacking details on which path was
taken within a rule.

The following grammar is unproblematic:

```javascript
sum: $ => seq($.number, '+', $.number)
```

The above translates to the OCaml type definition

```ocaml
type sum = number * Token.t * number
```

The output of parsing `1 + 1` is a `sum` node with 3 children `1`,
`+`, and `1`, whose json representation is essentially the following
(source location info omitted for brevity):

```json
{
  "type": "sum",
  "children": [
    { "type": "number" },
    { "type": "+" },
    { "type": "number }
  ]
}
```

However, other constructs than `seq()` are problematic because they
have multiple ways of matching, and the result doesn't indicate which
branch was taken. These constructs are:

* `choice()`
* `repeat()`
* `repeat1()`
* `optional()`

A simple grammar that results in ambiguous output is this:

```javascript
nums: $ => seq(optional($.number),
               optional($.number))
```

The OCaml CST type for this is:

```ocaml
type nums = number option * number option
```

Giving the input `42` to the tree-sitter parser results in just this
output:

```json
{
  "type": "nums",
  "children": [
    { "type": "number" },
  ]
}
```

This doesn't tell us which of the two options was matched. The OCaml
value could be `(Some "42", None)` or `(None, Some "42")`, depending
on parsing order. In this case, our OCaml parser proceeds from left to
right, longest-match-first, and will end up with `(Some "42", None)`.

Solution and trade-offs
--

We just saw that the parsing result of tree-sitter doesn't include
nodes for the different kinds of repeats (`optional()`, `repeat()`,
`repeat1()`). It also doesn't indicate which branch of a `choice()` was
matched.

For example, we could have something like this:

```javascript
exp: $ => choice(
  seq($.number, '+', $.number),
  seq($.number, '*', $.number)
)
```

The OCaml type for this is:
```ocaml
type exp = [
  | `Exp_num_PLUS_num of (number * Token.t * number)
  | `Exp_num_STAR_num of (number * Token.t * number)
]
```

Note the names that were generated by ocaml-tree-sitter for each
alternative. See [cst.md](cst.md) for a review of naming issues.

The json output of parsing `2 * 2` is the following:
```json
{
  "type": "exp",
  "children": [
    { "type": "number" },
    { "type": "*" },
    { "type": "number" }
  ]
}
```

The OCaml value that we recover from that needs to be
`` `Exp_num_STAR_num (..., ..., ...) ``.

In the end, the problem is to match a sequence of named tokens
(`number`, `*`, `number`) against a tree forming a regular
expression (combination of symbols, sequences, repeats, and alternatives).

Trade-offs:

* We don't bother with whether tree-sitter GLR parsing would
  prioritize one path over another, or whether it would never match
  a specific path. We proceed from left to right, longest match first,
  and then backtrack as needed.
* This is fine in practice because these regular expressions to match
  against `children` sequences of symbols are relatively simple and
  normally unambiguous.

The job of matching a regular expression over a sequence of symbols
and turning it into the appropriate OCaml CST node is done by
generated code. This is the code in `Parse.ml`, which is plain and
straightforward to understand. It is recommended to run the tests with
`make test` and then inspect the generated code for a simple example
under `tests`. We also maintain a handwritten example of such code as it
should be generated. This is the file
[`Sample.ml`](../src/run/lib/Sample.ml). It goes through typechecking
and serves as a model when making changes in the generated code.
