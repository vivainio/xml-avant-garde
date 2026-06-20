---
icon: lucide/package
---

# Packages

[`xsl:include` and `xsl:import`](reuse.md) compose stylesheets by **textual
merging** — every template in an imported file becomes part of yours, with all
the name-clash and precedence juggling that implies. XSLT 3.0 adds a real
module system on top: the **package**, a separately-compiled unit with an
explicit public surface. It is to `xsl:import` what a library with a header file
is to `#include`-ing source.

!!! info "Saxon edition"
    Package support — `xsl:package` and separately-compiled package libraries —
    is a Saxon **EE** feature in the
    [Saxon 12 feature matrix](https://www.saxonica.com/html/products/feature-matrix-12.html).
    The free Saxon-HE does **not** implement packages, so on HE you stay with
    [`xsl:import`/`xsl:include`](reuse.md). Everything below assumes a licensed
    edition.

## A package and its consumer

A package replaces `xsl:stylesheet` with `xsl:package`, names itself, and
declares the **visibility** of each component:

``` xml title="price-lib.xsl"
<xsl:package name="http://example.com/price-lib"
             package-version="1.0"
             xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
             xmlns:p="http://example.com/price">

  <xsl:function name="p:with-vat" as="xs:decimal" visibility="public">
    <xsl:param name="net" as="xs:decimal"/>
    <xsl:sequence select="$net * (1 + $p:rate)"/>
  </xsl:function>

  <xsl:variable name="p:rate" as="xs:decimal" select="0.24" visibility="private"/>
</xsl:package>
```

Another package pulls it in by **name and version**, not by file path, and uses
only what is public:

``` xml title="report.xsl"
<xsl:stylesheet version="3.0" …>
  <xsl:use-package name="http://example.com/price-lib" package-version="1.0"/>

  <xsl:template match="cd">
    <gross><xsl:value-of select="p:with-vat(xs:decimal(price))"/></gross>
  </xsl:template>
</xsl:stylesheet>
```

`$p:rate` is `private`, so the consumer cannot see or collide with it — the
encapsulation `xsl:import` could never give you.

## Visibility

Every component (template, function, variable, [mode](modes.md),
[attribute set](output.md)) carries a visibility, defaulting to `private`:

| Visibility | Meaning |
| --- | --- |
| `private` | usable only inside this package (the default) |
| `public` | exported; consumers may use it as-is |
| `final` | public **and** cannot be overridden |
| `abstract` | declared but not defined — the consumer **must** supply it |
| `hidden` | inherited from a used package but withdrawn from re-export |

`xsl:expose` can re-set visibility in bulk by name pattern, rather than
annotating each component.

## Overriding — controlled, not accidental

Where `xsl:import` lets any later template silently outrank an earlier one,
packages make overriding **explicit**. A consumer wraps replacements in
`xsl:override`, and may only override components the package declared `public`
or `abstract`:

``` xml
<xsl:use-package name="http://example.com/price-lib" package-version="1.0">
  <xsl:override>
    <!-- this deployment uses a reduced rate -->
    <xsl:variable name="p:rate" as="xs:decimal" select="0.10"/>
  </xsl:override>
</xsl:use-package>
```

`abstract` components turn a package into a **template for variation**: the
library leaves a hole, every consumer fills it. It is dependency injection for
stylesheets.

!!! tip "Why bother over `xsl:import`?"
    Three concrete wins: **compile once, reuse** (a package is compiled to its
    own SEF and linked, not re-parsed into every stylesheet); **real
    encapsulation** (private components can't clash or be depended on); and
    **versioned dependencies** (`package-version` + version ranges on
    `xsl:use-package`). For a one-off shared template, `xsl:import` is still
    fine — packages earn their keep on libraries used across many stylesheets.

## Where to go next

- [Reusing stylesheets](reuse.md) — `xsl:include`/`xsl:import`, the textual-merge model packages refine.
- [User-defined functions](functions.md) — the components you most often make `public`.
- [Overview](moving-to-3.md) — back to the map of 2.0/3.0 features.
