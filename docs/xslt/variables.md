# Variables

A variable gives a name to a value so you can reuse it without recomputing the
XPath every time. XSLT variables look familiar to programmers from other
languages, but they have one rule that surprises almost everyone: once bound, a
variable **never changes**.

## Two ways to declare a variable

There are two forms of `xsl:variable`, and the difference is what the variable
ends up holding.

The **value form** uses a `select` attribute. The XPath is evaluated and the
result — a string, number, boolean, or node-set — becomes the variable's value.

``` xml
<xsl:variable name="count" select="count(catalog/cd)"/>     <!-- a number   -->
<xsl:variable name="total" select="sum(catalog/cd/price)"/> <!-- a number   -->
<xsl:variable name="cds"   select="catalog/cd"/>            <!-- a node-set -->
<xsl:variable name="label" select="'CD Collection'"/>       <!-- a string   -->
```

!!! tip "Quote your string literals"
    `select="'CD Collection'"` has inner single quotes. Without them,
    `select="CD Collection"` is an XPath *location path* that looks for child
    elements named `CD` and `Collection` — almost never what you mean.

The **content form** has no `select`; instead the body of the element becomes
the value. The body is evaluated as a template and the result is captured as a
**result tree fragment** — a little tree of nodes you can drop into the output.

``` xml
<xsl:variable name="heading">
  <h2>My CD Collection</h2>
  <p>A small worked example.</p>
</xsl:variable>
```

Use the value form when you want a computed scalar or a selection of existing
nodes. Use the content form when you want to *build* some markup or assemble
text from several pieces.

!!! note "Result tree fragments are limited in XSLT 1.0"
    A result tree fragment can be copied into the output, but in XSLT 1.0 you
    cannot apply further XPath location steps to it (some processors offer a
    `node-set()` extension for that). If you only need to select existing input
    nodes, prefer the value form with `select`. XSLT 2.0+ removes this
    restriction.

## Referencing a variable

Wherever a value is allowed you reference the variable with a `$` prefix, both
in `xsl:value-of`/`select` and inside attribute value templates:

``` xml title="catalog-summary.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
<xsl:template match="/">
  <xsl:variable name="count" select="count(catalog/cd)"/>   <!-- (1)! -->
  <xsl:variable name="total" select="sum(catalog/cd/price)"/>
  <html>
  <body>
    <p>The catalog holds <xsl:value-of select="$count"/> CDs            <!-- (2)! -->
       worth <xsl:value-of select="$total"/> in total.</p>
    <p>Average price: <xsl:value-of select="$total div $count"/>.</p>   <!-- (3)! -->
  </body>
  </html>
</xsl:template>
</xsl:stylesheet>
```

1.  Capture two scalars once, up front.
2.  Reference them with `$count` and `$total`.
3.  Variables can be combined in further XPath expressions.

