# Producing XML output

Every page so far has produced HTML. But the most common real-world use of
XSLT is *XML-to-XML* transformation: you feed in one XML vocabulary and emit a
differently-shaped one. Same engine, same template-matching, just a different
kind of result tree.

## Controlling the output with `xsl:output`

`xsl:output` is a **top-level element** — a direct child of `xsl:stylesheet`,
not something you place inside a template. It tells the processor how to
serialise the result tree.

``` xml title="catalog-to-albums.xsl" linenums="1"
<xsl:stylesheet version="1.0"
                xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

  <xsl:output method="xml"               <!-- (1)! -->
              encoding="UTF-8"           <!-- (2)! -->
              indent="yes"               <!-- (3)! -->
              omit-xml-declaration="no"/> <!-- (4)! -->

</xsl:stylesheet>
```

1.  `method` picks the serialiser. `xml` (the default when a result looks like
    XML) writes well-formed XML with self-closing empty elements. `html` uses
    HTML rules (e.g. `<br>` instead of `<br/>`). `text` outputs only character
    data — no tags at all — which is how you generate CSV, plain text, or code.
2.  `encoding` sets the character encoding named in the XML declaration and used
    when writing bytes. `UTF-8` is the sensible default.
3.  `indent="yes"` lets the processor add whitespace so the result is readable;
    `indent="no"` keeps it compact (and byte-exact, which matters when
    whitespace is significant). Indentation is advisory — processors may format
    differently.
4.  `omit-xml-declaration="no"` keeps the `<?xml version="1.0"?>` declaration;
    set it to `"yes"` to suppress it, e.g. when the output is a fragment that
    gets embedded in a larger document.

## The XML-to-XML mindset

When the output is HTML you write literal `<table>` and `<li>` elements. The
mindset for XML output is identical — but now the literal result elements you
write land in an *XML* output document of your own design.

You are reshaping one tree into another: walk the input with
`xsl:apply-templates`, and for each input node emit the output elements that the
target vocabulary calls for. The input element names and the output element
names need not match at all.

``` xml title="catalog-to-albums.xsl" linenums="1"
<xsl:template match="/catalog">
  <albums>                              <!-- output vocabulary, not input -->
    <xsl:apply-templates select="cd"/>
  </albums>
</xsl:template>

<xsl:template match="cd">
  <album>
    <name><xsl:value-of select="title"/></name>
    <by><xsl:value-of select="artist"/></by>
  </album>
</xsl:template>
```

Here `<catalog>`/`<cd>`/`<title>` on the input side become
`<albums>`/`<album>`/`<name>` on the output side. A literal `<album>` in the
stylesheet is just an instruction: "emit an `album` element here".

## Output namespaces

A serious output vocabulary usually lives in a namespace. Declare it on the
result root and every literal result element below it inherits it:

``` xml title="catalog-to-albums.xsl" linenums="1"
<xsl:template match="/catalog">
  <albums xmlns="http://example.org/albums">
    <xsl:apply-templates select="cd"/>
  </albums>
</xsl:template>
```

The catch is that namespace declarations placed on the `xsl:stylesheet` element
can *leak* into the output. If your stylesheet declares helper prefixes only for
its own use, you don't want them appearing on result elements. Use
`exclude-result-prefixes` to strip them:

``` xml title="catalog-to-albums.xsl" linenums="1"
<xsl:stylesheet version="1.0"
                xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                xmlns:fn="http://example.org/internal"
                exclude-result-prefixes="fn">
  ...
</xsl:stylesheet>
```

!!! note

    The `xsl:` prefix itself is **excluded automatically** — XSLT processors
    never copy the XSLT namespace into the result. You only list your own
    extra prefixes (here `fn`) in `exclude-result-prefixes`.

## Computed nodes: `xsl:element` and `xsl:attribute`

A literal result element has its name baked into the stylesheet source: writing
`<album>` always produces an `album` element. When the element *name* must be
decided at runtime, use `xsl:element`:

``` xml title="dynamic-name.xsl" linenums="1"
<xsl:element name="{@tag}">             <!-- (1)! -->
  <xsl:value-of select="."/>
</xsl:element>
```

1.  The element's name comes from the `tag` attribute of the current node,
    evaluated at runtime. A literal result element cannot do this — its tag name
    is fixed text in the source and there is no `<{@tag}>` syntax.

Similarly, `xsl:attribute` produces an attribute whose name and/or value are
computed. It must appear before any child content of the element it decorates:

