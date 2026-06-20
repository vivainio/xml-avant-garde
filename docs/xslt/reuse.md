# Reusing stylesheets

A real transformation is rarely one file. As it grows, the same scaffolding —
formatting rules, common element templates, shared helpers — gets copied between
projects. The fix is the same as in any other language: factor the shared parts
into a **base** stylesheet and let each project pull it in. XSLT offers two
top-level elements for this, `xsl:include` and `xsl:import`, and the difference
between them is entirely about **precedence**.

## xsl:include — textual merge, same precedence

`xsl:include` is the simple one. It takes the templates from another stylesheet
and merges them in as if you had pasted them at the point of inclusion. Everything
ends up at **the same import precedence**.

``` xml title="main.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:include href="base.xsl"/>          <!-- (1)! -->

<xsl:template match="/">
  <catalogue>
    <xsl:apply-templates select="catalog/cd"/>   <!-- (2)! -->
  </catalogue>
</xsl:template>

</xsl:stylesheet>
```

1.  Pulls in every template from `base.xsl` at the same precedence as the rules
    written here. It is a top-level element — a direct child of `xsl:stylesheet`.
2.  The `cd` template that handles these can live in `base.xsl`; the merge makes
    it available exactly as if it were defined locally.

!!! warning "Same precedence means conflicts are errors"
    Because an included stylesheet sits at the *same* precedence as the including
    one, two templates matching the same nodes with the same priority is a genuine
    conflict. The XSLT 1.0 spec lets a processor either signal an error or pick the
    last one — it is **processor-dependent**, so never rely on it. Use `include`
    only when the files are partitioned so no rule overlaps.

## xsl:import — lower precedence, so you can override

`xsl:import` brings in another stylesheet too, but the imported templates get
**lower import precedence** than the importing stylesheet. That single rule is
what makes overriding possible: when both define a template for the same nodes,
the importing stylesheet **wins**, no conflict, no error.

!!! warning "`xsl:import` must come first"
    All `xsl:import` elements must appear **before** every other top-level element
    in the stylesheet — before any template, variable, output declaration, even
    before `xsl:include`. A processor will reject an `import` that follows other
    top-level content.

``` xml title="main.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:import href="base.xsl"/>           <!-- (1)! -->

<xsl:template match="cd">               <!-- (2)! -->
  <feature><xsl:value-of select="title"/></feature>
</xsl:template>

</xsl:stylesheet>
```

1.  Must be the first top-level element. `base.xsl` is now at lower precedence.
2.  This `cd` template overrides whatever `base.xsl` matched for `cd`, with no
    conflict — the higher-precedence (importing) definition simply wins.

## include vs import at a glance

| | `xsl:include` | `xsl:import` |
| --- | --- | --- |
| Precedence of pulled-in rules | **same** as host | **lower** than host |
| Can the host override them? | no — same precedence | yes — host wins cleanly |
| Conflicting matches | error / processor-dependent | resolved by precedence |
| Position among top-level elements | anywhere | **must be first** |
| Reach the overridden rule | n/a | `xsl:apply-imports` |

## Worked example: override one template, keep the rest

Here is the common shape. `base.xsl` provides a generic `cd` template that the
whole organisation shares:

``` xml title="base.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:template match="catalog">
  <list>
    <xsl:apply-templates select="cd"/>
  </list>
</xsl:template>

<xsl:template match="cd">                 <!-- (1)! -->
  <item>
    <xsl:value-of select="title"/> — <xsl:value-of select="price"/>
  </item>
</xsl:template>

</xsl:stylesheet>
```

1.  The generic rendering of a `<cd>`. Most projects are happy with it as-is.

One project wants the same overall structure but a richer `cd` line — with the
artist, and the price tagged. It imports the base and overrides **only** the
`cd` template; the `catalog` template is inherited untouched.

``` xml title="main.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:import href="base.xsl"/>             <!-- (1)! -->

<xsl:template match="cd">                 <!-- (2)! -->
  <item>
    <by><xsl:value-of select="artist"/></by>
    <xsl:apply-imports/>                  <!-- (3)! -->
  </item>
</xsl:template>

</xsl:stylesheet>
```

1.  First, as required. `base.xsl` is now lower-precedence.
2.  Overrides the base `cd` template. The base `catalog` template still fires and
    still calls `apply-templates select="cd"` — but now *this* rule answers.
3.  `xsl:apply-imports` runs the template this one overrode — the lower-precedence
    `cd` from `base.xsl` — against the **same** current node. So we add the artist,
    then delegate the title/price line back to the base instead of duplicating it.

Run against the [catalog](index.md#the-running-example):

<div class="xslt-result" markdown>
```
<list><item><by>Bob Dylan</by><item>Empire Burlesque — 10.90</item></item>...</list>
```
</div>

!!! tip "`apply-imports` is `super` for templates"
    Think of it as calling the base class's method. It does not take a `select` —
    it re-applies the imported templates for the **current** node only, letting an
    override decorate the inherited output rather than replace it wholesale.

!!! note "Where `href` is resolved"
    For both `include` and `import`, `href` is resolved **relative to the
    stylesheet that contains it** — not relative to the source document or the
    working directory. If `main.xsl` and `base.xsl` sit side by side,
    `href="base.xsl"` is correct; a base in a subfolder would be
    `href="common/base.xsl"`.

## Next

Pulling in other stylesheets has a sibling: pulling in other *data*.
[External documents](external-documents.md) covers `document()`, which lets a
transformation read a second XML file at run time.
