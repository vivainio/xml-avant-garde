# Conditionals

XSLT has two conditional instructions: `xsl:if` for a single yes/no test, and
`xsl:choose` for picking one branch out of several.

## xsl:if

`xsl:if` outputs its body only when the `test` expression is true. There is no
`else` — for that you need `xsl:choose`.

``` xml
<xsl:for-each select="catalog/cd">
  <p>
    <xsl:value-of select="title"/>
    <xsl:if test="price &gt; 10">
      <span style="color:red"> (premium)</span>
    </xsl:if>
  </p>
</xsl:for-each>
```

!!! warning "Escaping `<` and `>` in tests"
    A stylesheet is XML, so a literal `>` is fine but `<` must be written
    `&lt;` and, by convention, `>` is often written `&gt;`. So "price less than
    10" is `test="price &lt; 10"`.

## xsl:choose / when / otherwise

`xsl:choose` is XSLT's `switch`/`if-else-if`. The processor takes the **first**
`xsl:when` whose test is true; if none match, it takes `xsl:otherwise` (which is
optional).

``` xml title="cdcatalog-choose.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
<xsl:template match="/">
<html>
<body>
  <h2>My CD Collection</h2>
  <table border="1">
    <tr bgcolor="#9acd32">
      <th>Title</th>
      <th>Artist</th>
    </tr>
    <xsl:for-each select="catalog/cd">
    <tr>
      <td><xsl:value-of select="title"/></td>
      <xsl:choose>
        <xsl:when test="price &gt; 10">                <!-- (1)! -->
          <td bgcolor="#ff00ff"><xsl:value-of select="artist"/></td>
        </xsl:when>
        <xsl:otherwise>                                <!-- (2)! -->
          <td><xsl:value-of select="artist"/></td>
        </xsl:otherwise>
      </xsl:choose>
    </tr>
    </xsl:for-each>
  </table>
</body>
</html>
</xsl:template>
</xsl:stylesheet>
```

1.  CDs priced over 10 get a highlighted cell…
2.  …everything else falls through to the default cell.

Against the [catalog](index.md#the-running-example), only *Empire Burlesque*
(10.90) is over 10, so its artist cell is highlighted:

<div class="xslt-result" markdown>
| Title | Artist |
| --- | --- |
| Empire Burlesque | :material-square:{ style="color:#ff00ff" } Bob Dylan |
| Hide your heart | Bonnie Tyler |
| Greatest Hits | Dolly Parton |
</div>

## Writing good tests

The `test` is an XPath expression. XPath coerces the result to a boolean:

| Expression | True when |
| --- | --- |
| `price &gt; 10` | numeric comparison holds |
| `artist = 'Bob Dylan'` | string equality holds |
| `year` | a `<year>` child exists (non-empty node-set) |
| `not(@discontinued)` | the `discontinued` attribute is absent |
| `position() mod 2 = 0` | current node is at an even position |

That last `node exists` idiom is the usual way to guard optional content:
`<xsl:if test="notes">…</xsl:if>`.

## Next

Tests let you change *what* you output. [XPath predicates](predicates.md) push
that filtering into the node selection itself.
