# Sorting

By default, XSLT processes nodes in **document order** — the order they appear
in the source. `xsl:sort` overrides that, reordering the nodes a `for-each` or
`apply-templates` is about to process.

## xsl:sort

Place `xsl:sort` as the **first child** of the loop. It does not output
anything itself; it reorders the iteration.

``` xml linenums="1"
<xsl:for-each select="catalog/cd">
  <xsl:sort select="artist"/>          <!-- (1)! -->
  <p>
    <xsl:value-of select="artist"/> — <xsl:value-of select="title"/>
  </p>
</xsl:for-each>
```

1.  Sort the CDs by their `<artist>` child before iterating.

Sorting the [catalog](index.md#the-running-example) by artist:

<div class="xslt-result" markdown>
Bob Dylan — Empire Burlesque

Bonnie Tyler — Hide your heart

Dolly Parton — Greatest Hits
</div>

## Controlling the sort

`xsl:sort` takes several attributes:

| Attribute | Values | Effect |
| --- | --- | --- |
| `select` | XPath | The key to sort on (default: the node's string value) |
| `order` | `ascending` (default), `descending` | Direction |
| `data-type` | `text` (default), `number` | Compare as strings or numerically |
| `case-order` | `upper-first`, `lower-first` | Tie-break for mixed case |

The `data-type` distinction matters. As **text**, `"9.90"` sorts *after*
`"10.90"` because `"9"` &gt; `"1"` character-wise. To order by price correctly,
sort numerically:

``` xml
<xsl:sort select="price" data-type="number" order="descending"/>
```

<div class="xslt-result" markdown>
Empire Burlesque — 10.90

Hide your heart — 9.90

Greatest Hits — 9.90
</div>

## Multiple keys

Several `xsl:sort` elements act as successive tie-breakers, in order. This sorts
by price descending, then by title for CDs at the same price:

``` xml
<xsl:for-each select="catalog/cd">
  <xsl:sort select="price" data-type="number" order="descending"/>
  <xsl:sort select="title"/>
  ...
</xsl:for-each>
```

The same `xsl:sort` works inside `xsl:apply-templates` when you let the
processor drive iteration:

``` xml
<xsl:apply-templates select="catalog/cd">
  <xsl:sort select="artist"/>
</xsl:apply-templates>
```

## Where next

That covers the core of XSLT 1.0: templates, iteration, output, conditionals,
and sorting. From here the natural directions are **named templates and
parameters** for reuse, **keys** (`xsl:key`) for fast cross-references, and
moving to **XSLT 2.0/3.0** for grouping, regular expressions, and richer
functions — topics this tutorial will grow into.
