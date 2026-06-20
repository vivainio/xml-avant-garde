# Moving to XSLT 2.0 and 3.0

You know XSLT 1.0 well. The good news is that everything you have learned still
applies — 2.0 and 3.0 are supersets. The same templates, the same
`apply-templates`, the same XPath. What changes is that several things that were
awkward or impossible in 1.0 become natural, and a few rough edges (result tree
fragments, no user functions, no regex) simply disappear.

This page is a map of what is new and why it matters. The pages that follow go
into each topic in depth.

## Declaring the version

You switch versions with one attribute on the stylesheet element:

``` xml title="catalog-3.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="3.0"                                  <!-- (1)! -->
                xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                xmlns:xs="http://www.w3.org/2001/XMLSchema">   <!-- (2)! -->

  <xsl:output method="html" indent="yes"/>

  <xsl:template match="/">
    <p>A 3.0 stylesheet.</p>
  </xsl:template>

</xsl:stylesheet>
```

1.  `version="3.0"` (or `"2.0"`) instead of `"1.0"`. That is the whole switch.
2.  The `xs:` namespace is what lets you name types like `xs:string` and
    `xs:integer` in `as=` attributes (see below).

!!! note "Browsers do 1.0 only; you need Saxon"
    Web browsers ship a 1.0 processor and nothing newer, so a stylesheet
    declaring `version="2.0"` or `"3.0"` will **not** run in the browser. For
    2.0/3.0 you run the transformation through a modern processor — **Saxon** is
    the reference implementation and the one these pages assume. The version
    declaration is also a request, not a guarantee: a 1.0 engine handed a
    `version="3.0"` stylesheet will fall back to 1.0 behaviour rather than error.

## The headline change: sequences

XSLT 1.0 worked on **node-sets** — unordered collections of nodes, and nothing
else. A value was either a node-set, a string, a number, or a boolean, and these
lived in separate worlds.

2.0 and 3.0 replace the node-set with the **sequence**: an *ordered* list of
items, where each item is either a node *or* an atomic value, and the two may be
mixed freely. A single string is just a sequence of length one. This one idea
removes a surprising amount of friction.

``` xml
<xsl:variable name="nums"   select="(3, 1, 2)"/>          <!-- (1)! -->
<xsl:variable name="mixed"  select="('total', sum(catalog/cd/price))"/> <!-- (2)! -->
<xsl:variable name="titles" select="catalog/cd/title"/>   <!-- (3)! -->
```

1.  A sequence of three integers, in the order written.
2.  A sequence mixing a string and a number — impossible as a 1.0 node-set.
3.  A sequence of nodes, just like an old node-set, but now ordered.

The most practical payoff: a **temporary tree** built with the content form of
`xsl:variable` is a real, navigable node tree. In 1.0 the same construct gave
you a *result tree fragment* you could copy into the output but not apply XPath
steps to without a `node-set()` extension (see [Variables](variables.md)). In
2.0/3.0 that restriction is gone — you can select into it directly:

``` xml
<xsl:variable name="cheap">                                <!-- (1)! -->
  <cd><title>Sample One</title><price>5.50</price></cd>
  <cd><title>Sample Two</title><price>7.25</price></cd>
</xsl:variable>

<xsl:value-of select="$cheap/cd[price &lt; 6]/title"/>     <!-- (2)! -->
```

1.  A temporary tree — the content form, exactly as in 1.0.
2.  In 1.0 this XPath into `$cheap` would have been an error. In 2.0/3.0 it just
    works.

<div class="xslt-result" markdown>
Sample One
</div>

## Typing values with `as=`

In 1.0 nothing had a declared type; a wrong assumption surfaced late, as
mangled output. 2.0/3.0 let you state the **sequence type** of a variable,
parameter, or function result with an `as=` attribute. The processor then checks
it, so mistakes show up earlier and with a clear message.

A sequence type is an item type plus an optional **cardinality marker**:

| Marker | Means |
| --- | --- |
| *(none)* | exactly one |
| `?` | zero or one |
| `*` | zero or more |
| `+` | one or more |

The item type can be an atomic type (`xs:string`, `xs:integer`, `xs:decimal`,
`xs:date`, …) or a node test such as `element(cd)`.

``` xml title="typed.xsl" linenums="1"
<xsl:variable name="count" as="xs:integer" select="count(catalog/cd)"/>   <!-- (1)! -->
<xsl:variable name="total" as="xs:decimal" select="sum(catalog/cd/price)"/>
<xsl:variable name="when"  as="xs:date"    select="xs:date('2026-06-20')"/>

<xsl:param    name="label" as="xs:string"  select="'CD Collection'"/>     <!-- (2)! -->
<xsl:param    name="discs" as="element(cd)*"/>                            <!-- (3)! -->
```

1.  Exactly one integer. If the `select` produced something else, the processor
    complains here rather than later.