Against the [catalog](index.md#the-running-example) this produces:

<div class="xslt-result" markdown>
The catalog holds 3 CDs worth 30.7 in total.

Average price: 10.233333333333333.
</div>

## What a variable can hold

| Form | Example | Value type |
| --- | --- | --- |
| `select="count(...)"` | `count(catalog/cd)` | number |
| `select="sum(...)"` | `sum(catalog/cd/price)` | number |
| `select="'text'"` | `'CD Collection'` | string |
| `select="price &gt; 10"` | a comparison | boolean |
| `select="catalog/cd"` | a location path | node-set |
| content form | `<h2>…</h2>` body | result tree fragment |

## Immutability — the rule that trips everyone up

!!! warning "A variable is bound once and can never be reassigned"
    There is no `x = x + 1` in XSLT. Once `<xsl:variable name="x" select="0"/>`
    runs, `$x` is `0` for the rest of its scope — permanently. This is the
    single most important thing to internalise coming from an imperative
    language. The following does **not** accumulate a running total:

    ``` xml
    <!-- WRONG: this does not work in XSLT -->
    <xsl:variable name="sum" select="0"/>
    <xsl:for-each select="catalog/cd">
      <xsl:variable name="sum" select="$sum + price"/>  <!-- a NEW $sum, local to this iteration -->
    </xsl:for-each>
    <!-- the outer $sum is still 0 here -->
    ```

    Each `xsl:variable` inside the loop creates a *fresh* variable scoped to
    that one iteration; it shadows the outer one and then vanishes. The outer
    `$sum` is untouched. Nothing is being added up.

So how *do* you compute a running total? Two idioms.

**(a) Let XPath aggregate.** For sums, counts, minima and the like, an XPath
expression does the whole job at once — no loop, no accumulator:

``` xml
<xsl:variable name="total" select="sum(catalog/cd/price)"/>
```

<div class="xslt-result" markdown>
total = 30.7
</div>

**(b) Recursion.** When there is no built-in aggregate (e.g. a running product,
or a custom fold), the loop becomes a recursive [named
template](named-templates.md) that passes the running value along as a
parameter. The "mutation" is expressed as *a new call with a new value*:

``` xml
<xsl:template name="running-total">
  <xsl:param name="nodes"/>
  <xsl:param name="acc" select="0"/>          <!-- the value carried so far -->
  <xsl:choose>
    <xsl:when test="$nodes">
      <xsl:call-template name="running-total">
        <xsl:with-param name="nodes" select="$nodes[position() &gt; 1]"/>
        <xsl:with-param name="acc"   select="$acc + $nodes[1]/price"/>
      </xsl:call-template>
    </xsl:when>
    <xsl:otherwise>
      <xsl:value-of select="$acc"/>           <!-- base case: emit the result -->
    </xsl:otherwise>
  </xsl:choose>
</xsl:template>
```

Each call binds a brand-new `$acc`; nothing is ever reassigned.

## Scope

Where a variable is visible depends on where you declare it.

A **local** variable (declared inside a template or other block) is visible to
its *following siblings and their descendants* in that same block — not before
it, and not outside the block:

``` xml
<xsl:template match="/">
  <xsl:variable name="count" select="count(catalog/cd)"/>
  <p><xsl:value-of select="$count"/></p>   <!-- OK: follows the declaration -->
  <xsl:for-each select="catalog/cd">
    <p><xsl:value-of select="$count"/></p> <!-- OK: descendant of a following sibling -->
  </xsl:for-each>
</xsl:template>
<!-- $count is NOT visible in any other template -->
```

A **global** variable is declared as a direct child of `xsl:stylesheet`. It is
visible everywhere — in every template:

``` xml title="global-variable.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:variable name="title" select="'My CD Collection'"/>   <!-- (1)! -->

<xsl:template match="/">
  <html><body>
    <h2><xsl:value-of select="$title"/></h2>               <!-- (2)! -->
  </body></html>
</xsl:template>

</xsl:stylesheet>
```

1.  Top-level: a child of `xsl:stylesheet`, so it is global.
2.  Any template may reference `$title`.

!!! info "A local declaration can shadow a global one"
    If a local variable has the same name as a global one, the local binding
    wins for the rest of its block. This is shadowing, not reassignment — the
    global variable is unchanged everywhere else.

## Variables vs. parameters

`xsl:param` looks almost identical to `xsl:variable` and obeys the same
immutability rule, but with one key difference: a parameter's value can be
*supplied from outside*. A caller can override it.

``` xml
<xsl:variable name="rate" select="0.2"/>   <!-- fixed: always 0.2          -->
<xsl:param    name="rate" select="0.2"/>   <!-- a default: caller may override -->
```

- A **global** `xsl:param` can be set by the calling application when the
  transformation is run (a stylesheet input).
- A **template** `xsl:param` receives values via `xsl:with-param` when the
  template is called or applied.
- A variable has no such hook — its value is whatever its `select`/content says,
  full stop.

Rule of thumb: use `xsl:variable` for a value the stylesheet computes for
itself; use `xsl:param` for a value someone else gets to choose.

## Next

Variables capture values; [loops and output](loops-and-output.md) put those
values to work by iterating over your data and writing the result.
