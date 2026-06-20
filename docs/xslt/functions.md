# User-defined functions

A [named template](named-templates.md) is an *instruction*: you reach it with
`xsl:call-template`, and it writes nodes to the result tree. An **`xsl:function`**
is different in kind — it returns a **value**, and you call it from inside *any
XPath expression*: in a `select`, in a predicate `[...]`, inside `value-of`,
even as an argument to another function. That makes functions the right tool
whenever you need a value to *compose* into a larger expression rather than a
chunk of output to emit.

This is XSLT 2.0/3.0 territory — `version="3.0"`, Saxon — so the typed
machinery below is fair game.

## A function callable from XPath

`xsl:function` lives at the top level of the stylesheet. Its `name` must be a
**namespaced** QName, so we bind a prefix first.

``` xml title="fn-basic.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="3.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
    xmlns:my="http://example.org/fn">          <!-- (1)! -->

<xsl:function name="my:with-tax" as="xs:decimal">   <!-- (2)! -->
  <xsl:param name="net"  as="xs:decimal"/>          <!-- (3)! -->
  <xsl:param name="rate" as="xs:decimal"/>
  <xsl:sequence select="$net * (1 + $rate)"/>       <!-- (4)! -->
</xsl:function>

<xsl:template match="/">
  <gross><xsl:value-of select="my:with-tax(20.00, 0.25)"/></gross>  <!-- (5)! -->
</xsl:template>

</xsl:stylesheet>
```

1.  Bind a prefix to a namespace you control. User function names live here —
    never in the XSLT or XPath built-in namespaces.
2.  `as=` declares the **return type**. The function promises an `xs:decimal`.
3.  Each `xsl:param` carries its own `as=` type. Arguments are matched **by
    position**, not by name (unlike `xsl:with-param`).
4.  The body usually ends in `xsl:sequence`, which returns the computed value
    rather than emitting a text node.
5.  And here is the point: the function is called straight inside an XPath
    expression. No `call-template`, no `with-param` — just `my:with-tax(...)`.

<div class="xslt-result" markdown>
```
<gross>25</gross>
```
</div>

!!! warning "The name must be namespaced"
    `<xsl:function name="with-tax">` is a static error. A user function name
    **must** carry a prefix bound to a non-null namespace (here
    `http://example.org/fn`). This keeps your names from colliding with the
    built-in `fn:` library.

## Calling a function inside select and predicates

The real advantage shows when the function is woven into a bigger expression.
Given this catalogue:

``` xml title="catalog.xml" linenums="1"
<catalog>
  <cd><title>Quiet Harbour</title><artist>The Lanterns</artist><price>12.00</price></cd>
  <cd><title>Open Road</title><artist>Mara Vale</artist><price>9.50</price></cd>
  <cd><title>Slow Tide</title><artist>The Lanterns</artist><price>15.00</price></cd>
</catalog>
```

A function that builds a label, plus one that computes a gross price, used in a
`select`, a predicate, and a sort:

``` xml title="fn-catalog.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="3.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:xs="http://www.w3.org/2001/XMLSchema"
    xmlns:my="http://example.org/fn">

<xsl:function name="my:full-label" as="xs:string">     <!-- (1)! -->
  <xsl:param name="cd" as="element(cd)"/>
  <xsl:sequence select="$cd/artist || ' — ' || $cd/title"/>  <!-- (2)! -->
</xsl:function>

<xsl:function name="my:with-tax" as="xs:decimal">
  <xsl:param name="net"  as="xs:decimal"/>
  <xsl:param name="rate" as="xs:decimal"/>
  <xsl:sequence select="$net * (1 + $rate)"/>
</xsl:function>

<xsl:template match="/">
  <report>
    <!-- function in a predicate: keep only the dearer discs -->
    <xsl:for-each select="catalog/cd[my:with-tax(price, 0.25) gt 15]">  <!-- (3)! -->
      <xsl:sort select="my:full-label(.)"/>                            <!-- (4)! -->
      <item label="{my:full-label(.)}"                                 <!-- (5)! -->
            gross="{my:with-tax(price, 0.25)}"/>
    </xsl:for-each>
  </report>
</xsl:template>

</xsl:stylesheet>
```

1.  This function takes an **element** (`element(cd)`) and returns a string.
    Functions can accept and return any XPath type, including nodes.
2.  `||` is the XPath 3.0 string-concatenation operator. The element references
    `$cd/artist` are atomised to their string values here.
3.  A function **inside a predicate**. A named template could never appear here
    — predicates are pure XPath, and only value-returning functions fit.
4.  The same function drives the `xsl:sort` key.
5.  And again inside attribute value templates. One definition, many call sites.

<div class="xslt-result" markdown>
```
<report>
   <item label="Mara Vale — Open Road" gross="11.875"/>
   <item label="The Lanterns — Slow Tide" gross="18.75"/>
</report>
```
</div>

!!! note "`12.00` is filtered out"
    `my:with-tax(12.00, 0.25)` is `15`, which is **not** `gt 15`, so *Quiet
    Harbour* drops out. *Open Road* (`9.50 → 11.875`) and *Slow Tide*
    (`15.00 → 18.75`) survive, then sort by their label.

## Functions versus named templates

| | `xsl:function` | `xsl:template name="..."` |
|---|---|---|
| Returns | a **value** (any sequence) | writes to the **result tree** |
| Called from | any XPath expression | only via `xsl:call-template` |
| In a predicate? | yes | no |
| Arguments | positional, typed | by name, with `xsl:with-param` |
| Built for | pure computation, composition | emitting output, the push model |

In short: if you want a result you can drop into a `select`, a predicate, or a
sort key, write a **function**. If you want to *produce a block of output* —
especially recursive output that builds nodes — reach for a
[named template](named-templates.md). The two complement each other: a function
computes the value, a template lays it out.

## Recursion and a peek at higher-order functions

Functions may **call themselves**, which is the classic way to fold over a
sequence without a mutable counter:

``` xml title="fn-sum.xsl" linenums="1"
<xsl:function name="my:running-total" as="xs:decimal">
  <xsl:param name="prices" as="xs:decimal*"/>          <!-- (1)! -->
  <xsl:sequence select="
    if (empty($prices)) then 0
    else $prices[1] + my:running-total($prices[position() gt 1])"/>  <!-- (2)! -->
</xsl:function>
```

1.  `xs:decimal*` — the `*` means *zero or more*. Functions accept sequences,
    not just single values.
2.  The recursive call peels off the first item and sums the rest.

!!! tip "In 3.0 you rarely need to hand-roll recursion"
    XPath 3.0 has **inline anonymous functions** —
    `function($x as xs:decimal) { $x * 1.25 }` — and **higher-order**
    functions that take them as arguments, like `fold-left(...)`,
    `for-each(...)` and `filter(...)`. The whole `my:running-total` above is
    just `fold-left($prices, 0, function($a, $b) { $a + $b })`. A topic for
    another day, but worth knowing it exists.

## Next

Filtering by a computed value naturally leads to **collecting** related items.
[Grouping](grouping.md) introduces `xsl:for-each-group` — XSLT 2.0's answer to
turning a flat list into nested structure.
