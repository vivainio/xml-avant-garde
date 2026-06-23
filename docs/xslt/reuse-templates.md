---
icon: lucide/layers-2
---

# Case study: reusing match templates

[Reusing stylesheets](reuse.md) made the general point: `xsl:include` and
`xsl:import` carry **every** top-level declaration across, and import precedence
resolves clashes. For params, variables, and functions, overriding is the whole
story — the higher-precedence declaration simply replaces the lower one, and that
is why the [most common reuse patterns](reuse.md#worked-example-override-parameters-not-templates)
never touch a `match` template.

Match templates are the one declaration kind with **extra machinery**, and a whole
family of real reuse problems lives in that machinery: not "supply a different
value" but "render this element a bit differently while keeping everything else."
This page is that family — when template overriding is the right seam, and the two
tools (`xsl:apply-imports`, `xsl:next-match`) that make it work.

## When templates are the right seam

Reach for importing-and-overriding a `match` template when you are inheriting a
**body of rendering rules** and need to change or decorate a few of them. The
recurring cases:

| Case | What you import | What you override |
| --- | --- | --- |
| **Shared rendering base, per-project tweaks** | an org-wide stylesheet that renders every element | the handful of elements one project wants different |
| **Decorate, don't replace** | a base rule whose output is *almost* right | the same rule, wrapping/extending the inherited output |
| **Layered output formats** | base templates over one content model | the presentation rules that differ per medium (HTML / print / EPUB) |
| **Customizing a third-party stylesheet** | DocBook XSL, TEI, a generated stylesheet you don't own | selected templates, without editing upstream |

The tell in every row: you want **almost all** of the imported behaviour. If you
find yourself overriding *most* of the templates, you don't have a base — write
the stylesheet directly. And if what actually varies is a *value* or a *format*
rather than how an element renders, override a [param or
function](reuse.md#precedence-applies-to-every-declaration-kind) instead — it has
none of the machinery below and needs none of it.

## The mechanism: the override wins, the shadowed rule stays reachable

Overriding a `match` template works like any other declaration — higher import
precedence wins — but templates have the one extra capability: an override can
**re-run the rule it replaced**, against the current node, with `xsl:apply-imports`.

Here `base.xsl` provides a generic `cd` template the whole organisation shares:

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
    override decorate the inherited output rather than replace it wholesale. There
    is no equivalent for params, variables, or functions: those have no "call the
    one I overrode," because the higher-precedence declaration just *is* the value.

## `next-match`: the sibling for same-precedence rules

`xsl:apply-imports` reaches across an **import boundary** — it re-runs the rule
the override replaced in a lower-precedence module. Its XSLT 2.0+ sibling
`xsl:next-match` is more general: it re-applies the **next** template that would
have matched the current node, regardless of whether the loser is in an imported
module *or* simply a lower-priority rule in the **same** stylesheet.

That makes `next-match` the tool when you build a chain of rules for one element by
**specificity** rather than by import layer:

``` xml title="One element, a general rule and a special case" linenums="1"
<xsl:template match="cd">                       <!-- (1)! -->
  <item><xsl:value-of select="title"/></item>
</xsl:template>

<xsl:template match="cd[@remastered = 'true']">  <!-- (2)! -->
  <xsl:next-match/>                               <!-- (3)! -->
  <badge>remastered</badge>
</xsl:template>
```

1.  The general rule. Lower priority, because its pattern is less specific.
2.  A more specific pattern, so it wins for remastered CDs — no import involved,
    both rules live here.
3.  Re-applies the next-best match — the general `cd` rule above — then adds the
    badge. `apply-imports` could not reach this rule; it is the same precedence,
    not an imported one.

Rule of thumb: **`apply-imports` to reach into a base you imported; `next-match`
to fall through to a less-specific rule beside you.** Both decorate rather than
replace, and both run against the current node only.

## The archetype: customizing a third-party stylesheet

The case that makes all of this pay off is the one where you *can't* edit the base:
a published stylesheet like **DocBook XSL** or **TEI**, or a stylesheet a tool
generated for you. You want its hundreds of carefully-tuned rendering rules, but a
dozen of them must behave differently for your house style.

``` xml title="house-style.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="2.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:import href="http://docbook.sourceforge.net/release/xsl-ns/current/html/docbook.xsl"/>  <!-- (1)! -->

<xsl:param name="admon.graphics" select="1"/>   <!-- (2)! -->

<xsl:template match="d:note" xmlns:d="http://docbook.org/ns/docbook">  <!-- (3)! -->
  <aside class="note callout">
    <xsl:apply-imports/>
  </aside>
</xsl:template>

</xsl:stylesheet>
```

1.  Import the whole upstream stylesheet, untouched and un-forked. It is now the
    lower-precedence layer; you never edit a line of it.
2.  Tune behaviour the upstream exposes as a **parameter** — the simplest override,
    no template needed (the [param pattern](reuse.md#worked-example-override-parameters-not-templates)).
3.  Override exactly one rendering rule, and use `apply-imports` to reuse the
    upstream's note-body rendering inside your own wrapper.

This is import precedence doing its defining job: your rules win cleanly, the
upstream's win everywhere you stayed silent, and an upgrade of the imported
stylesheet flows straight through. The [DocBook xslTNG walkthrough](at-scale.md)
is a production-scale version of exactly this layering, with modes and an element
family per module.

## Multi-format layering

The same shape solves "one content model, several output media." A base layer owns
the structural/semantic templates; per-format layers import it and override only
the templates that differ by medium:

```
content-base.xsl          ← the semantic rules: what a <section>, <cd>, <note> *means*
   ▲          ▲
html.xsl    print-fo.xsl  ← each imports the base, overrides only presentation rules
```

`html.xsl` renders `<note>` as `<aside>`; `print-fo.xsl` renders it as an
`fo:block` with a border — but both inherit the base's decision about *which*
elements are notes and how they nest. Nothing structural is duplicated, and a new
format is a new thin import layer, not a fork.

## When not to reach for this

The whole machinery above earns its keep only when you are inheriting rendering
rules. Two cheaper seams come first:

- **Values or formatting vary, not rendering** → override a `param` or `function`.
  No precedence subtleties, no `apply-imports`, and it is the seam that scales to
  [generated binding modules](generated-and-handwritten.md).
- **Rules partition cleanly, nothing overlaps** → use `xsl:include`, not `import`.
  There is no base/override relationship to model, and the
  [duplicate-declaration error](generated-and-handwritten.md#two-tools-two-jobs)
  becomes a useful guardrail.

## Next

For the param-and-function side of reuse — and why the most common patterns avoid
templates entirely — see [Reusing stylesheets](reuse.md). For reuse with an
*enforced* public surface rather than a textual merge, see
[Packages](packages.md).
