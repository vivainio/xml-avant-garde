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

## A template only fires when it is reached

This is the single most common gap for newcomers, so it is worth stating
plainly:

!!! warning "Defining a template does **not** run it"
    A `xsl:template` is a *dormant rule*. It produces output **only when some
    `xsl:apply-templates` reaches a node it matches** — never just because it
    exists in the stylesheet.

In the example above, the `title` template fires for one reason only: the `cd`
template explicitly pushes the node to it with `<xsl:apply-templates
select="title"/>`. Delete that line and the `title` template never runs — its
red `<span>` simply never appears in the output, even though the rule is still
sitting there.

!!! info "\"Children\" means children in the **source document**"
    `xsl:apply-templates` always works on the document being transformed,
    relative to the **current node** — never on the layout of the stylesheet.
    The templates sit side by side as top-level rules; none is nested inside
    another. When the `cd` template runs, the current node is a `<cd>` element
    *in the source XML*, and its children are the source elements `<title>`,
    `<artist>`, `<price>` (plus the whitespace text between them). The push model
    walks the **data** tree; `match` patterns decide which **rule** catches each
    node. The physical nesting of templates in the `.xsl` file is irrelevant.

Two things follow from this that often trip people up:

- **It is not the `select="title"` that matters — it is the reach.** A bare
  `<xsl:apply-templates/>` in the `cd` template would *also* fire the `title`
  template, because the bare form processes every child of the current `<cd>`
  *source* element, and the `<title>` element is one of them (so is `<artist>`,
  plus the built-in rule for the text nodes). `select="title"` just narrows
  *which* of those children get processed; it does not "call" the template by
  name.

- **The chain has to actually arrive at the node.** Processing flows root →
  `apply-templates` → `cd` → `apply-templates` → `title`. Break the chain at any
  link above and the `title` template becomes unreachable, no matter how well its
  `match` pattern fits.

This is the difference between **pull** and **push**. The single big template on
the [first page](first-transformation.md) *pulls*: `<xsl:value-of
select="title"/>` goes and fetches the text where it is written. Here the `cd`
template *pushes*: `apply-templates` hands the `title` node to whichever rule
matches it. No push, no match — no output.

## What the two select calls actually control

Splitting the work into two sequential instructions —

``` xml
<xsl:apply-templates select="title"/>
<xsl:apply-templates select="artist"/>
```

— does two distinct things, and it is worth separating them.

**Order.** The result order follows the order of the *instructions*, not the
source document. You get title-then-artist because that is the order the two
lines are written, even if the source had `<artist>` first. A bare
`<xsl:apply-templates/>`, by contrast, processes children in **document order**,
so the output would mirror the source. (Within a *single* `select`, order is
always document order — sequencing separate instructions is what lets you
override it, short of [`xsl:sort`](sorting.md).)

**Filtering.** `<cd>` also has a `<price>` child. With explicit
`select="title"` / `select="artist"`, the `<price>` node is never reached, so it
is omitted. A bare `<xsl:apply-templates/>` *would* reach it, and since there is
no `price` template, the built-in rule would dump its text (`10.90`) into the
output.

!!! note "In this example, the filtering matters more than the order"
    The source `catalog.xml` already lists `<title>` before `<artist>`, so a
    bare `apply-templates` would produce the same order here. The explicit
    selects earn their keep by *excluding* `<price>` — and by guaranteeing the
    order regardless of how the source happens to be arranged.

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

Match templates handle "for every node of this kind." When you instead need a
reusable routine you can call by name and pass arguments to, you want
[named templates and parameters](named-templates.md).
