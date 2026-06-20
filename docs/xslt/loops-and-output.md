# Loops and output

Two instructions do most of the day-to-day work in a stylesheet:
`xsl:for-each` to iterate, and `xsl:value-of` to write text.

## xsl:value-of

`xsl:value-of` evaluates an XPath expression and inserts its **string value**
into the result tree.

``` xml
<xsl:value-of select="title"/>     <!-- text of the child <title>      -->
<xsl:value-of select="."/>         <!-- text of the current node       -->
<xsl:value-of select="@id"/>       <!-- value of the id attribute       -->
<xsl:value-of select="price * 2"/> <!-- XPath can compute, too          -->
```

If the expression selects several nodes, only the **first** is output. To emit
all of them, loop.

## xsl:for-each

`xsl:for-each` processes every node in a selected set, running its body once per
node. Inside the loop the *current node* is the node being processed, so
relative paths like `title` resolve against it.

``` xml title="breakfast-style loop" linenums="1"
<xsl:for-each select="catalog/cd">
  <div>
    <span style="font-weight:bold"><xsl:value-of select="title"/></span>
    — <xsl:value-of select="artist"/>
    (<xsl:value-of select="price"/>)
  </div>
</xsl:for-each>
```

Run against the [catalog](index.md#the-running-example) this produces one `div`
per CD:

<div class="xslt-result" markdown>
**Empire Burlesque** — Bob Dylan (10.90)

**Hide your heart** — Bonnie Tyler (9.90)

**Greatest Hits** — Dolly Parton (9.90)
</div>

## Literal result elements and attributes

Anything in the stylesheet that is *not* in the `xsl:` namespace is copied to
the output verbatim — `<div>`, `<span>`, `style="…"` and so on. These are
**literal result elements**.

When you need a *computed* attribute value, you cannot just write
`<td bgcolor="{...}">` blindly — but you can with an **attribute value
template**: an XPath expression in curly braces inside an attribute.

``` xml
<!-- the class attribute is filled in from the data -->
<tr class="cd-{position()}">
  <td><xsl:value-of select="title"/></td>
</tr>
```

`position()` is an XPath function giving the 1-based index of the current node
within the loop — handy for striping rows or numbering items.

!!! warning "value-of writes text, not markup"
    `xsl:value-of` escapes its output: if a node contains `A & B`, you get
    `A &amp; B` in the result, which is correct. It will not inject raw HTML.
    To copy nodes *as nodes* use `xsl:copy-of` instead.

## Next

Output often needs to depend on the data — highlight expensive CDs, skip empty
fields. That is the job of [conditionals](conditionals.md).
