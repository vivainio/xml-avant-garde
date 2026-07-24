---
icon: lucide/layers-3
---

# Sequences, comparisons, and the XDM

XQuery does not pass around XML fragments as a special kind of value. Everything
is an **XDM sequence**: zero or more *items*, where an item is either a node or an
atomic value such as a string, number, date, map, array, or function.

That one rule explains how paths, FLWOR expressions, function arguments, and
constructed output fit together. It also explains the errors that surprise people
moving from XPath 1.0: an empty sequence is not an empty string, and comparing two
sequences depends on which comparison operator you choose.

## One value model

The comma constructs a sequence:

``` xquery
("rock", 1985, /catalog/cd[1])
```

This is one sequence of three items: a string, an integer, and an element node.
Sequences are ordered, may contain duplicates, and can mix item types.

There are two important edge cases:

``` xquery
()                    (: the empty sequence: zero items :)
("rock")              (: just the string "rock", not a wrapper :)
("rock", ("pop"))     (: the same as ("rock", "pop") :)
```

There is no one-item sequence distinct from its item, and no nested sequence.
Parentheses group expressions; they do not make a container. Maps and arrays are
real single items, so use an array when nesting or positional boundaries matter:

``` xquery
[("rock", "pop"), ("country")]     (: one array containing two members :)
```

## Cardinality is part of the type

XQuery sequence types say both *what kind of item* and *how many*:

| Type | Means |
| --- | --- |
| `xs:string` | exactly one string |
| `xs:string?` | zero or one string |
| `xs:string+` | one or more strings |
| `xs:string*` | zero or more strings |
| `element(cd)` | exactly one `cd` element |
| `node()*` | any number of nodes |
| `item()*` | any sequence at all |
| `empty-sequence()` | no items |

Put those contracts on variables and functions and the processor catches mistakes
where they originate:

``` xquery
declare function local:price($cd as element(cd)) as xs:decimal {
  xs:decimal($cd/price)
};
```

Calling `local:price(())` is a type error: the parameter requires exactly one
`cd`. So is passing two elements. A genuinely optional input should say so with
`element(cd)?`.

## Empty is not blank

These values are different:

``` xquery
()                    (: no item :)
""                    (: one string containing no characters :)
<title/>               (: one element node with an empty string value :)
```

Test for absence with `empty($value)` and `exists($value)`, not with a string
comparison:

``` xquery
for $cd in /catalog/cd
return
  if (empty($cd/year))
  then concat($cd/title, " — year unknown")
  else concat($cd/title, " — ", $cd/year)
```

`count(())` is `0`; `string(())` is `""`; but a function declared to accept one
string does not accept `()` merely because those can look alike when displayed.

## Atomization: from nodes to their typed values

Paths return nodes. Arithmetic and most comparisons need atomic values, so XQuery
**atomizes** the nodes: it takes each node's typed value. With untyped input XML,
an element's typed value is usually `xs:untypedAtomic`; casting makes the intended
type explicit:

``` xquery
let $price-node := /catalog/cd[1]/price
let $price := xs:decimal($price-node)
return ($price-node instance of element(price), $price instance of xs:decimal)
```

The result is `(true(), true())`: the variables hold different kinds of item even
though both serialize visibly as `10.90`.

This matters for sorting and arithmetic. Strings sort lexically (`"10"` before
`"9"`); decimals sort numerically. Prefer:

``` xquery
order by xs:decimal($cd/price)
```

## General comparisons: does any pair match?

The familiar operators `=`, `!=`, `<`, `<=`, `>`, and `>=` are **general
comparisons**. They atomize both operands and return true if *any pair* of values
satisfies the comparison:

``` xquery
(9.90, 10.90) = 9.90       (: true :)
(9.90, 10.90) != 9.90      (: also true: 10.90 differs :)
```

This existential behavior is convenient for XML:

``` xquery
/catalog/cd/@genre = ("rock", "country")
```

It asks whether any genre equals any value in the allowed sequence. But `!=` is
not the negation of `=` when sequences contain several values. Write
`not($a = $b)` when that is what you mean.

## Value comparisons: compare one with one

The word operators `eq`, `ne`, `lt`, `le`, `gt`, and `ge` are **value
comparisons**. Each operand must be empty or a singleton:

``` xquery
xs:decimal(/catalog/cd[1]/price) gt 10       (: true :)
/catalog/cd/price eq 9.90                    (: error: three prices :)
```

If either operand is empty, a value comparison returns the empty sequence rather
than `false()`:

``` xquery
()/year eq 1985             (: () :)
boolean(()/year eq 1985)    (: false() :)
```

Use value comparisons when the data model promises one value and violating that
promise should be visible. Use general comparisons when “any matching value” is
the intended question.

| Intent | Operator |
| --- | --- |
| Does any item on the left match any on the right? | `=`, `!=`, `<`, … |
| Compare two required or optional singleton values | `eq`, `ne`, `lt`, … |
| Are these the exact same node? | `is` |
| Is one node before another in document order? | `<<`, `>>` |

## Node identity is not value equality

Two elements can contain the same text without being the same node:

``` xquery
let $a := <title>Empire Burlesque</title>
let $b := <title>Empire Burlesque</title>
return (
  $a = $b,       (: true: their atomized string values match :)
  $a is $b       (: false: they are distinct constructed nodes :)
)
```

Use `is` for identity and `<<` / `>>` for relative document order. All three
require one node on each side (or an empty operand).

## Effective boolean value

An `if`, `where`, `and`, or `or` needs a boolean. XQuery computes an
expression's **effective boolean value** (EBV):

- an empty sequence is false;
- a sequence whose first item is a node is true;
- a single boolean, string, URI, or number converts in the expected way;
- most other atomic sequences are an error.

That makes the everyday existence test concise:

``` xquery
where $cd/year
```

It is true when the path finds a node. But this is an error, not true:

``` xquery
if (("rock", "pop")) then "yes" else "no"
```

The processor refuses to guess what a two-string sequence means. State the
question: `exists($genres)`, `$genres = "rock"`, or
`every $g in $genres satisfies string-length($g) gt 0`.

## Paths normalize nodes; commas do not

A path expression returns nodes in document order with duplicates removed. A
comma preserves order and duplicates:

``` xquery
let $title := /catalog/cd[1]/title
return (
  ($title, $title),          (: two references to the same node :)
  ($title | $title)          (: one node: union removes the duplicate :)
)
```

This distinction matters when assembling results. A FLWOR sequence represents
rows and may intentionally repeat values; a path represents navigation through a
tree and normalizes its node result.

## A practical debugging pattern

When an expression behaves unexpectedly, inspect its count and types before
changing the query:

``` xquery
let $value := /catalog/cd/price
return (
  count($value),
  $value ! string(.),
  $value ! type-name(.)
)
```

The simple map operator `!` evaluates the right side once for each item. Here it
shows that `$value` contains three element nodes—not one number. That observation
usually reveals whether the fix is a predicate, aggregation, cast, or different
comparison operator.

## Where to go next

Now put sequences and comparisons to work across more than one document:
[Joining XML documents](joins.md) uses FLWOR bindings to connect catalog entries
to separate artist and label data.

