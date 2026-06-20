---
icon: lucide/list-ordered
---

# Sequences and types

The single biggest change from 1.0 is underfoot in everything else on these
pages: 1.0's **node-set** is gone, replaced by the **sequence**. Understanding
the sequence — and the `as=` type system that describes it — is what makes
`xsl:function`, [maps and arrays](maps-and-arrays.md), and
[higher-order functions](higher-order-functions.md) feel natural rather than
bolted on.

## What a sequence is

A sequence is an **ordered** list of **items**, where an item is either a *node*
or an *atomic value* (a string, number, boolean, date, …). That is the whole
model. Two consequences trip up people coming from 1.0:

- Sequences carry **order** — document order, or the order you built them in.
  Node-sets were unordered; sequences are not.
- Sequences are **flat**. There is no such thing as a sequence of sequences:
  `(1, (2, 3), 4)` *is* `(1, 2, 3, 4)`. Nesting only exists once you reach for
  [arrays](maps-and-arrays.md).

You build them with the comma operator, and the empty sequence is `()`:

``` xml
<xsl:variable name="primes" select="(2, 3, 5, 7)"/>
<xsl:variable name="prices" select="/catalog/cd/price"/>   <!-- a sequence of 3 nodes -->
<xsl:variable name="nothing" select="()"/>
```

A single value and a one-item sequence are the same thing — `42` and `(42)` are
indistinguishable. This is why a function can declare it returns *one* integer
and you use the result without unwrapping.

## Describing a sequence: `as=`

A **sequence type** names what is allowed: an *item type* plus an *occurrence
indicator*. You attach it with `as=` on
[`xsl:variable`](variables.md), [`xsl:param`](named-templates.md),
and [`xsl:function`](functions.md):

``` xml
<xsl:variable name="count"  as="xs:integer"   select="count(/catalog/cd)"/>
<xsl:variable name="titles" as="xs:string*"   select="/catalog/cd/title/string()"/>
<xsl:param    name="cd"     as="element(cd)"/>
```

The occurrence indicator on the end controls *how many*:

| Type | Means |
| --- | --- |
| `xs:integer` | exactly one integer |
| `xs:integer?` | zero or one |
| `xs:integer*` | zero or more |
| `xs:integer+` | one or more |

And the item type controls *what*:

| Item type | Matches |
| --- | --- |
| `xs:string`, `xs:decimal`, `xs:date`, … | a typed atomic value |
| `element(cd)` | an element named `cd` |
| `element()` / `attribute()` | any element / attribute |
| `node()` | any node |
| `item()` | anything — node or atomic |
| `function(*)`, `map(*)`, `array(*)` | a [function](higher-order-functions.md), [map, or array](maps-and-arrays.md) |

So `element(cd)+` is "one or more `cd` elements" and `item()*` is "any sequence
at all" — the implicit default when you omit `as=`.

!!! info "Types in HE vs schema-awareness (EE)"
    Everything above works in the free **Saxon-HE**: the built-in atomic types
    (`xs:integer`, `xs:date`, …), occurrence indicators, and element-name tests
    like `element(cd)`. What needs **Saxon-EE** is *schema-awareness* —
    validating input against an [XSD](../xsd/index.md) so elements carry their
    schema-defined types, plus the schema-aware tests `schema-element(cd)` /
    `schema-attribute(...)`. In HE, `cd/price` atomizes to an *untyped* value
    (effectively a string), which is exactly why the examples cast explicitly
    with `xs:decimal(...)` rather than relying on a schema.

!!! tip "Declare types — the errors move earlier"
    `as=` is optional, but adding it turns a class of silent
    wrong-answer bugs into **compile-time errors**. Declare a function returns
    `xs:integer` and accidentally return a string, and Saxon tells you at compile
    time instead of producing surprising output at run time. The discipline pays
    for itself on anything non-trivial.

## Atomization: nodes become values

When an operation wants a *value* but you hand it a *node*, the node is
**atomized** — its typed value is extracted. This is why comparing a node to a
number works:

``` xml
<!-- price is a node; 9.90 is a number; atomization bridges them -->
<xsl:if test="cd/price > 9.90"> … </xsl:if>
```

Atomization is also why `xs:decimal(cd/price)` is worth writing when you mean a
number: it pins the type down rather than leaving it as untyped atomic, so later
arithmetic and sorting behave.

## Returning items, not just text

In 1.0, [`xsl:value-of`](loops-and-output.md) was the only way out and it
produced **text**. 2.0/3.0 add `xsl:sequence`, which returns the *items
themselves* — nodes, atomics, [maps, functions](maps-and-arrays.md) — without
flattening them to a string:

``` xml
<xsl:variable name="expensive" as="element(cd)*">
  <xsl:sequence select="/catalog/cd[price > 9.90]"/>
</xsl:variable>
```

`$expensive` now holds actual `cd` *elements* you can navigate into
(`$expensive/title`), not a copy and not a string. `xsl:sequence` is the
workhorse for building and returning typed values, and it is what
[`xsl:function`](functions.md) uses to hand back its result.

## Testing and converting types

A handful of XPath operators work on sequence types directly:

``` xml
. instance of element(cd)        <!-- boolean: is the context an element named cd? -->
$x castable as xs:date           <!-- boolean: would the cast succeed? -->
$x cast as xs:integer            <!-- convert, or error if it can't -->
$x treat as element()+           <!-- assert the type without converting -->
```

`instance of` and `castable as` are the safe, test-first pair; `cast as` and
`treat as` commit and raise a [type error](error-handling.md) if they are wrong.

## Where to go next

- [User-defined functions](functions.md) — the first big payoff of typed sequences.
- [Maps and arrays](maps-and-arrays.md) — the structured item types `map(*)` and `array(*)`.
- [Higher-order functions](higher-order-functions.md) — when `function(*)` is the type you pass around.
