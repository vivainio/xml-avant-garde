# Modern identity and text

XSLT 3.0 keeps everything you have learned and adds a handful of conveniences
that quietly delete boilerplate. Two of them stand out for everyday work: a
declaration that gives you the **identity transform for free**, and **text value
templates** that let you write `{...}` directly inside element content. Both
need a 3.0 processor such as Saxon, so the stylesheets here declare
`version="3.0"`.

## The identity transform, without the template

In [Producing XML output](output.md) the identity transform was a hand-written
template — the canonical "copy everything, recurse into attributes and children"
rule:

``` xml title="identity-1.0.xsl" linenums="1"
<xsl:template match="@*|node()">          <!-- the 1.0 boilerplate -->
  <xsl:copy>
    <xsl:apply-templates select="@*|node()"/>
  </xsl:copy>
</xsl:template>
```

XSLT 3.0 lets you ask for that behaviour with a single top-level declaration. The
`xsl:mode` element configures what happens to a node that **no template
matches**, and `on-no-match="shallow-copy"` means *copy it* — exactly what the
boilerplate above does, recursively, but without writing any template at all.

``` xml title="round-prices-3.0.xsl" linenums="1"
<xsl:stylesheet version="3.0"
                xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

  <xsl:output method="xml" encoding="UTF-8" indent="yes"/>

  <xsl:mode on-no-match="shallow-copy"/>    <!-- (1)! -->

  <xsl:template match="price">              <!-- override just <price> -->
    <price>
      <xsl:value-of select="format-number(., '0')"/>
    </price>
  </xsl:template>

</xsl:stylesheet>
```

1.  This *one line* replaces the whole `match="@*|node()"` identity template.
    Any node with no explicit template is shallow-copied and its attributes and
    children are processed, so the whole document flows through unchanged — every
    node except `<price>`, which the specific template below rewrites.

The result is identical to the hand-written version in
[Producing XML output](output.md): every element is copied verbatim, and only
`<price>` is reshaped — here, rounded to a whole number.

### The other `on-no-match` values

`shallow-copy` is the one that gives you the identity transform, but
`on-no-match` accepts a whole family of defaults:

- `deep-copy` — copy the node *and its entire subtree* verbatim, no recursion
  into templates.
- `shallow-copy` — copy the node itself, then process its attributes and
  children (the identity transform).
- `text-only-copy` — emit the node's text content but drop its markup.
- `shallow-skip` — emit nothing for the node, but still process its children.
- `deep-skip` — emit nothing for the node *or* its descendants.
- `fail` — raise a dynamic error if any node goes unmatched (useful to prove
  your stylesheet handles every input shape on purpose).

## `xsl:mode` sets defaults for a mode

`on-no-match` is just one setting on `xsl:mode`. The element is the general place
to configure a **mode** — including named modes, which you met in
[Template modes](modes.md). A bare `<xsl:mode .../>` configures the *unnamed*
default mode; add `name="..."` to configure one of your own:

``` xml title="named-mode-defaults.xsl" linenums="1"
<xsl:mode on-no-match="shallow-copy"/>             <!-- default mode -->
<xsl:mode name="prune" on-no-match="shallow-skip"/> <!-- a named mode -->
```

Now `xsl:apply-templates` with no mode copies nodes through, while
`apply-templates select="..." mode="prune"` drops unmatched elements but keeps
descending into them — two different default behaviours, one per mode.

## Text value templates: `{...}` in element content

You already use `{...}` in **attributes** — attribute value templates, as in
`<album price="{price}"/>`. XSLT 3.0 extends the same syntax to **text
content** when you switch it on with `expand-text="yes"`.

In 1.0, putting a computed value inside an element means `xsl:value-of`, and any
literal text around it usually needs `xsl:text` to control whitespace — see
[Whitespace and xsl:text](whitespace.md):

``` xml title="label-1.0.xsl" linenums="1"
<label>
  <xsl:text>Title: </xsl:text>
  <xsl:value-of select="title"/>
  <xsl:text> (</xsl:text>
  <xsl:value-of select="artist"/>
  <xsl:text>)</xsl:text>
</label>
```

With `expand-text="yes"` you write the same thing as ordinary text with `{...}`
holes in it:

