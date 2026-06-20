# Template modes

Sometimes a single pass over the document is not enough. You may need to walk the
*same* nodes more than once and render them **differently each time** — a compact
table of contents up top, then the full details below. Both passes select the
same `<cd>` elements, but one should emit a one-line title and the other a full
row. A plain `match="cd"` template can only do *one* of those things.

**Modes** solve this. A template can be tagged with a `mode`, and only an
`apply-templates` that asks for that same mode will reach it. The same source
element can therefore have several templates — one per mode — each producing its
own output.

## The problem: one element, two renderings

Suppose you want both an index of titles and a detail listing. Without modes a
second `<xsl:template match="cd">` would simply *override* the first — you cannot
have two unrelated rules for the same pattern in the same mode.

## A mode is a tag on the rule

Add `mode="x"` to a template, and add the matching `mode="x"` to the
`apply-templates` that should reach it.

``` xml title="modes.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:template match="catalog">
  <report>
    <toc>
      <xsl:apply-templates select="cd" mode="toc"/>     <!-- (1)! -->
    </toc>
    <details>
      <xsl:apply-templates select="cd" mode="detail"/>  <!-- (2)! -->
    </details>
  </report>
</xsl:template>

<xsl:template match="cd" mode="toc">                   <!-- (3)! -->
  <item><xsl:value-of select="title"/></item>
</xsl:template>

<xsl:template match="cd" mode="detail">                <!-- (4)! -->
  <row>
    <xsl:value-of select="title"/> — <xsl:value-of select="artist"/>
    (<xsl:value-of select="price"/>)
  </row>
</xsl:template>

</xsl:stylesheet>
```

1.  The **same** `cd` nodes, selected a first time in `mode="toc"`.
2.  The **same** nodes again, this time in `mode="detail"`.
3.  This template is considered **only** when something applies templates in
    `mode="toc"`. It is invisible to the `detail` pass and to the default mode.
4.  A second rule for the very same `match="cd"` pattern — legal, because it lives
    in a *different* mode. No clash with the `toc` rule above.

<div class="xslt-result" markdown>
```
<report>
  <toc>
    <item>Empire Burlesque</item>
    <item>Hide your heart</item>
    <item>Greatest Hits</item>
  </toc>
  <details>
    <row>Empire Burlesque — Bob Dylan (10.90)</row>
    <row>Hide your heart — Bonnie Tyler (9.90)</row>
    <row>Greatest Hits — Dolly Parton (9.90)</row>
  </details>
</report>
```
</div>

The catalog was traversed twice. Each `<cd>` matched a *different* template
purely because the two `apply-templates` calls asked for different modes.

## Modes are independent namespaces of rules

A mode partitions your templates into separate worlds:

- `apply-templates` **with** `mode="toc"` only ever considers templates that
  carry `mode="toc"`.
- `apply-templates` **with no** `mode` uses the **default (unnamed)** mode and
  considers only templates that have no `mode` attribute.

So a rule in one mode is completely invisible from another. You can have
`match="cd"` in the default mode, in `mode="toc"`, and in `mode="detail"`, and
they never interfere — each fires only when its own mode is in play.

!!! note "The default mode is just the unnamed one"
    There is nothing special about modes versus "no mode": the default is simply
    the mode with no name. Writing `mode="detail"` on both the rule and the call
    above, then dropping the attribute from both, would put that pass back in the
    default mode with identical results.

## Built-in templates carry the mode down

XSLT's built-in templates — the ones that recurse into children when you have not
written a matching rule — **preserve the current mode** as they descend. If you
apply templates in `mode="toc"` to an element you have *not* given a `toc`
template, the built-in rule still fires, and it keeps recursing **in `toc`
mode**, looking for `toc` templates on the descendants.

``` xml title="inherited-mode.xsl" linenums="1"
<xsl:template match="/">
  <out>
    <xsl:apply-templates mode="toc"/>      <!-- (1)! -->
  </out>
</xsl:template>

<xsl:template match="title" mode="toc">    <!-- (2)! -->
  <t><xsl:value-of select="."/></t>
</xsl:template>
```

1.  No rule matches `catalog` or `cd` in `mode="toc"`, so the **built-in**
    template handles them — and it descends *still in `toc` mode*.
2.  Only when recursion reaches a `<title>` does an explicit `toc` rule fire.

<div class="xslt-result" markdown>
```
<out><t>Empire Burlesque</t><t>Hide your heart</t><t>Greatest Hits</t></out>
```
</div>

!!! warning "Forgetting the mode silently does nothing useful"
    If you write a `mode="detail"` template but call `apply-templates` without
    `mode`, the default built-in rules run instead — they ignore your `detail`
    template entirely and just spit out text content. A pass that produces only
    whitespace or raw text is almost always a missing or mismatched `mode`.

## Modes versus named templates

Both modes and [named templates](named-templates.md) let one element be handled
more than one way, but they pull in opposite directions:

- **Modes** are still the *push* model: `apply-templates ... mode="x"` **selects
  and moves to** each node, and the matching mode rule runs with that node as the
  current node. Use a mode when you are walking node-sets and want a *different
  rendering* of the same nodes.
- **Named templates** are the *call* model: `call-template name="x"` runs in the
  caller's context and **does not move** the current node. Use a name for
  reusable, parameterised logic that is not "one template per node".

!!! info "XSLT 2.0 / 3.0 mode tokens"
    Later versions add convenience tokens to `apply-templates`/`xsl:template`:
    `mode="#default"` refers to the default mode explicitly, and `mode="#all"` on a
    template makes it apply in *every* mode. In 1.0 you simply name each mode and
    repeat the rule where you need it.

## Next

Once a stylesheet grows several modes and shared rules, you will want to split it
into pieces and pull them together. [Reusing stylesheets](reuse.md) covers
`xsl:include` and `xsl:import` so common templates live in one place.
