---
icon: lucide/braces
---

# Advanced XPath 3 expressions

The [XPath section](../xpath/index.md) covers 1.0 — location paths, axes,
predicates, the four-type model. XPath **2.0 and 3.1** (the version XSLT 3.0
carries) add a layer of *expression* syntax on top of paths: ways to map, bind,
quantify, branch, pipe, and test types inline, without an `xsl:for-each` or an
`xsl:variable` for every intermediate step.

This page is a tour of the expressions you will actually reach for, built on the
familiar catalog:

``` xml title="catalog.xml"
<catalog>
  <cd genre="rock"><title>Empire Burlesque</title><artist>Bob Dylan</artist><price>10.90</price><year>1985</year></cd>
  <cd genre="pop"><title>Hide your heart</title><artist>Bonnie Tyler</artist><price>9.90</price></cd>
  <cd genre="country"><title>Greatest Hits</title><artist>Dolly Parton</artist><price>9.90</price></cd>
</catalog>
```

## The `!` simple map operator

`!` is the **simple map operator**, added in XPath 3.0. `A ! B` evaluates the
expression `B` **once for every item** in `A`, with that item as the context
item, and concatenates the results into one sequence:

``` xpath
catalog/cd ! title          (1)
catalog/cd ! upper-case(title)   (2)
(1, 2, 3) ! (. * .)         (3)
```

1.  For each `cd`, evaluate `title` — the three title elements.
2.  For each `cd`, compute `upper-case(title)` — three **strings**.
3.  `.` is the current item; this yields `1, 4, 9`. The left side is *atomics*,
    not nodes — something `/` cannot do.

<div class="xslt-result" markdown>
`(1, 2, 3) ! (. * .)` → `1 4 9`
</div>

### `!` versus `/`

They look similar — both "do something for each item on the left" — but they are
not interchangeable. The path operator `/` does three things that `!` does
**not**:

| | `/` (path step) | `!` (simple map) |
| --- | --- | --- |
| Right side must yield | **nodes** | **anything** (atomics, maps, nodes…) |
| Result order | **document order** | **iteration order** (left-to-right) |
| Duplicate nodes | **removed** | **kept** |

Three consequences fall out of that table:

``` xpath
(1, 2, 3) ! (. * 2)          (1)
(1, 2, 3) / (. * 2)          (2)
catalog//cd ! title          (3)
(catalog/cd | catalog/cd)    (4)
```

1.  `2, 4, 6` — the left side is atomic, and `!` is happy with that.
2.  **Error.** The left operand of `/` *must* be a sequence of nodes; an atomic
    value on the left is not allowed. This is the cleanest reason `!` exists.
3.  Order matters: a path like `a//b` returns nodes in **document order** even
    when the navigation reached them out of order. `!` instead keeps the
    left-to-right **iteration order** of its input.
4.  Union sorts into document order and removes duplicates → **3** nodes. The
    same nodes mapped with `!` (e.g. `catalog/cd ! .` twice over) keep every
    occurrence — `!` never deduplicates.

!!! tip "Rule of thumb"
    Use `/` when you are **navigating to nodes** and want document order with no
    duplicates — the normal case. Reach for `!` when the per-item result is a
    **computed value** (a number, a string, a map) or when you specifically want
    to **keep order and duplicates**. `catalog/cd ! string-length(title)` is the
    idiomatic "title length of each CD."

!!! warning "Don't confuse it with `!=` or the annotation `(1)!`"
    `!=` is the *not-equal* comparison — a different token entirely. And the
    `<!-- (1)! -->` markers in this book's code samples are documentation
    annotations, not XPath. The simple-map `!` is always *binary*: something on
    each side.

## The `=>` arrow operator

`=>` (3.0) feeds the value on its left in as the **first argument** of the
function call on its right. It turns nested calls inside-out so they read in the
order data flows:

``` xpath
<!-- nested, read inside-out: -->
string-join(sort(distinct-values(catalog/cd/@genre)), ', ')

<!-- piped, read left-to-right: -->
catalog/cd/@genre => distinct-values() => sort() => string-join(', ')
```

<div class="xslt-result" markdown>
`country, pop, rock`
</div>

Both forms are identical to the processor; the arrow is purely about
readability. It pairs naturally with `!`: map to values, then pipe them onward —
`catalog/cd ! xs:decimal(price) => sum()`. See
[higher-order functions](higher-order-functions.md) for `=>` with function
values.

## `for` — mapping with a bound variable

`!` maps with `.`; `for $x in … return …` maps with a **named** variable, which
you need when expressions nest and `.` would be ambiguous. The result is again a
flattened sequence:

