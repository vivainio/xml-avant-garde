# Functions and data types

A location path always evaluates to a node-set, but XPath also *computes*. Every
expression produces a value of one of **four** types, and the core function
library turns those values into one another — counting nodes, slicing strings,
adding prices, asking yes/no questions. This page is the catalogue of those
types, their conversion rules, and the functions every 1.0 processor provides.

The running document is the familiar catalog, where each `cd` carries a `genre`
attribute and the first keeps an optional `year`:

``` xml title="catalog.xml" linenums="1"
<catalog>
  <cd genre="rock"><title>Empire Burlesque</title><artist>Bob Dylan</artist><price>10.90</price><year>1985</year></cd>
  <cd genre="pop"><title>Hide your heart</title><artist>Bonnie Tyler</artist><price>9.90</price></cd>
  <cd genre="country"><title>Greatest Hits</title><artist>Dolly Parton</artist><price>9.90</price></cd>
</catalog>
```

## The four data types

XPath 1.0 has exactly four types. Everything an expression returns is one of
them:

| Type | What it holds | Example expression |
| --- | --- | --- |
| **node-set** | a set of nodes | `catalog/cd`, `//title` |
| **string** | a sequence of characters | `name(/catalog)`, `'rock'` |
| **number** | an IEEE 754 double | `count(//cd)`, `3.14`, `price * 2` |
| **boolean** | true or false | `@genre = 'rock'`, `not(year)` |

!!! info "No integer type"
    A 1.0 number is *always* a double-precision float — there is no separate
    integer type. `count(//cd)` is the double `3`, and division never truncates:
    `7 div 2` is `3.5`, not `3`. The special values `NaN` (not-a-number, e.g.
    `number('rock')`) and positive/negative infinity are numbers too.

The interesting part is what happens when a value of one type is used where
another is expected. XPath converts **implicitly**, and the rules — especially
for node-sets — are where most surprises live.

### A node-set as a string

When a node-set is used where a string is wanted, XPath takes the **string value
of the first node in document order** and ignores the rest. So `string(//title)`
is *not* all three titles — it is just the first:

``` xpath
string(//title)
```

<div class="xslt-result" markdown>
Empire Burlesque
</div>

The string value of an *element* is all of its text descendants concatenated;
the string value of an attribute is its value. An empty node-set converts to the
empty string `""`.

### A value in boolean context

In a predicate, an `and`/`or`, or inside `boolean()`, a value is coerced to
true/false:

- A **node-set** is true when it is **non-empty**, false when empty.
- A **string** is true when it is **non-empty** (`""` is false).
- A **number** is true unless it is `0` or `NaN`.
- A **boolean** is itself.

So `cd[year]` keeps the CDs that *have* a `year` child (the node-set is
non-empty), and `cd[@genre]` keeps those carrying a `genre` attribute:

``` xpath
catalog/cd[year]
```

<div class="xslt-result" markdown>
Empire Burlesque … (only the first CD has a `<year>`)
</div>

### Comparing a node-set with `=`

Comparison is where the node-set rules bite hardest. When **either** side of a
comparison is a node-set, the operator is *existential*: it is true if **any**
node in the set satisfies the comparison.

``` xpath
catalog/cd/price = '9.90'
```

<div class="xslt-result" markdown>
true — because *at least one* `price` equals `9.90` (two of them do)
</div>

!!! warning "Node-set `=` means 'any node matches'"
    `catalog/cd/price = '9.90'` does **not** ask "are all prices 9.90?" — it asks
    "is there *some* price equal to 9.90?". It is true here even though one CD
    costs 10.90.

    The consequence that catches everyone: `!=` is **not** the negation of `=`.
    `catalog/cd/price != '9.90'` is *also* true, because some price (10.90)
    differs from 9.90. To negate a `=` test, wrap it: `not(price = '9.90')`.
    When one side is a node-set, the value is compared against each node, and a
    single match anywhere makes the whole expression true.

## Operators

### Comparison

