# Named templates and parameters

So far every template has fired by **matching** a node. A template can instead
be given a **name** and invoked deliberately, like a subroutine, with
`xsl:call-template`. This is how you package reusable logic — and, because XSLT
1.0 has no mutable loop counters, how you build **recursion**.

## A template you call by name

Add a `name` attribute and there is no `match` pattern to satisfy — the template
does nothing until something calls it explicitly.

``` xml title="named.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:template match="/">
  <report>
    <xsl:call-template name="banner"/>    <!-- (1)! -->
  </report>
</xsl:template>

<xsl:template name="banner">              <!-- (2)! -->
  <line>== Catalogue ==</line>
</xsl:template>

</xsl:stylesheet>
```

1.  `xsl:call-template` invokes the rule whose `name` matches. Unlike
    `apply-templates`, it names *exactly one* template — no pattern matching, no
    node selection.
2.  A `name`d template with no `match`. It will never fire on its own; only a
    `call-template` reaches it.

<div class="xslt-result" markdown>
```
<report><line>== Catalogue ==</line></report>
```
</div>

## Passing arguments: param and with-param

A named template becomes useful when it takes arguments. Declare them inside the
template with `xsl:param`; supply them at the call site with `xsl:with-param`.

``` xml title="greet.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:template match="/">
  <out>
    <xsl:call-template name="label">
      <xsl:with-param name="text" select="'Total'"/>   <!-- (1)! -->
      <xsl:with-param name="value" select="29.70"/>
    </xsl:call-template>
    <xsl:call-template name="label">
      <xsl:with-param name="text" select="'Subtotal'"/> <!-- (2)! -->
    </xsl:call-template>
  </out>
</xsl:template>

<xsl:template name="label">
  <xsl:param name="text"/>                  <!-- (3)! -->
  <xsl:param name="value" select="0"/>      <!-- (4)! -->
  <row><xsl:value-of select="$text"/>: <xsl:value-of select="$value"/></row>
</xsl:template>

</xsl:stylesheet>
```

1.  `with-param` binds an argument by name. The quoted `'Total'` is an XPath
    *string literal*; without the inner quotes `select="Total"` would mean
    "the child element named `Total`".
2.  The second call omits `value`, so its default applies.
3.  A required-looking param with no default behaves as the empty string if the
    caller leaves it out.
4.  A param with a default via `select`. If the caller passes `value`, that
    wins; otherwise it is `0`.

<div class="xslt-result" markdown>
```
<out><row>Total: 29.7</row><row>Subtotal: 0</row></out>
```
</div>

!!! note "Default by content, not just `select`"
    A default can also be written as element content, which is handy for markup:

    ``` xml
    <xsl:param name="value">0</xsl:param>
    ```

    Use `select` for an XPath expression (a number, a node-set, a string
    literal); use content when the default is literal text or markup.

!!! warning "Names must line up"
    `xsl:with-param name="x"` only reaches an `xsl:param name="x"` in the called
    template. A `with-param` with no matching `param` is silently ignored — a
    classic source of "my argument vanished".

## The crucial difference from apply-templates

This trips up almost everyone, so state it plainly:

!!! warning "`call-template` does **not** change the current node"
    `xsl:call-template` runs the named template **in the caller's context**. The
    current node, `position()`, and `last()` are all exactly what they were at
    the call site. `xsl:apply-templates`, by contrast, *moves* to each selected
    node — inside the matched template the current node **is** that node.

Watch the same `<cd>` data treated both ways:

``` xml title="context.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:template match="catalog">
  <xsl:apply-templates select="cd"/>       <!-- (1)! -->
</xsl:template>

<xsl:template match="cd">
  <pushed><xsl:value-of select="title"/></pushed>   <!-- (2)! -->
  <xsl:call-template name="here"/>                  <!-- (3)! -->
</xsl:template>

<xsl:template name="here">
  <called><xsl:value-of select="title"/></called>   <!-- (4)! -->
</xsl:template>

</xsl:stylesheet>
```

1.  `apply-templates` selects each `<cd>` and **moves** to it. Inside the `cd`
    template the current node is that `<cd>`.
2.  So `select="title"` resolves against the current `<cd>` — its own title.
3.  `call-template` does **not** move. The current node is *still* this `<cd>`.
4.  Therefore `select="title"` inside `here` resolves against that **same**
    `<cd>` — it inherited the context, it did not receive a node.