2.  A typed parameter, with a default; a caller may still override it.
3.  Zero or more `cd` elements — `element(cd)` is the item type, `*` the
    cardinality.

!!! tip "Types catch mistakes early"
    `as="xs:integer"` on a variable you intend to count with means a stray
    decimal or empty sequence is reported at the point of binding, with a type
    error, instead of silently flowing into your output. Adding `as=` to the
    public parameters and functions of a stylesheet is cheap insurance.

## `xsl:sequence` vs `xsl:value-of`

`xsl:value-of` has always produced a **text node**: it takes its value and
flattens it to a string. 2.0/3.0 add `xsl:sequence`, which returns the actual
**items** — nodes or atomic values — without that flattening. It is how you
return a real result from a function or a template, rather than its string form.

``` xml
<xsl:value-of select="catalog/cd[1]"/>   <!-- (1)! -->
<xsl:sequence select="catalog/cd[1]"/>   <!-- (2)! -->
```

1.  Emits the *string value* of the first `cd` — its text, run together.
2.  Returns the `cd` *element itself* — still a node, with structure intact.

## New XPath expressions

XPath grew a small set of expressions you will reach for constantly:

``` xml
for $p in catalog/cd/price return $p * 1.1     <!-- (1)! -->
let $n := count(catalog/cd) return $n div 2    <!-- (2)! -->
some $p in catalog/cd/price satisfies $p &gt; 10  <!-- (3)! -->
every $p in catalog/cd/price satisfies $p &gt; 0  <!-- (4)! -->
catalog/cd/price => sum()                      <!-- (5)! -->
```

1.  `for … return` maps over a sequence, producing a new sequence.
2.  `let … return` binds a value inline (2.0 in XQuery, available in XSLT via
    `xsl:variable`; the inline `let` is an XPath 3.0 expression).
3.  `some … satisfies` is true if *any* item matches.
4.  `every … satisfies` is true if *all* items match.
5.  The `=>` **arrow operator** (3.0) feeds the left value as the first argument
    of the call on the right — `catalog/cd/price => sum()` is `sum(catalog/cd/price)`,
    but reads left-to-right and chains cleanly.

## What else is new

A quick map of the bigger features, each covered on its own page:

| Feature | One-liner | Page |
| --- | --- | --- |
| Sequences & types | The `as=` type system that underpins everything below | [sequences and types](sequences-and-types.md) |
| User-defined functions | Write your own XPath functions with `xsl:function` | [functions](functions.md) |
| Higher-order functions | Functions as values: `fold-left`, `filter`, `=>` | [higher-order functions](higher-order-functions.md) |
| Maps & arrays | Dictionaries and nested lists, the 3.1 data structures | [maps and arrays](maps-and-arrays.md) |
| Reading & writing JSON | `parse-json`, maps & arrays, `method="json"` | [JSON](json.md) |
| Grouping | `xsl:for-each-group` replaces the 1.0 "Muenchian" trick | [grouping](grouping.md) |
| Regex & strings | `matches`, `replace`, `tokenize`, `xsl:analyze-string` | [regex](regex.md) |
| Dates & times | Real date types, arithmetic, and `format-date` | [dates and times](dates-and-times.md) |
| Modern identity & text | Declarative `xsl:mode` identity and text value templates | [modern identity and text](modern-identity.md) |
| Multiple outputs | `xsl:result-document` writes many files in one run | [result documents](result-documents.md) |
| Error handling | `xsl:try`/`xsl:catch`, `xsl:assert` | [error handling](error-handling.md) |
| Streaming | Process documents too large to fit in memory | [streaming](streaming.md) |
| Packages | `xsl:package` — real modules with visibility | [packages](packages.md) |

## 1.0 vs 2.0/3.0 at a glance

| | XSLT 1.0 | XSLT 2.0 / 3.0 |
| --- | --- | --- |
| Core value | Node-set (unordered) | Sequence (ordered, may mix nodes + atomics) |
| Temporary tree | Result tree fragment — needs `node-set()` to navigate | Real node tree, navigable directly |
| Typing | None | `as=` sequence types, checked early |
| Returning items | `xsl:value-of` (text only) | `xsl:value-of` *or* `xsl:sequence` (items) |
| Own functions | Named templates only | `xsl:function` (callable in XPath) |
| Grouping | Hand-rolled (Muenchian keys) | `xsl:for-each-group` |
| String tools | `translate`, `substring-before/after` | `matches`, `replace`, `tokenize`, `upper-case`, … |
| XPath | Location paths, basic predicates | `for` / `let` / `some` / `every`, `=>` (3.0) |
| Runs in | Every browser and processor | Saxon and other modern processors only |

## Next

[Sequences and types](sequences-and-types.md) — the model under everything on
this list: what a sequence is, and how `as=` types turn silent bugs into
compile-time errors.