`=`, `!=`, `<`, `<=`, `>`, `>=`. Numeric comparisons (`<` and friends) convert
both sides to numbers first, so `price > 10` works on the text content directly.

!!! warning "Escape `<` inside XML attributes"
    Because XPath usually lives inside an XML attribute (an XSLT `select`,
    `match`, or `test`), a literal `<` would break the XML. Write it as `&lt;`:
    `test="price &lt; 10"`. `>` is legal but `&gt;` is conventional. See
    [XSLT predicates](../xslt/predicates.md).

### Arithmetic

`+`, `-`, `*`, and — note the spelling — `div` and `mod`, **not** `/` and `%`.
The slash is reserved for location-path steps and `%` has no meaning in XPath,
so division and remainder use word operators:

``` xpath
price * 2          (1)
10 div 4           (2)
10 mod 3           (3)
```

1. The first matched `price` doubled.
2. `2.5` — division is floating-point.
3. `1` — the remainder.

### Boolean

`and` and `or` combine boolean values (there is no `&&` / `||`). `or` is
short-circuit-friendly and lower precedence than `and`:

``` xpath
@genre = 'rock' or @genre = 'pop'
```

## The core function library

XPath 1.0 ships a fixed set of functions, conventionally grouped by the type
they work on. Signatures below use `node-set?` for an optional argument that
defaults to the context node.

### Node-set functions

| Function | Returns |
| --- | --- |
| `last()` | size of the current node-set (number) |
| `position()` | position of the context node (number) |
| `count(node-set)` | number of nodes in the set |
| `name(node-set?)` | the qualified name (e.g. `cd`) of the first node |
| `local-name(node-set?)` | the local part of that name, without prefix |
| `namespace-uri(node-set?)` | the namespace URI of the first node |
| `id(object)` | element(s) with the given `ID`-typed attribute |

``` xpath
count(catalog/cd)                       (1)
count(catalog/cd[@genre = 'rock'])      (2)
name(/catalog/cd)                       (3)
catalog/cd[position() = last()]/title   (4)
```

1. `3` — every CD.
2. `1` — only the rock CD.
3. `cd` — the name of the first matched element.
4. `Greatest Hits` — the title of the last CD.

<div class="xslt-result" markdown>
`count(catalog/cd)` → 3

`count(catalog/cd[@genre = 'rock'])` → 1

`name(/catalog/cd)` → cd

`catalog/cd[position() = last()]/title` → Greatest Hits
</div>

### String functions

| Function | Returns |
| --- | --- |
| `string(object?)` | the object converted to a string |
| `concat(s1, s2, …)` | the arguments joined (two or more) |
| `contains(haystack, needle)` | boolean: does `haystack` contain `needle`? |
| `starts-with(s, prefix)` | boolean: does `s` begin with `prefix`? |
| `substring(s, start, len?)` | a slice; **positions are 1-based** |
| `substring-before(s, sep)` | the part of `s` before the first `sep` |
| `substring-after(s, sep)` | the part of `s` after the first `sep` |
| `string-length(s?)` | number of characters |
| `normalize-space(s?)` | trims ends and collapses internal whitespace |
| `translate(s, from, to)` | per-character replacement |

``` xpath
concat(title, ' — ', artist)            (1)
substring-before(price, '.')            (2)
contains(@genre, 'rock')                (3)
```

