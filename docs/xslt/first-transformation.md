# Your first transformation

The smallest useful stylesheet matches the document root and emits some HTML.
This one turns the [CD catalog](index.md#the-running-example) into a table.

``` xml title="cdcatalog.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
<xsl:template match="/">                              <!-- (1)! -->
<html>
<body>
  <h2>My CD Collection</h2>
  <table border="1">
    <tr bgcolor="#9acd32">
      <th style="text-align:left">Title</th>
      <th style="text-align:left">Artist</th>
    </tr>
    <xsl:for-each select="catalog/cd">                <!-- (2)! -->
    <tr>
      <td><xsl:value-of select="title"/></td>         <!-- (3)! -->
      <td><xsl:value-of select="artist"/></td>
    </tr>
    </xsl:for-each>
  </table>
</body>
</html>
</xsl:template>
</xsl:stylesheet>
```

1.  `match="/"` matches the **document root** — the rule fires once, for the
    whole document. Everything inside is the template for what to output.
2.  `xsl:for-each` loops over every `<cd>` element under `<catalog>`.
3.  `xsl:value-of` pulls out the text of a child element and writes it into the
    result.

## Anatomy of a stylesheet

A stylesheet is itself a well-formed XML document.

- The root element is `xsl:stylesheet` (or its synonym `xsl:transform`).
- The `xsl:` prefix is bound to the namespace
  `http://www.w3.org/1999/XSL/Transform`. The processor uses this namespace to
  tell *instructions* (things it should execute) apart from *literal result
  elements* (things it should copy to the output, like `<html>` and `<table>`).
- `version="1.0"` selects the XSLT version. 1.0 is universally supported; 2.0
  and 3.0 add more features but need a processor like Saxon.

## The result

Applied to the catalog, this produces an HTML table:

<div class="xslt-result" markdown>
**My CD Collection**

| Title | Artist |
| --- | --- |
| Empire Burlesque | Bob Dylan |
| Hide your heart | Bonnie Tyler |
| Greatest Hits | Dolly Parton |
</div>

## Next

The single big template works, but it does not scale: as documents grow you end
up with one giant rule. [Templates](templates.md) show how to break the work
into small, composable pieces.
