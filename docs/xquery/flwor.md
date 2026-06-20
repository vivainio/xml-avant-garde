---
icon: lucide/flower-2
---

# FLWOR expressions

A **FLWOR** expression is XQuery's loop. The name is its five clauses in order
— **F**or, **L**et, **W**here, **O**rder by, **R**eturn — and it reads top to
bottom like a `SELECT` statement turned sideways:

``` xquery
for $cd in /catalog/cd      (1)
let $price := $cd/price      (2)
where $price > 9.90          (3)
order by $cd/artist          (4)
return $cd/title             (5)
```

1. **`for`** — bind `$cd` to *each* item of a sequence, one iteration per item.
2. **`let`** — bind a helper variable for this iteration (no looping; one value).
3. **`where`** — keep only the iterations whose test is true.
4. **`order by`** — sort the surviving iterations.
5. **`return`** — the result produced for each iteration; the results concatenate
   into one sequence.

Only `for` (or `let`) and `return` are required. The other clauses are optional
and can repeat.

## `for` — iterate

`for` is the workhorse. Each binding runs the rest of the expression once per
item, in document order:

``` xquery
for $cd in /catalog/cd
return $cd/title
```

``` xml title="result"
<title>Empire Burlesque</title>
<title>Hide your heart</title>
<title>Greatest Hits</title>
```

Two `for` clauses produce a **cross join**, exactly like nested loops or a
comma-separated `FROM` in SQL. Add `at $i` to capture the position:

``` xquery
for $cd at $i in /catalog/cd
return concat($i, ". ", $cd/title)
```

## `let` — bind without looping

`let` binds a name to a *single* value — the whole sequence at once, not one
item at a time. It is for naming sub-expressions you reuse:

``` xquery
for $cd in /catalog/cd
let $price := xs:decimal($cd/price)
where $price > 9.90
return $cd/title
```

The `xs:decimal(...)` cast matters: `$cd/price` is a node, and comparing a node
to `9.90` works by atomization, but casting makes the numeric intent explicit
and lets you do arithmetic on `$price` later.

!!! note "`:=` binds, `=` compares"
    XQuery uses `:=` in `let` and `for` bindings — the same spelling as in
    [XSLT 3.0's `let`](../xslt/moving-to-3.md), which it borrowed. A bare `=` is
    the general comparison operator, as in [XPath](../xpath/functions-and-types.md).

## `where` — filter

`where` drops iterations whose test is false. It is the FLWOR equivalent of an
XPath predicate, but it can reference *any* variable in scope, not just the
context node:

``` xquery
for $cd in /catalog/cd
where $cd/@genre = "rock" or xs:decimal($cd/price) < 9.95
return $cd/title
```

For a simple test on the iterated node, a predicate is shorter
(`/catalog/cd[@genre = "rock"]`); reach for `where` when the condition spans
several bound variables.

## `order by` — sort

`order by` sorts the result sequence — something XPath alone cannot do.
Multiple keys, `ascending`/`descending`, and `empty greatest|least` for missing
values all work:

``` xquery
for $cd in /catalog/cd
order by xs:decimal($cd/price) descending, $cd/title
return $cd/title
```

## `group by` — aggregate (3.0+)

`group by` collapses iterations that share a key into one. Inside the `return`,
the grouping variable is a single value, while every *other* variable becomes
the **whole group** — ready for `count`, `sum`, `avg`, `max`:

``` xquery
<genres>{
  for $cd in /catalog/cd
  group by $g := $cd/@genre
  order by $g
  return <genre name="{ $g }" count="{ count($cd) }"/>
}</genres>
```

``` xml title="result"
<genres>
  <genre name="country" count="1"/>
  <genre name="pop" count="1"/>
  <genre name="rock" count="1"/>
</genres>
```

This is the same job [XSLT does with `xsl:for-each-group`](../xslt/grouping.md) —
here it is one clause.

## Building output: element constructors

A FLWOR clause returns a *sequence*; to shape it into a document you wrap it in
a **direct element constructor** — literal XML with `{ … }` holes that splice
in computed values:

``` xquery
<report generated="{ current-date() }">{
  for $cd in /catalog/cd
  return <album>{ string($cd/title) }</album>
}</report>
```

Inside an *attribute* value, `{ … }` works too (`title="{ $cd/title }"`). When
the element name itself must be computed, use a **computed constructor**:
`element { $name } { $content }` and `attribute { $name } { $value }` — the
analogue of [`xsl:element`](../xslt/output.md).

!!! warning "Whitespace in constructors is literal"
    Text between tags in a direct constructor is kept verbatim, including
    newlines and indentation. Wrap dynamic content in `{ … }` and let the
    [serializer](../xslt/whitespace.md) handle pretty-printing, rather than
    relying on the layout of your query source.

## `input` functions

Standalone queries rarely start from a bare `/catalog`; they name their input:

| Function | What it loads |
| --- | --- |
| `doc("catalog.xml")` | a single document by URI |
| `collection("music")` | every document in a named collection (database-backed) |
| `fn:parse-xml($string)` | a document from a string of markup |
| `.` (context item) | whatever the host/CLI bound, e.g. `saxon -s:catalog.xml` |

The collection form is where XQuery earns its keep: `collection("music")//cd`
streams across thousands of indexed documents in a
[native XML database](real-world.md) as easily as `/catalog/cd` walks one file.

## Functions and modules

Define reusable functions with `declare function`, in your own namespace
(`local:` is built in for this):

``` xquery
declare function local:label($cd as element(cd)) as xs:string {
  concat($cd/title, " — ", $cd/artist)
};

for $cd in /catalog/cd
return local:label($cd)
```

Group related functions into a **library module** (`module namespace …`) and
pull it into a main query with `import module` — the same modular reuse XSLT
gets from [`xsl:include`/`xsl:import`](../xslt/reuse.md).

## Where to go next

- [XQuery vs XSLT](xquery-vs-xslt.md) — the same data model from two directions.
- [XQuery in the real world](real-world.md) — running these queries against a database.