1. `Empire Burlesque — Bob Dylan` (string value of the first CD's children).
2. `10` — the whole-currency part of the first price.
3. `true` for the rock CD.

<div class="xslt-result" markdown>
`concat(title, ' — ', artist)` → Empire Burlesque — Bob Dylan

`substring-before(price, '.')` → 10

`contains(@genre, 'rock')` → true
</div>

!!! tip "These are taught in depth on the XSLT side"
    The XSLT Tutorial works through the same string functions with fuller
    examples. See [String functions](../xslt/strings.md) rather than re-learning
    them here.

### Number functions

| Function | Returns |
| --- | --- |
| `number(object?)` | the object converted to a number |
| `sum(node-set)` | the sum of the string-values-as-numbers of every node |
| `round(x)` | `x` rounded to the nearest integer (.5 rounds up) |
| `floor(x)` | the largest integer not greater than `x` |
| `ceiling(x)` | the smallest integer not less than `x` |

`sum` is the one that surprises: it walks the **whole** node-set (unlike the
first-node string rule), converting each node's string value to a number:

``` xpath
sum(catalog/cd/price)       (1)
round(sum(catalog/cd/price) div count(catalog/cd))  (2)
```

1. `30.7` — `10.90 + 9.90 + 9.90`.
2. `10` — the average price, rounded.

<div class="xslt-result" markdown>
`sum(catalog/cd/price)` → 30.7

`round(sum(catalog/cd/price) div count(catalog/cd))` → 10
</div>

### Boolean functions

| Function | Returns |
| --- | --- |
| `boolean(object)` | the object converted to a boolean |
| `not(boolean)` | the logical negation |
| `true()` | the constant true |
| `false()` | the constant false |
| `lang(string)` | true if the context node's `xml:lang` matches |

There are no boolean *literals* in XPath 1.0 — you call `true()` and `false()`.
`not()` is the correct way to negate a node-set test, sidestepping the `!=`
trap above:

``` xpath
not(catalog/cd/price = '9.90')      (1)
boolean(catalog/cd[year])           (2)
```

1. `false` — there *is* a 9.90 price, so the negation is false.
2. `true` — at least one CD has a `year`, so the node-set is non-empty.

<div class="xslt-result" markdown>
`not(catalog/cd/price = '9.90')` → false

`boolean(catalog/cd[year])` → true
</div>

## At a glance

| Group | Functions |
| --- | --- |
| Node-set | `last`, `position`, `count`, `name`, `local-name`, `namespace-uri`, `id` |
| String | `string`, `concat`, `contains`, `starts-with`, `substring`, `substring-before`, `substring-after`, `string-length`, `normalize-space`, `translate` |
| Number | `number`, `sum`, `round`, `floor`, `ceiling` |
| Boolean | `boolean`, `not`, `true`, `false`, `lang` |

This is the *entire* 1.0 library — there is nothing else. If you need a function
that is not in this table, you are reaching for 2.0.

## A note on 2.0 and 3.0

XPath 2.0 and 3.0 rework the foundations:

- The four types give way to a **rich type system over sequences**, tied to XML
  Schema. A sequence is ordered and may mix atomic values and nodes, and a
  single item is just a sequence of length one.
- The function library grows enormously: **regular expressions**
  (`matches`, `replace`, `tokenize`), full **date and time** arithmetic,
  `string-join`, `upper-case` / `lower-case`, `min`, `max`, `avg`, `distinct-values`,
  and many more.
- New **operators and inline expressions**: the `!` simple map, the `=>` arrow,
  `for` / `let` / `if` / `some` / `every`, and the `instance of` / `cast` type
  tests — plus your **own functions**.

These need a 2.0+ processor — the same one behind XSLT 2.0/3.0. The XSLT
Tutorial covers the territory: [Moving to XSLT 2.0 and 3.0](../xslt/moving-to-3.md),
[advanced XPath 3 expressions](../xslt/xpath-advanced.md) for the operators above,
[Regular expressions and strings](../xslt/regex.md), and
[User-defined functions](../xslt/functions.md).

## Where next

You now know XPath 1.0 end to end: the [tree of nodes](index.md), how
[location paths](location-paths.md) and [axes](axes.md) walk it,
how [node tests and predicates](node-tests-predicates.md) filter each step, and —
on this page — the four data types, their conversion rules, and the core
function library that computes over them.

XPath is an addressing language; to *do* something with what it selects you need
a host. The [XSLT Tutorial](../xslt/index.md) is where this same XPath is put to
work transforming documents — every `select`, `match`, and `test` there is an
expression of exactly the kind you have just learned. When you outgrow the four
types, [Moving to XSLT 2.0 and 3.0](../xslt/moving-to-3.md) opens the sequence
model and the larger function library.

Back to the section [Overview](index.md).