Run against the [catalog](index.md#the-running-example), every `here` call sees
the very node that called it:

<div class="xslt-result" markdown>
```
<pushed>Empire Burlesque</pushed><called>Empire Burlesque</called>
<pushed>Hide your heart</pushed><called>Hide your heart</called>
<pushed>Greatest Hits</pushed><called>Greatest Hits</called>
```
</div>

!!! tip "Need the call to act on a different node?"
    `call-template` has no `select`, so the only way to hand it a node is through
    a parameter: `<xsl:with-param name="cd" select="..."/>`, then operate on
    `$cd` inside. The current node still does not move — you just have a node-set
    in a variable.

## Recursion: the XSLT 1.0 loop

XSLT 1.0 has **no** loop with a mutable counter. `xsl:for-each` iterates over an
existing node-set, but you cannot say "do this N times" or "keep going until a
condition flips" by mutating a variable — variables are immutable. The idiom for
*all* such problems is **a named template that calls itself** with a changed
parameter.

Here is "repeat a string N times":

``` xml title="repeat.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:template match="/">
  <bar>
    <xsl:call-template name="repeat">
      <xsl:with-param name="text" select="'#'"/>
      <xsl:with-param name="count" select="5"/>
    </xsl:call-template>
  </bar>
</xsl:template>

<xsl:template name="repeat">
  <xsl:param name="text"/>
  <xsl:param name="count"/>
  <xsl:if test="$count &gt; 0">              <!-- (1)! -->
    <xsl:value-of select="$text"/>           <!-- (2)! -->
    <xsl:call-template name="repeat">        <!-- (3)! -->
      <xsl:with-param name="text" select="$text"/>
      <xsl:with-param name="count" select="$count - 1"/>
    </xsl:call-template>
  </xsl:if>
</xsl:template>

</xsl:stylesheet>
```

1.  The **base case**. When `count` reaches `0` the `xsl:if` body is skipped and
    the recursion stops. Every recursive template needs one, or it never
    terminates.
2.  Emit one copy of the string.
3.  The **recursive step**: call ourselves with `count` decremented. There is no
    assignment — the smaller value is passed as a fresh parameter binding.

<div class="xslt-result" markdown>
```
<bar>#####</bar>
```
</div>

The same shape walks a delimited list — peel off the head, recurse on the tail —
which is the standard way to tokenise a string in XSLT 1.0:

``` xml title="split.xsl" linenums="1"
<xsl:template name="sum-csv">
  <xsl:param name="list"/>
  <xsl:param name="acc" select="0"/>
  <xsl:choose>
    <xsl:when test="contains($list, ',')">                         <!-- (1)! -->
      <xsl:call-template name="sum-csv">
        <xsl:with-param name="list" select="substring-after($list, ',')"/>
        <xsl:with-param name="acc"
          select="$acc + number(substring-before($list, ','))"/>   <!-- (2)! -->
      </xsl:call-template>
    </xsl:when>
    <xsl:otherwise>
      <xsl:value-of select="$acc + number($list)"/>                <!-- (3)! -->
    </xsl:otherwise>
  </xsl:choose>
</xsl:template>
```

1.  While a comma remains, there is more than one item: take the part before it
    as the head, the part after it as the tail.
2.  Carry the running total in an accumulator parameter — the immutable-variable
    way to keep state across the recursion.
3.  Base case: no comma left, so `$list` is the final item; add it and emit the
    total. Calling with `list="3,4,5"` yields `12`.

!!! info "XSLT 2.0 has real alternatives"
    If your toolchain is XSLT 2.0/3.0 you also have `xsl:for-each` over computed
    sequences, the `tokenize()` function, and `xsl:iterate` — recursion is no
    longer the *only* option. In 1.0 it is, so it is worth getting comfortable
    with the head/tail pattern above.

## name and match can coexist

A template may carry **both** a `name` and a `match`. It then works two ways: it
fires by matching when `apply-templates` reaches a matching node, and it can also
be invoked directly by name.

``` xml title="both.xsl" linenums="1"
<xsl:template match="cd" name="cd-row">
  <row><xsl:value-of select="title"/></row>
</xsl:template>
```

- Reached by `<xsl:apply-templates select="cd"/>` → current node is the matched
  `<cd>`, and the template runs against it.
- Invoked by `<xsl:call-template name="cd-row"/>` → current node is **whatever
  the caller had**; if that is not a `<cd>`, `select="title"` simply finds
  nothing.

!!! tip "When to reach for each"
    Use `match` templates for **walking the document** (the push model). Use
    `name`d templates for **reusable, parameterised logic** and **recursion** —
    work that is not naturally "one template per node type".

## Next

Both `xsl:param` and `xsl:with-param` introduce *bindings* you read with `$name`.
[Variables](variables.md) covers `xsl:variable` — the third member of that family,
and why "variable" in XSLT does not mean what it does elsewhere.
