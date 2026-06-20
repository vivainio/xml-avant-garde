# Whitespace and xsl:text

Sooner or later your output grows a stray space, a run of blank lines, or
indentation you never asked for. Almost always the cause is whitespace — in the
*stylesheet*, in the *source document*, or added by the serialiser. This page
shows where each kind of whitespace comes from and the small tools that let you
control it: `xsl:text`, `xsl:strip-space` / `xsl:preserve-space`,
`normalize-space()`, and `xsl:output indent`.

## Whitespace in the stylesheet

A whitespace-only text node sitting *between* XSLT elements is stripped from the
stylesheet automatically. The indentation that makes your stylesheet readable
therefore does **not** reach the output:

``` xml
<xsl:template match="cd">
  <xsl:value-of select="title"/>
  <xsl:value-of select="artist"/>
</xsl:template>
```

The newlines and spaces around the two `xsl:value-of` elements are
whitespace-only nodes between instructions, so they are discarded. The title and
artist come out glued together:

<div class="xslt-result" markdown>
Empire BurlesqueBob Dylan
</div>

The one exception is text inside `xsl:text` — that is **never** stripped (see
below). Everything else that is whitespace-only between elements goes away.

!!! warning "Indentation inside literal result content is not whitespace-only"

    The rule only removes text nodes that are *entirely* whitespace. The moment
    a text node also contains real characters, all of its whitespace is kept.
    So indentation *inside* a literal result element surfaces in the output:

    ``` xml
    <line>
      <xsl:value-of select="title"/>
    </line>
    ```

    The newline-plus-spaces before and after the `xsl:value-of` are kept,
    because they are part of the `<line>` element's content. The result is
    `<line>` with a leading newline, indentation, the title, then another
    newline — rarely what you want. Pull the `xsl:value-of` onto the same line as
    its parent, or use `xsl:text`, to avoid it.

## xsl:text — emit text exactly

`xsl:text` writes its content to the output **verbatim**, and its content is
exempt from the whitespace-stripping rule. That makes it the precise tool for
two jobs: forcing a space (or newline) to appear, and stopping unwanted
whitespace from appearing.

### Force exactly one space

To put a single space between two values, do **not** rely on the layout of your
stylesheet — wrap the separator in `xsl:text`:

``` xml
<xsl:value-of select="title"/><xsl:text> </xsl:text><xsl:value-of select="artist"/>
```

The space inside `<xsl:text> </xsl:text>` is real content, so it is emitted
exactly once:

<div class="xslt-result" markdown>
Empire Burlesque Bob Dylan
</div>

A literal newline works the same way — `<xsl:text>&#10;</xsl:text>` emits one
line break, which is the usual way to end a record in `method="text"` output.

### Suppress unwanted whitespace

An empty `<xsl:text/>` outputs nothing, but it *is* an element. Placing it
between two values lets you spread instructions across several indented lines
while the whitespace between them stays whitespace-only (and so is stripped):

``` xml
<xsl:value-of select="title"/>
<xsl:text/>                       <!-- (1)! -->
<xsl:value-of select="artist"/>
```

1.  The newlines either side of `<xsl:text/>` are whitespace-only text between
    elements, so they are removed — the two values join with nothing between
    them, even though the source is nicely indented.

<div class="xslt-result" markdown>
Empire BurlesqueBob Dylan
</div>

!!! tip "Make separators explicit"

    When output spacing matters, stop depending on stylesheet layout and state
    every separator with `xsl:text`. It reads more verbosely but it is
    unambiguous, and it survives reformatting your stylesheet.

## Whitespace in the source document

Whitespace in the **input** is a separate question. By default the processor
treats text nodes in the source as **significant** — even ones that are only
indentation. A pretty-printed source therefore carries stray newlines and
spaces in its text nodes.

Two top-level elements control this:

- `xsl:strip-space elements="..."` — remove whitespace-only text nodes from the
  listed source elements.
- `xsl:preserve-space elements="..."` — keep them (the default for everything).

Both take a space-separated list of element names, or `*` for all elements:

``` xml title="strip-space.xsl" linenums="1"
<xsl:stylesheet version="1.0"
                xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

  <xsl:strip-space elements="*"/>          <!-- (1)! -->
  <xsl:preserve-space elements="title"/>   <!-- (2)! -->

</xsl:stylesheet>
```

1.  Drop whitespace-only text nodes from every source element — the indentation
    between `<catalog>`, `<cd>` and their children disappears, so things like
    `count(node())` no longer trip over phantom text nodes.
2.  Override the blanket strip for `<title>`, where you want any whitespace-only
    text kept.