``` xml title="dynamic-attr.xsl" linenums="1"
<album>
  <xsl:attribute name="{@kind}">         <!-- attribute name computed -->
    <xsl:value-of select="@code"/>        <!-- attribute value computed -->
  </xsl:attribute>
</album>
```

!!! tip "Attribute value templates are the lighter option"

    If only the *value* is dynamic and the attribute *name* is fixed, you don't
    need `xsl:attribute`. Use an attribute value template — `{...}` — directly on
    a literal result element:

    ``` xml
    <album price="{price}"/>
    ```

    Reach for `xsl:attribute` only when the attribute name itself is computed,
    or when the attribute is added conditionally.

## The identity transform and copying

Sometimes you want to keep most of the input unchanged and tweak only a small
part. The foundation is the **identity transform** — the canonical template that
copies every node, recursively:

``` xml title="identity.xsl" linenums="1"
<xsl:template match="@*|node()">
  <xsl:copy>
    <xsl:apply-templates select="@*|node()"/>
  </xsl:copy>
</xsl:template>
```

`xsl:copy` is a **shallow** copy: it copies the current node only (an element
with its name and namespace, but *not* its attributes or children). That is why
the identity template recurses with `xsl:apply-templates select="@*|node()"` —
the recursion is what carries attributes and descendants along.

### Copy everything, override one thing

Add a second, more specific template for the node you want to change. XSLT's
template priority means the specific match wins over the generic
`@*|node()` one:

``` xml title="round-prices.xsl" linenums="1"
<xsl:template match="@*|node()">          <!-- identity: copy everything -->
  <xsl:copy>
    <xsl:apply-templates select="@*|node()"/>
  </xsl:copy>
</xsl:template>

<xsl:template match="price">              <!-- override just <price> -->
  <price>
    <xsl:value-of select="format-number(., '0')"/>
  </price>
</xsl:template>
```

Every element flows through the identity template untouched, except `<price>`,
which gets reshaped. This "copy-and-override" idiom is the workhorse of XML
editing pipelines.

### `xsl:copy` vs `xsl:copy-of`

- `xsl:copy` — **shallow**, current node only; you decide what (if anything) to
  copy of its content.
- `xsl:copy-of` — **deep**, copies a whole subtree (selected by an expression)
  verbatim, attributes and descendants included, no recursion needed.

``` xml title="copy-of.xsl" linenums="1"
<xsl:copy-of select="cd"/>   <!-- copies every <cd> and all its children -->
```

!!! info

    Use `xsl:copy-of` when you want to splice a branch of the input through
    unchanged. Use `xsl:copy` + `xsl:apply-templates` when you want to copy a
    node but still let templates transform its descendants.

## Worked example: catalog → albums

Putting it together. We turn `catalog.xml` into an `albums` document in a result
namespace, with `indent="yes"`, and one **computed attribute** — an `id` whose
value is built from the position.

``` xml title="catalog-to-albums.xsl" linenums="1"
<xsl:stylesheet version="1.0"
                xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

  <xsl:output method="xml" encoding="UTF-8" indent="yes"/>

  <xsl:template match="/catalog">
    <albums xmlns="http://example.org/albums">
      <xsl:apply-templates select="cd"/>
    </albums>
  </xsl:template>

  <xsl:template match="cd">
    <album id="a{position()}" price="{price}">  <!-- (1)! -->
      <name><xsl:value-of select="title"/></name>
      <by><xsl:value-of select="artist"/></by>
    </album>
  </xsl:template>

</xsl:stylesheet>
```

1.  Two attribute value templates: `id` is computed from `position()`
    (`a1`, `a2`, …), and `price` lifts the child `<price>` element up into an
    attribute. Both attributes inherit the `http://example.org/albums` namespace
    declared on the root.

Applied to the shared `catalog.xml`, the result is:

``` xml title="albums.xml"
<?xml version="1.0" encoding="UTF-8"?>
<albums xmlns="http://example.org/albums">
  <album id="a1" price="10.90">
    <name>Empire Burlesque</name>
    <by>Bob Dylan</by>
  </album>
  <album id="a2" price="9.90">
    <name>Hide your heart</name>
    <by>Bonnie Tyler</by>
  </album>
  <album id="a3" price="9.90">
    <name>Greatest Hits</name>
    <by>Dolly Parton</by>
  </album>
</albums>
```

The input's element-centric shape (`<price>10.90</price>`) became an
attribute-centric shape (`price="10.90"`), wrapped in a new vocabulary and
namespace — a different tree, produced entirely from literal result elements
plus a couple of attribute value templates.

## Next

[Sorting](sorting.md) — once you are emitting your own output vocabulary, you'll
usually want its elements in a meaningful order rather than document order.
