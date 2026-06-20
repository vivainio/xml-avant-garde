---
icon: lucide/files
---

# Multiple result documents

A 1.0 stylesheet produces exactly **one** output. `xsl:result-document` (2.0+)
lifts that limit: a single transform can write **many** files — one HTML page
per chapter, one document per invoice, a sitemap plus the pages it lists. The
principal output stream still goes where it always did; each
`xsl:result-document` opens a *secondary* one.

## The basics

Wrap the content for a secondary file in `xsl:result-document`, giving it an
`href` (resolved against the *base output URI*) and, optionally, its own
[output settings](output.md):

``` xml
<xsl:template match="/catalog">
  <!-- principal output: an index -->
  <html><body>
    <h1>Catalogue</h1>
    <ul>
      <xsl:for-each select="cd">
        <li><a href="{position()}.html"><xsl:value-of select="title"/></a></li>
      </xsl:for-each>
    </ul>
  </body></html>

  <!-- one secondary file per cd -->
  <xsl:for-each select="cd">
    <xsl:result-document href="{position()}.html" method="html" indent="yes">
      <html><body>
        <h1><xsl:value-of select="title"/></h1>
        <p><xsl:value-of select="artist"/> — <xsl:value-of select="price"/></p>
      </body></html>
    </xsl:result-document>
  </xsl:for-each>
</xsl:template>
```

Run once, this emits the index as the main result plus `1.html`, `2.html`,
`3.html` alongside it. Each `xsl:result-document` can carry its own `method`,
`encoding`, `indent` — they are independent documents, not fragments of one.

## Per-document output settings

Because each result is independent, you can mix formats in one run — an HTML
page and a companion JSON data file, say:

``` xml
<xsl:result-document href="data.json" method="json">
  <xsl:sequence select="array { /catalog/cd ! map { 'title': string(title) } }"/>
</xsl:result-document>
```

See [JSON output](json.md) for `method="json"`, and [Producing XML
output](output.md) for the full set of `xsl:output` controls a result document
accepts.

!!! warning "You cannot read what you are writing"
    Within one transform you may not open a result document at a URI and then
    `doc()` it back in — the result does not exist until the transform finishes.
    Splitting a job into a pipeline (write in pass 1, read in pass 2) is the way
    around it; on a [streaming](streaming.md) run, each result document is still
    written eagerly as it is encountered.

!!! note "Validity and grouping"
    `xsl:result-document` is **not** allowed where it would sit inside a
    temporary tree being constructed — it must execute as an instruction, not as
    part of building a node to return. In practice: call it from a template or
    `xsl:for-each` body, not from inside an [`xsl:variable`](variables.md) that
    builds an element.

## Where to go next

- [Producing XML output](output.md) — the `xsl:output` settings each result accepts.
- [Reading and writing JSON](json.md) — a common secondary-document format.
- [Streaming](streaming.md) — splitting a huge input into many outputs in one pass.
