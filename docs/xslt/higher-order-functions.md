---
icon: lucide/sigma
---

# Higher-order functions

In 3.0 a **function is a value**. You can put one in a variable, pass it to
another function, and return it as a result — exactly as you would a string or a
node. A function that takes or returns another function is a **higher-order
function** (HOF), and a small library of them replaces a lot of hand-written
[`xsl:for-each`](loops-and-output.md) plumbing.

## Three ways to get a function value

``` xml
<!-- 1. a named reference: name#arity -->
<xsl:variable name="upper" select="upper-case#1"/>

<!-- 2. an inline (anonymous) function -->
<xsl:variable name="vat" select="function($p as xs:decimal) as xs:decimal { $p * 1.24 }"/>

<!-- 3. a reference to your own xsl:function -->
<xsl:variable name="label" select="local:label#1"/>
```

The `#1` / `#2` is the **arity** — how many arguments — because the same name
can have several signatures (`concat#2` vs `concat#3`). You call any of them by
applying it to arguments:

``` xml
<xsl:value-of select="$upper('hello')"/>   <!-- HELLO -->
<xsl:value-of select="$vat(10.90)"/>        <!-- 13.516 -->
```

The matching [sequence type](sequences-and-types.md) is `function(*)` for "any
function," or a precise signature like `function(xs:decimal) as xs:decimal`.

## The HOF library

The point of function values is to hand them to the built-in combinators, so you
say *what* to do, not *how* to loop:

| Function | Does |
| --- | --- |
| `for-each($seq, $f)` | apply `$f` to every item — a `map`/`select` |
| `filter($seq, $pred)` | keep items where `$pred` is true |
| `fold-left($seq, $init, $f)` | reduce left-to-right to a single value |
| `fold-right($seq, $init, $f)` | reduce right-to-left |
| `for-each-pair($a, $b, $f)` | zip two sequences with `$f` |
| `sort($seq, (), $key)` | sort by a key function (3.1) |
| `apply($f, $args)` | call `$f` with an array of arguments |

``` xml
<!-- total catalogue value, no loop and no accumulator variable -->
<xsl:variable name="prices" as="xs:decimal*" select="/catalog/cd/xs:decimal(price)"/>
<xsl:value-of select="fold-left($prices, 0.0, function($acc, $p) { $acc + $p })"/>
```

`fold-left` walks `$prices`, threading a running total through the inline
function — the functional equivalent of a loop with an accumulator, in one
expression.

## The arrow operator `=>`

`=>` chains calls left to right, feeding each result in as the **first**
argument of the next — the readable alternative to deeply nested parentheses:

``` xml
<!-- nested: -->
<xsl:value-of select="string-join(sort(distinct-values(/catalog/cd/@genre)), ', ')"/>

<!-- the same, as a pipeline: -->
<xsl:value-of select="/catalog/cd/@genre => distinct-values() => sort() => string-join(', ')"/>
```

Both produce `country, pop, rock`, but the second reads in the order the data
actually flows.

## A worked example: sort by a computed key

`sort#3` takes a *key function*, so you can order by something the elements
don't store directly — here, title length:

``` xml
<xsl:variable name="byTitleLength"
  select="sort(/catalog/cd, (), function($cd) { string-length($cd/title) })"/>

<xsl:for-each select="$byTitleLength">
  <xsl:value-of select="title"/><xsl:text>&#10;</xsl:text>
</xsl:for-each>
```

The middle `()` is the optional collation; the third argument is the function
that produces the sort key for each item. This is the kind of thing that needed
an awkward `xsl:sort select=` dance — or was impossible — in 1.0.

!!! note "Partial application and closures"
    An inline function *captures* the variables in scope where it is written.
    That lets you build specialised functions on the fly — a "make a discounter"
    that returns a function:

    ``` xml
    <xsl:variable name="discount"
      select="function($rate) { function($p) { $p * (1 - $rate) } }"/>
    <xsl:value-of select="$discount(0.1)(10.90)"/>   <!-- 9.81 -->
    ```

## Where to go next

- [Maps and arrays](maps-and-arrays.md) — function values plus structured data, the 3.1 pairing.
- [Sequences and types](sequences-and-types.md) — the `function(*)` item type these rely on.
- [User-defined functions](functions.md) — `xsl:function`, the named counterpart.