!!! note "It only removes *whitespace-only* nodes"

    `xsl:strip-space` never touches text that contains real characters. A
    `<title>` of `Empire Burlesque` keeps its text untouched; only nodes that
    are *entirely* whitespace are candidates for removal. To clean leading,
    trailing or internal whitespace from a real value, use `normalize-space()`.

## normalize-space() — per-value cleanup

Stripping handles whole nodes; `normalize-space()` cleans a single string. It
trims leading and trailing whitespace and collapses every internal run of
spaces, tabs and newlines to one space. It is the standard defence against messy
source text:

``` xml
<xsl:value-of select="normalize-space(artist)"/>
```

If the source held `<artist>  Bob    Dylan  </artist>`, the output is the tidy:

<div class="xslt-result" markdown>
Bob Dylan
</div>

See [String functions](strings.md) for the rest of the XPath 1.0 string
toolbox.

## Output indentation

`xsl:output indent="yes"` asks the serialiser to pretty-print the result tree by
adding its own newlines and indentation:

``` xml
<xsl:output method="xml" indent="yes"/>
```

This is purely for human readability of the *result*.

!!! warning "indent is a hint, not a contract"

    `indent="yes"` is advisory — different processors indent differently, and it
    may add or alter whitespace inside your elements. Never turn it on for output
    whose whitespace must be preserved byte-for-byte (signed documents,
    whitespace-significant formats). Use `indent="no"` when exactness matters.

See [Producing XML output](output.md) for the full `xsl:output` reference.

## disable-output-escaping

Normally the serialiser escapes special characters: a `<` in your text becomes
`&lt;`, an `&` becomes `&amp;`. `disable-output-escaping="yes"` on
`xsl:value-of` or `xsl:text` turns that off, so the characters are written
**raw**:

``` xml
<xsl:text disable-output-escaping="yes">&lt;hr/&gt;</xsl:text>
```

Instead of the visible text `<hr/>`, this writes the raw markup `<hr/>` into the
output stream.

!!! warning "Last resort — and not portable"

    `disable-output-escaping` is a serialisation hack: the processor is allowed
    to ignore it entirely, and it does nothing when the output is consumed as a
    node tree rather than serialised bytes. It is easy to produce ill-formed
    output with it. Prefer building real result nodes (`xsl:element`,
    `xsl:copy-of`, literal result elements) and reach for
    `disable-output-escaping` only when nothing else can inject the markup you
    need.

## Worked example: title and artist, controlled separator

The shared `catalog.xml`:

``` xml title="catalog.xml"
<catalog>
  <cd><title>Empire Burlesque</title><artist>Bob Dylan</artist><price>10.90</price></cd>
  <cd><title>Hide your heart</title><artist>Bonnie Tyler</artist><price>9.90</price></cd>
</catalog>
```

We want one line per CD, `title — artist`, with exactly one space either side of
the dash. The `xsl:text` separators make the spacing explicit, and a final
`<xsl:text>&#10;</xsl:text>` ends each line:

``` xml title="join.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
  <xsl:output method="text"/>
  <xsl:strip-space elements="*"/>          <!-- (1)! -->

  <xsl:template match="/catalog">
    <xsl:for-each select="cd">
      <xsl:value-of select="title"/>
      <xsl:text> — </xsl:text>             <!-- (2)! -->
      <xsl:value-of select="artist"/>
      <xsl:text>&#10;</xsl:text>           <!-- (3)! -->
    </xsl:for-each>
  </xsl:template>
</xsl:stylesheet>
```

1.  Drop indentation from the source so no phantom text nodes leak through.
2.  The separator — space, em dash, space — emitted exactly as written.
3.  A literal newline ends the record.

<div class="xslt-result" markdown>
Empire Burlesque — Bob Dylan

Hide your heart — Bonnie Tyler
</div>

Now compare the same template **without** `xsl:text`, relying on stylesheet
layout for the spacing:

``` xml title="join-broken.xsl" linenums="1"
<xsl:template match="/catalog">
  <xsl:for-each select="cd">
    <xsl:value-of select="title"/> — <xsl:value-of select="artist"/>
  </xsl:for-each>
</xsl:template>
```

The `—` here sits in a text node that also holds the surrounding spaces and
newlines. Because that node is **not** whitespace-only (it contains the dash),
its whitespace is *kept* — including the newline and indentation before the next
iteration's `title`. The records run together with ragged spacing:

<div class="xslt-result" markdown>
Empire Burlesque — Bob Dylan
    Hide your heart — Bonnie Tyler
</div>

The lesson: when spacing matters, say it with `xsl:text` rather than letting the
shape of the stylesheet decide.

## Next

[Template modes](modes.md) — once whitespace is under control, modes let you
process the same nodes more than once, each pass producing a different kind of
output.
