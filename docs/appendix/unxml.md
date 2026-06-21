---
icon: lucide/scan-text
---

# unxml — reading XML compactly

Throughout this site, schemas, stylesheets, and the odd large document are shown
**rendered with `unxml`** rather than as raw markup. This page explains the tool
those renders come from.

!!! note "Full disclosure"
    [`unxml`](https://github.com/vivainio/unxml-rs) is written by the author of
    this site, so putting it in an appendix is shameless self-promotion. It earns
    the page anyway: the site *uses* it on nearly every real-world page as a
    reading aid, and the renders you see are meaningless unless the notation is
    documented somewhere. This is that somewhere. It is an appendix, not a
    chapter, precisely because the tool is a convenience, not part of the XML
    standards the rest of the site teaches.

## The problem it solves

XML is a fine format for *machines* and a tiring one for *eyes*. The information
density of a document is low: a large fraction of the bytes are closing tags,
`xs:`/`xsl:` ceremony, and namespace declarations. When you are trying to grasp
the **shape** of a schema or the **logic** of a stylesheet, that ceremony is
noise.

`unxml` flattens an XML file into an indented, bracket-free outline that keeps the
structure and drops the syntax. It auto-detects the document kind, and has
dedicated rendering modes for the XML dialects this site covers.

## Plain mode — instance documents

With no flags, `unxml` flattens an ordinary document. Elements become indented
names, **attributes fold into `( … )`**, and **leaf text becomes `name = value`**:

``` xml title="invoice.xml — the raw document"
<invoice xmlns="urn:example:invoice" id="INV-42" currency="EUR">
  <supplier>
    <name>Acme Records</name>
    <country>FI</country>
  </supplier>
  <line sku="LP-001" qty="3">
    <desc>Vinyl LP</desc>
    <price>19.90</price>
  </line>
  <total>59.70</total>
</invoice>
```

``` text title="unxml invoice.xml"
invoice(currency="EUR", id="INV-42", xmlns="urn:example:invoice")
  supplier
    name = Acme Records
    country = FI
  line(qty="3", sku="LP-001")
    desc = Vinyl LP
    price = 19.90
  total = 59.70
```

The tree is the same; the angle brackets are gone. (On this site, *instance*
documents are usually shown as raw XML anyway — for a short specimen the markup is
already clear. The compact form earns its keep on the schemas and stylesheets
below, which are far denser.)

### Prose with inline elements

Document-style XML interleaves text with small inline elements — a `<para>` with
a `<command>` inside it. Flattening every run onto its own line would shred the
sentence, so `unxml` keeps such *mixed content* on one line, as verbatim XML:

``` text title="a paragraph with inline spans"
para = The <command>widget</command> daemon keeps its <filename>state.db</filename> in one place.
```

The rule is structural, not a list of known tags: an element flows inline when
its whole subtree is *inline-safe* — text interleaved with elements that are
themselves inline-safe. A leaf with significant multi-line text (a
`<programlisting>` or `<screen>`) is **not** inline-safe, so its parent stays in
the flattened block form and the listing keeps its line breaks. The principle:
**flatten the scaffolding, quote the prose** — angle brackets vanish from the
document skeleton but remain on the short inline spans, where the markup *is* the
content. This is what makes the [DocBook skeleton](../realworld/docbook.md) read
the way it does.

## `--xsd` — XML Schema

A schema's job is to declare a data model, but `xs:complexType` / `xs:sequence` /
`xs:element` scaffolding buries it. `--xsd` rewrites the schema vocabulary into a
type-declaration syntax that reads like the model it describes:

``` text title="unxml --xsd invoice.xsd"
schema
  element invoice
    line +
      type
        desc : xs:string
        price : xs:decimal
        @sku : xs:string (required)
        @qty : xs:integer
    @currency : xs:string
```

`:` is "typed as", `@` an attribute, `+` one-or-more (`?` optional, `*`
zero-or-more), `(required)` a use constraint. The whole [XSD](../xsd/index.md)
vocabulary compresses to a handful of symbols — see the
[XSD section](../xsd/index.md) for where this is used in anger.

## `--xslt` — stylesheets

`--xslt` is what the [XSLT at scale](../xslt/at-scale.md) page uses to show wide
spans of the DocBook stylesheets. It turns templates, `xsl:choose`, functions,
and output ceremony into pseudocode:

``` text title="unxml --xslt transform.xsl"
xsl:stylesheet(version="3.0", xmlns:xsl="http://www.w3.org/1999/XSL/Transform")
  match /invoice:
    html
      body
        apply line
  match line:
    p
      <- desc
      ":"
      <- price
```

The notation:

| `unxml --xslt` | XSLT it stands for |
|----------------|--------------------|
| `match X:`     | `xsl:template match="X"` |
| `apply` / `apply X` | `xsl:apply-templates` (of children / of a select) |
| `<- expr`      | `xsl:value-of select="expr"` (a string) |
| `<-- expr`     | `xsl:sequence select="expr"` (a sequence — the doubled arrow) |
| `<- expr ?? "text"` | a value-of with literal fallback text |
| `name as T := …` | a typed `xsl:variable` / `xsl:param` |
| `function f:n(args) -> T:` | `xsl:function name="f:n" as="T"` |
| `choose: / when X: / else:` | `xsl:choose` / `xsl:when` / `xsl:otherwise` |

## `--schematron` and `--wsdl`

The same idea covers the other validation and service dialects. `--schematron`
reduces a rule set to context-and-assertion lines:

``` text title="unxml --schematron rules.sch"
schema
  pattern
    rule line
      assert price > 0
        = Price must be positive.
```

`--wsdl` renders a WSDL 1.1 / SOAP service description, including its embedded
`<types>` schema via the `--xsd` transformation — it is what the
[SOAP and WSDL](../realworld/soap-wsdl.md) page uses.

## Handy flags

A few options come up often:

| Flag | What it does |
|------|--------------|
| `--auto` | Pick the mode from each file's extension (`.xsl`/`.xslt`, `.sch`, `.xsd`). Lets you point it at a mixed set of files. |
| `--select NAME` | Render only the subtrees whose tag matches `NAME`, as top-level fragments — how this site quotes one template out of a big module. |
| `--hide-ns p,q` | Drop namespace prefixes (and their `xmlns:` declarations) to cut noise — e.g. `--hide-ns cbc,cac` on UBL. |
| `--expand` | Inline `xsl:apply-templates` by pulling in matching templates from imports — trace a transform without chasing files. |
| `--bat` | Pipe through `bat` for syntax-highlighted, paged output (implies `--auto`). |
| `--stdin` | Read XML from standard input instead of a file. |

## Installing it

``` text
uv tool install unxml-rs        # from PyPI
# or grab a binary from the GitHub releases page
# or, if you have a Rust toolchain:
cargo install --git https://github.com/vivainio/unxml-rs
```

There is a gallery of real documents rendered side-by-side at
[vivainio.github.io/unxml-demos](https://vivainio.github.io/unxml-demos/), and
the full set of schema transformations is documented in the
[`unxml` XSD docs](https://github.com/vivainio/unxml-rs/blob/main/docs/xsd.md).

!!! tip "It is itself an XML-processing program"
    `unxml` is written in Rust on top of the `quick-xml` streaming parser — so it
    doubles as a worked example for the [Rust XML APIs](../realworld/api-rust.md)
    page. Reading XML compactly is, after all, just one more thing programs do
    with XML.
