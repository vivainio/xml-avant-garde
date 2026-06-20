---
icon: lucide/cable
---

# Tunnel parameters

When you process a document with [`apply-templates`](templates.md), control flows
through a *chain* of templates — root → section → row → cell. If a value decided
at the top (the output language, a currency, a base URL) is needed at the bottom,
1.0 forces every template in between to declare the parameter and forward it,
even though it never uses it. **Tunnel parameters** (2.0) remove that chore: the
value rides invisibly through the chain and is picked up only where wanted.

## The problem, then the fix

Without tunnelling, the middle template must relay a parameter it has no interest
in:

``` xml
<!-- 1.0 style: 'currency' threaded by hand through every level -->
<xsl:template match="catalog">
  <xsl:param name="currency"/>
  <list>
    <xsl:apply-templates select="cd">
      <xsl:with-param name="currency" select="$currency"/>   <!-- pure boilerplate -->
    </xsl:apply-templates>
  </list>
</xsl:template>
```

With `tunnel="yes"`, you set the parameter once and read it once; the templates
in between never mention it:

``` xml
<xsl:template match="/">
  <xsl:apply-templates>
    <xsl:with-param name="currency" select="'EUR'" tunnel="yes"/>   <!-- (1)! -->
  </xsl:apply-templates>
</xsl:template>

<xsl:template match="catalog">
  <list><xsl:apply-templates/></list>          <!-- (2)! says nothing about currency -->
</xsl:template>

<xsl:template match="cd">
  <xsl:param name="currency" tunnel="yes"/>     <!-- (3)! picks it up, levels deep -->
  <cd>{title} — {price} {$currency}</cd>
</xsl:template>
```

1. Set the tunnel parameter at the top of the chain.
2. The intermediate template forwards it **automatically** — no `xsl:with-param`.
3. Any descendant template receives it by declaring `tunnel="yes"`; templates
   that don't declare it simply pass it on.

## The rules

- A tunnel parameter is **set** with `<xsl:with-param … tunnel="yes"/>` on an
  `xsl:apply-templates` (or `xsl:call-template`, `xsl:apply-imports`,
  `xsl:next-match`).
- It is **read** with `<xsl:param … tunnel="yes"/>` in the receiving template.
  Both ends must say `tunnel="yes"`; the names must match.
- It stays in scope for the **entire sub-tree** of processing below where it was
  set, surviving any number of intermediate templates that ignore it.
- A template that does *not* declare the tunnel param keeps tunnelling it onward
  unchanged — you can override it for a sub-tree by setting it again.

!!! tip "What tunnelling is for — and not for"
    Reach for it for **ambient context** that many templates might need but most
    don't: locale, currency, a numbering offset, a "draft vs final" flag, the
    base output URI. For a value used by the *very next* template only, an
    ordinary [non-tunnel parameter](named-templates.md) is clearer — tunnelling a
    short hop hides data flow for no gain.

## Where to go next

- [Named templates and parameters](named-templates.md) — ordinary parameters and `xsl:with-param`.
- [Templates](templates.md) — the `apply-templates` chain tunnelling rides through.
- [Template modes](modes.md) — the other way to vary processing across a chain.
