---
icon: lucide/wand-2
---

# Dynamic XPath with `xsl:evaluate`

Every XPath you have written so far is **fixed** at the time you write the
stylesheet. `xsl:evaluate` (3.0) lifts that: it takes an XPath expression held in
a **string** — read from config, a mapping table, or user input — and evaluates
it at run time. It is the reflection escape hatch of XSLT, and there is nothing
like it in 1.0.

## The basics

`xsl:evaluate` takes the expression in `xpath=` and the node to evaluate it
against in `context-item=`:

``` xml
<xsl:variable name="expr" select="'title'"/>
<xsl:evaluate xpath="$expr" context-item="/catalog/cd[1]"/>
<!-- evaluates  title  against the first cd → Empire Burlesque -->
```

The expression string can be *any* XPath — paths, function calls, arithmetic —
decided while the transform runs.

## The motivating case: data-driven extraction

The point of `xsl:evaluate` is a transform whose *behaviour* lives in data. Here
a small table maps output column names to the XPath that fills them; adding a
column means editing the table, not the code:

``` xml
<xsl:variable name="columns">
  <col name="Title"  path="title"/>
  <col name="Artist" path="artist"/>
  <col name="VAT"    path="xs:decimal(price) * 0.24"/>
</xsl:variable>

<xsl:template match="cd">
  <row>
    <xsl:variable name="cd" select="."/>
    <xsl:for-each select="$columns/col">
      <cell name="{@name}">
        <xsl:evaluate xpath="@path" context-item="$cd"/>
      </cell>
    </xsl:for-each>
  </row>
</xsl:template>
```

The `VAT` row shows the power: an arbitrary *computed* expression, not just a
field name, supplied entirely as data.

## Passing variables and fixing the result type

The dynamic expression runs in its own scope, so variables it references must be
passed in explicitly with `xsl:with-param` (these become `$name` inside the
expression). `as=` constrains the [result type](sequences-and-types.md), and
`namespace-context=` supplies in-scope namespaces:

``` xml
<xsl:evaluate xpath="'$rate * count(cd)'"
              context-item="/catalog"
              as="xs:decimal">
  <xsl:with-param name="rate" select="0.24"/>
</xsl:evaluate>
```

## Caveats — power has a price

!!! warning "Security, performance, and availability"
    - **Untrusted input.** Evaluating an expression string from outside the
      stylesheet is code injection's cousin — a hostile `xpath` could read other
      documents or loop expensively. Only evaluate expressions you trust, or
      sanitise them.
    - **It can be switched off.** Because of the above, a processor may *disable*
      dynamic evaluation by configuration. Test
      `system-property('xsl:supports-dynamic-evaluation')` (`yes`/`no`) before
      relying on it, or put an [`xsl:fallback`](error-handling.md) inside the
      `xsl:evaluate` for when the feature is unavailable.
    - **No static analysis.** The optimiser cannot see inside a string, so an
      `xsl:evaluate` expression is never [streamable](streaming.md) and misses
      compile-time type and path checking. Reach for it only when a *fixed* XPath
      genuinely cannot express the requirement — most "dynamic" needs are better
      met by [higher-order functions](higher-order-functions.md) or a
      [map](maps-and-arrays.md) of pre-written functions.

## Where to go next

- [Higher-order functions](higher-order-functions.md) — the safer, statically-checked way to parameterise behaviour.
- [Maps and arrays](maps-and-arrays.md) — a lookup table of functions, often a better fit than evaluating strings.
- [Sequences and types](sequences-and-types.md) — the `as=` constraint on the result.