``` xml title="label-3.0.xsl" linenums="1"
<xsl:stylesheet version="3.0"
                xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                expand-text="yes">              <!-- (1)! -->

  <xsl:template match="cd">
    <label>Title: {title} ({artist})</label>    <!-- (2)! -->
  </xsl:template>

</xsl:stylesheet>
```

1.  Set `expand-text="yes"` on `xsl:stylesheet` to enable it everywhere, or on
    any literal result element to enable it just for that element and its
    descendants (and `expand-text="no"` turns it back off for a subtree).
2.  Each `{...}` is an XPath expression evaluated and inserted into the text —
    `{title}` and `{artist}` here. The surrounding characters (`Title: `, the
    parentheses, the spaces) are kept exactly as written, so no `xsl:text` is
    needed.

!!! tip "Escaping a literal brace"

    Because `{` now starts an expression in text content, a brace you want to
    appear *literally* must be doubled: write `{{` for a single `{` and `}}` for
    a single `}`. For example `<code>{{ {name} }}</code>` outputs `{ Bob }`
    when `name` is `Bob`. (This is the same doubling rule attribute value
    templates have always used.)

## Worked example: copy through, rewrite one element's text

Combining both features: a stylesheet that turns on `expand-text` *and* uses
`on-no-match="shallow-copy"`. The whole `catalog.xml` flows through unchanged,
and one template rewrites the text of `<title>` using a `{...}` hole — no
`xsl:copy` boilerplate and no `xsl:value-of`.

``` xml title="retitle.xsl" linenums="1"
<xsl:stylesheet version="3.0"
                xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                expand-text="yes">

  <xsl:output method="xml" encoding="UTF-8" indent="yes"/>

  <xsl:mode on-no-match="shallow-copy"/>           <!-- (1)! -->

  <xsl:template match="title">                     <!-- (2)! -->
    <title>{upper-case(.)} [{../artist}]</title>
  </xsl:template>

</xsl:stylesheet>
```

1.  The free identity transform: `<catalog>`, `<cd>`, `<artist>`, `<price>` and
    every other unmatched node is shallow-copied through unchanged.
2.  The only explicit rule. The new `<title>` text is built with two `{...}`
    holes — `{upper-case(.)}` upper-cases the title (a 2.0/3.0 string function),
    and `{../artist}` pulls in the sibling artist — joined by literal text.

Applied to the shared `catalog.xml`, the result keeps the original structure and
only the title text changes:

``` xml title="output.xml"
<?xml version="1.0" encoding="UTF-8"?>
<catalog>
  <cd>
    <title>EMPIRE BURLESQUE [Bob Dylan]</title>
    <artist>Bob Dylan</artist>
    <price>10.90</price>
  </cd>
  <cd>
    <title>HIDE YOUR HEART [Bonnie Tyler]</title>
    <artist>Bonnie Tyler</artist>
    <price>9.90</price>
  </cd>
  <cd>
    <title>GREATEST HITS [Dolly Parton]</title>
    <artist>Dolly Parton</artist>
    <price>9.90</price>
  </cd>
</catalog>
```

!!! note

    The two features are independent — you can use `expand-text="yes"` in a
    plain 1.0-style stylesheet structure, or `on-no-match="shallow-copy"`
    without any text value templates. They simply compose well: declarative
    "copy everything" plus inline "rewrite this bit of text" is a remarkably
    small amount of stylesheet for a precise edit.

## Where next

That is the end of the tutorial. You started with the smallest complete 1.0
stylesheet and worked through templates, modes, variables, predicates, string
and number handling, whitespace control, reuse, and external documents — then
finished with the 2.0/3.0 conveniences on this page. You now have a working
picture of XSLT from its 1.0 fundamentals through its modern form.

XSLT 3.0 has much more to explore once you need it:

- **Higher-order functions** — pass functions as values, with `xsl:function`,
  inline `function(...) { ... }`, and operators like `fn:filter` and `fn:fold-left`.
- **Maps and arrays** — first-class associative and ordered data structures
  (`map { ... }`, `array { ... }`) for building and reshaping structured data.
- **Streaming** — process documents far larger than memory with `xsl:stream`
  and the streamable, window-friendly `xsl:iterate`.
- **Packages** — share and version libraries of templates and functions with
  `xsl:package` and `xsl:use-package`.

When you want to revisit any earlier topic, head back to the
[Overview](index.md).
