# XPath predicates

A **predicate** is a `[...]` filter attached to a location step. The step first
selects a set of nodes; the predicate then keeps only those for which the
bracketed expression is true. You can read `cd[price > 10]` as "the `cd`
elements that have a `price` child greater than 10".

For these examples the [catalog](index.md#the-running-example) gains a `genre`
attribute, and the first CD keeps its optional `<year>` child while the others
omit it:

``` xml title="catalog.xml" linenums="1"
<catalog>
  <cd genre="rock"><title>Empire Burlesque</title><artist>Bob Dylan</artist><price>10.90</price><year>1985</year></cd>
  <cd genre="pop"><title>Hide your heart</title><artist>Bonnie Tyler</artist><price>9.90</price></cd>
  <cd genre="country"><title>Greatest Hits</title><artist>Dolly Parton</artist><price>9.90</price></cd>
</catalog>
```

## Boolean and comparison predicates

The expression inside `[...]` is evaluated relative to each candidate node and
coerced to a boolean. Comparisons are the common case:

``` xml
<xsl:for-each select="catalog/cd[price &gt; 10]">     <!-- (1)! -->
  <p><xsl:value-of select="title"/></p>
</xsl:for-each>
```

1.  Only CDs whose `<price>` child is numerically greater than 10.

<div class="xslt-result" markdown>
Empire Burlesque
</div>

String equality works the same way. `catalog/cd[artist = 'Bob Dylan']` keeps the
CDs whose `<artist>` child equals that string:

<div class="xslt-result" markdown>
Empire Burlesque
</div>

!!! warning "Escaping `<` in predicates"
    A stylesheet is XML, so `<` must be written `&lt;` inside a predicate:
    `cd[price &lt; 10]`. A literal `>` is legal, but it is conventional to
    write `&gt;`. Single quotes around string literals avoid clashing with the
    double quotes of the surrounding `select="..."`.

## Existence predicates

A bare child or attribute name is true when that node exists, because XPath
treats a non-empty node-set as `true`. This is the standard way to filter on
optional content.

``` xml
<xsl:for-each select="catalog/cd[year]">     <!-- (1)! -->
  <p><xsl:value-of select="title"/></p>
</xsl:for-each>
```

1.  Keep only CDs that have a `<year>` child.

<div class="xslt-result" markdown>
Empire Burlesque
</div>

Wrap the name in `not(...)` to invert it. `catalog/cd[not(year)]` selects the
CDs that have **no** `<year>` child:

<div class="xslt-result" markdown>
Hide your heart

Greatest Hits
</div>

## Attribute predicates

Attributes are addressed with the `@` prefix, and behave exactly like child
elements in a predicate — test their value, or just their existence.

``` xml
<xsl:value-of select="catalog/cd[@genre = 'pop']/title"/>   <!-- (1)! -->
```

1.  The title of the CD whose `genre` attribute is `pop`.

<div class="xslt-result" markdown>
Hide your heart
</div>

`catalog/cd[@genre]` would match every CD here, since all three carry a `genre`
attribute; `catalog/cd[@id]` would match none, since no CD has an `id`.

## Positional predicates

A numeric predicate selects by **position** within the current node-set.
Positions are **1-based**, and `[1]` is simply shorthand for
`[position() = 1]`.

| Predicate | Meaning |
| --- | --- |
| `cd[1]` | the first `cd` (same as `cd[position() = 1]`) |
| `cd[last()]` | the last `cd` |
| `cd[position() &lt;= 2]` | the first two `cd` elements |

``` xml
<xsl:value-of select="catalog/cd[1]/title"/>
<xsl:value-of select="catalog/cd[last()]/title"/>
```

<div class="xslt-result" markdown>
Empire Burlesque

Greatest Hits
</div>

`position()` and `last()` are also useful arithmetically. A common trick is
**row striping** — shading every even row — with the `mod` operator:

``` xml
<xsl:for-each select="catalog/cd">
  <tr>
    <xsl:if test="position() mod 2 = 0">          <!-- (1)! -->
      <xsl:attribute name="bgcolor">#eeeeee</xsl:attribute>
    </xsl:if>
    <td><xsl:value-of select="title"/></td>
  </tr>
</xsl:for-each>
```

1.  True for positions 2, 4, 6, … — i.e. every other row.

The selector `catalog/cd[position() mod 2 = 0]` likewise keeps just the
even-positioned CDs.

## Chaining predicates

Multiple predicates apply **left to right**, and each one re-numbers the
node-set it receives. Order therefore matters:

| Expression | Reading |
| --- | --- |
| `cd[price &gt; 9][1]` | filter to CDs over 9, **then** take the first of those |
| `cd[1][price &gt; 9]` | take the first CD, **then** keep it only if its price > 9 |

``` xml
<xsl:value-of select="catalog/cd[price &gt; 9][1]/title"/>    <!-- (1)! -->
```

1.  All three CDs exceed 9, so this is the first of them.

<div class="xslt-result" markdown>
Empire Burlesque
</div>

Here both expressions happen to yield *Empire Burlesque*, but they would differ
if the first CD were the cheap one: `cd[1][price > 9]` could then select
*nothing*, while `cd[price > 9][1]` would still return the first qualifying CD.

## The `//cd[1]` gotcha

!!! warning "`//cd[1]` is not the first `cd` in the document"
    A predicate binds to its location step, not to the whole path. In
    `//cd[1]`, the `[1]` applies to the `cd` step, selecting **every `cd` that
    is the first `cd` among its own siblings** — potentially many nodes spread
    across the document. To get the single global first node, parenthesise the
    path so the predicate applies to the *combined* result:

    ``` xml
    (//cd)[1]      <!-- the one first cd in the whole document -->
    //cd[1]        <!-- each cd that is its parent's first cd -->
    ```

In this flat catalog the two expressions coincide, but as soon as `cd` elements
appear under more than one parent they diverge. When you mean "the first match
anywhere", reach for `(//cd)[1]`.

## Predicates in match patterns

Everything above applies wherever an XPath expression is used — the `select` of
`xsl:for-each`, `xsl:apply-templates`, and `xsl:value-of`, and also the `match`
pattern of a template. A predicate in a `match` pattern restricts which nodes
the template fires for:

``` xml
<xsl:template match="cd[price &gt; 10]">      <!-- (1)! -->
  <p class="premium"><xsl:value-of select="title"/></p>
</xsl:template>

<xsl:template match="cd">                    <!-- (2)! -->
  <p><xsl:value-of select="title"/></p>
</xsl:template>
```

1.  Fires only for CDs over 10; the more specific pattern wins.
2.  The fallback for every other `cd`.

<div class="xslt-result" markdown>
Empire Burlesque  *(premium)*

Hide your heart

Greatest Hits
</div>

## Summary

| Form | Selects |
| --- | --- |
| `cd[price &gt; 10]` | CDs whose price compares true |
| `cd[artist = 'Bob Dylan']` | CDs with that exact artist string |
| `cd[year]` | CDs that have a `<year>` child |
| `cd[not(discount)]` | CDs with no `<discount>` child |
| `item[@type = 'book']` | items whose `type` attribute equals `book` |
| `item[@id]` | items that have an `id` attribute |
| `cd[1]` | the first CD (`= [position() = 1]`, 1-based) |
| `cd[last()]` | the last CD |
| `cd[position() &lt;= 2]` | the first two CDs |
| `cd[position() mod 2 = 0]` | even-positioned CDs (striping) |
| `cd[price &gt; 9][1]` | first CD among those over 9 |
| `(//cd)[1]` | the single first CD in the whole document |

## Next

Predicates often test text, and real filters need to trim, match, and compare
strings precisely — see [String functions](strings.md).