``` xpath
for $cd in catalog/cd
return concat($cd/title, ' (', $cd/@genre, ')')
```

<div class="xslt-result" markdown>
`Empire Burlesque (rock)`, `Hide your heart (pop)`, `Greatest Hits (country)`
</div>

Multiple `in` clauses make a **cross product**, evaluated in nested-loop order —
handy for pairing every item with every other:

``` xpath
for $a in (1, 2), $b in ('x', 'y')
return concat($a, $b)            (1)
```

1.  `1x, 1y, 2x, 2y` — the second variable varies fastest.

## `let` — naming a subexpression inline

`let $v := … return …` binds a value without an `xsl:variable`, so you can name
a repeated subexpression right inside one XPath:

``` xpath
let $avg := avg(catalog/cd/xs:decimal(price))
return catalog/cd[xs:decimal(price) gt $avg]/title   (1)
```

1.  Compute the average once, then select the titles priced above it →
    `Empire Burlesque`. Without `let`, you would repeat the whole `avg(…)` inside
    the predicate.

`let` and `for` compose freely: `for $cd in catalog/cd let $p := xs:decimal($cd/price) return …`.

## `some` / `every` — quantified tests

These answer "any?" and "all?" over a sequence, returning a single boolean:

``` xpath
some $p in catalog/cd/price satisfies xs:decimal($p) gt 10     (1)
every $p in catalog/cd/price satisfies xs:decimal($p) gt 0     (2)
```

1.  `true` — at least one CD costs more than 10.
2.  `true` — all prices are positive.

They make the 1.0 node-set `=` trap explicit: instead of relying on the
existential quirk of `price = '9.90'`, you *say which* quantifier you mean.

## `if`/`then`/`else` — a conditional that is an expression

XPath's own conditional returns a value, so it slots inside a larger expression —
no `xsl:choose` needed for a simple branch. The `else` is mandatory:

``` xpath
catalog/cd ! (if (xs:decimal(price) gt 10) then 'premium' else 'standard')
```

<div class="xslt-result" markdown>
`premium`, `standard`, `standard`
</div>

## Type expressions: `instance of`, `cast`, `treat`

The sequence type system (see [sequences and types](sequences-and-types.md))
shows up inline as four operators:

``` xpath
$x instance of xs:integer+         (1)
$s castable as xs:date             (2)
xs:decimal(price)                  (3)
$node treat as element(cd)         (4)
```

1.  Boolean: is `$x` one-or-more integers? A safe guard before using a value.
2.  Boolean: *would* casting `$s` to a date succeed? Test before you `cast` to
    avoid a runtime error.
3.  An actual cast — `xs:decimal(price)` turns the price text into a typed
    decimal so arithmetic and comparison behave numerically.
4.  An assertion to the type checker that `$node` is a `cd` — changes the static
    type, not the value; errors at runtime if untrue.

## The `?` lookup operator

For [maps and arrays](maps-and-arrays.md) (and JSON), `?` reaches into a
structure by key or position, and unary `?*` takes everything:

``` xpath
$map?title                  (1)
$array?1                    (2)
$arrayOfMaps?*?price        (3)
```

1.  The value under key `title` in a map.
2.  The first member of an array (arrays are 1-based).
3.  The `price` of every map in an array — `?*` fans out, then `?price` maps over
    the results. This is how you walk parsed JSON.

## Putting it together

A single expression that combines several of these — average price per genre,
formatted, sorted, as one string — shows why the operators earn their keep:

``` xpath
distinct-values(catalog/cd/@genre)
  => sort()
  ! (let $g := .
     return concat($g, ': ',
       avg(catalog/cd[@genre = $g] ! xs:decimal(price))))
  => string-join('; ')
```

<div class="xslt-result" markdown>
`country: 9.9; pop: 9.9; rock: 10.9`
</div>

Read it as a pipeline: the genres, deduplicated and sorted, then **mapped**
(`!`) — binding each to `$g` with `let` so the inner predicate can reuse it — to
a `"genre: average"` string, then joined. The same result in 1.0 would take a
Muenchian key, a sort, and several templates.

## Where to go next

- [Sequences and types](sequences-and-types.md) — the model these operators compute over.
- [Higher-order functions](higher-order-functions.md) — `=>` with function values, `fold-left`, `filter`.
- [Maps and arrays](maps-and-arrays.md) — the `?` lookup operator in full.
- [Regular expressions and strings](regex.md) — `matches`, `replace`, `tokenize`.
