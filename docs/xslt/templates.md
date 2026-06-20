# Templates

XSLT is at heart a **rule-based**, declarative language. Instead of one big
procedure, you write many small `xsl:template` rules — each says "when you see
*this* kind of node, produce *that* output" — and let the processor decide which
rule applies where.

## apply-templates

The engine is `xsl:apply-templates`. It tells the processor: "find the matching
template for each selected node and run it." Compare this version with the
single-template one from the [previous page](first-transformation.md):

``` xml title="cdcatalog-templates.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:template match="/">                  <!-- (1)! -->
  <html>
  <body>
  <h2>My CD Collection</h2>
  <xsl:apply-templates/>
  </body>
  </html>
</xsl:template>

<xsl:template match="cd">                 <!-- (2)! -->
  <p>
    <xsl:apply-templates select="title"/>
    <xsl:apply-templates select="artist"/>
  </p>
</xsl:template>

<xsl:template match="title">              <!-- (3)! -->
  Title: <span style="color:#ff0000"><xsl:value-of select="."/></span>
  <br />
</xsl:template>

<xsl:template match="artist">
  Artist: <span style="color:#00ff00"><xsl:value-of select="."/></span>
  <br />
</xsl:template>

</xsl:stylesheet>
```

1.  The root template lays out the page, then delegates with a bare
    `xsl:apply-templates` — process all my children.
2.  A template for `cd` elements. It only cares about laying out one CD, and
    delegates the actual fields to *their* templates.
3.  A template for `title`. `select="."` means "the current node" — here, the
    `title` element whose text we want.

Each rule has one job. Adding a `<year>` field later means adding one small
template, not editing a monolith.

## How a node is matched

`xsl:apply-templates` selects a set of nodes (its children by default, or
whatever `select` names). For each node, the processor picks the **best-matching**
template by its `match` pattern. The patterns above (`cd`, `title`, `artist`)
are XPath expressions evaluated relative to the current node.

## The built-in templates

You never wrote a template for the text *inside* `title`, yet the text appeared.
That is because XSLT ships **built-in templates**:

- For any element with no matching rule: recurse into its children
  (`apply-templates`).
- For text and attribute nodes: copy their value to the output.

This is why a stylesheet with *no* templates at all still emits all the text
content of a document. It is convenient, but also a common surprise — stray text
in the output usually means a node fell through to the built-in rule.

!!! tip "select vs match"
    They look similar but do different jobs. `match` (on `xsl:template`) is a
    *pattern* that decides **whether** a rule applies. `select` (on
    `apply-templates`, `value-of`, `for-each`) is an *expression* that chooses
    **which** nodes to act on.

## Two styles, one result

You now have two ways to walk a document:

| Style | Drives iteration | Best when |
| --- | --- | --- |
| `xsl:for-each` | You, explicitly | The structure is simple and regular |
| `xsl:apply-templates` | The processor, by matching | Content is mixed, recursive, or reused |

The next page looks at [loops and output](loops-and-output.md) — `for-each` and
`value-of` — in more detail.
