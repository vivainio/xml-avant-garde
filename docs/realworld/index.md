---
icon: lucide/boxes
---

# Real-world XML applications

The [e-invoicing section](../einvoicing/index.md) followed *one* XML vocabulary
all the way down — UBL, its XSD, its Schematron, its code lists. This section does
the opposite: it visits **many** real XML applications, briefly, to show how the
format is actually used *in the wild* — the vocabularies people exchange, and the
APIs programs use to read and write them.

Two strands run through it:

- **The vocabularies** — SVG, SOAP, Office documents, feeds, XBRL, and the rest.
  The moment XML leaves the textbook, the interesting part is rarely the elements;
  it is the **namespaces**: how a document mixes two of them, how a schema in one
  namespace `import`s another, how a base vocabulary is *extended* without being
  changed. Each vocabulary page is built around one of those patterns.
- **Working with XML in code** — the parsing models (DOM, SAX, pull) and the APIs
  built on them, where the same namespace ideas reappear as `NamespaceContext`,
  prefix-to-URI maps, and the streaming-vs-tree trade-off every XML program makes.

!!! tip "Read the namespace chapters first"
    These pages assume you know what a namespace *is* and how a schema declares
    one. If `targetNamespace`, `elementFormDefault`, or
    [`import` vs `include`](../xsd/modular-schemas.md) are fuzzy, skim
    [XSD → Modular schemas](../xsd/modular-schemas.md) and the
    [namespaces part of Anatomy of a UBL invoice](../einvoicing/ubl-invoice.md#three-namespaces)
    first — they are the prerequisites for everything below.

## Reading XML with `unxml`

Real documents are *verbose* — and **schemas** worst of all. A 500-line
[XSD](../xsd/index.md) buries its data model under `xs:complexType` /
`xs:sequence` / `xs:element` scaffolding. So whenever these pages show a schema,
they show it rendered with [**`unxml --xsd`**](https://github.com/vivainio/unxml-rs),
a small tool that rewrites the `xs:*` vocabulary into a terse,
type-declaration-like pseudocode that reads like the data model it describes:

``` text title="unxml --xsd (a schema fragment)"
type wptType
  ele : xsd:decimal ?
  time : xsd:dateTime ?
  @lat : latitudeType (required)
  @lon : latitudeType (required)
type latitudeType : xsd:decimal [-90.0..90.0]
```

`?` is optional, `*` zero-or-more, `:` is "typed as", `@` an attribute,
`[m..n]` a range — the whole [XSD](../xsd/index.md) vocabulary compressed to a
handful of symbols. It makes a long schema *browseable*. (`unxml` also flattens
plain instance documents into the same indented form — attributes into `( … )`,
text as `name = value` — but for the short instances on these pages the raw XML is
clearer, so we show that directly.)

!!! info "Try it yourself"
    `unxml` installs from PyPI (`uv tool install unxml-rs`) or
    [GitHub releases](https://github.com/vivainio/unxml-rs/releases). There is a
    gallery of real documents rendered side-by-side at
    [vivainio.github.io/unxml-demos](https://vivainio.github.io/unxml-demos/). The
    full set of schema transformations is documented in the
    [`unxml` XSD docs](https://github.com/vivainio/unxml-rs/blob/main/docs/xsd.md).

## The tour — vocabularies

Each page is self-contained — jump to whichever vocabulary you care about. They
are ordered roughly from "you see it every day" to "you only see it in finance".

| # | Vocabulary | The namespace pattern it shows |
| --- | --- | --- |
| 1 | [**SVG**](svg.md) | A *default* namespace plus a borrowed `xlink:` prefix — and the same SVG **embedded inside HTML**, the canonical mixed-namespace document |
| 2 | [**SOAP & WSDL**](soap-wsdl.md) | Several namespaces as a *contract*: an envelope wrapping a payload, and a WSDL that `import`s its XSD types |
| 3 | [**Office: OOXML & ODF**](office-ooxml-odf.md) | A ZIP of XML *parts*, each a forest of namespaces, wired together by relationships |
| 4 | [**DocBook**](docbook.md) | A vocabulary *growing up*: DocBook 4's no-namespace DTD becomes DocBook 5's namespaced RELAX NG — semantic markup, one source to many outputs |
| 5 | [**Atom & feed extensions**](atom-feeds.md) | *Extending* a base vocabulary — Dublin Core, iTunes podcast tags — without touching it |
| 6 | [**XSL-FO & Apache FOP**](xsl-fo-fop.md) | A vocabulary that is *generated*, not authored: XSLT → `fo:` → PDF |
| 7 | [**XML Signature**](xml-dsig.md) | Why namespaces force **canonicalization**: signing bytes that mean the same thing |
| 8 | [**SAML**](saml.md) | Signed assertions and SSO: four namespaces (`saml:`/`samlp:`/`ds:`/`md:`), one per layer, and XML-DSig doing real work |
| 9 | [**GPX & KML**](geo-gpx-kml.md) | Two geo vocabularies, two extension styles (`<extensions>` vs a `gx:` prefix) |
| 10 | [**Factur-X & ZUGFeRD**](facturx-zugferd.md) | XML as a *payload*, not the document: CII **embedded in a PDF/A-3**, with XMP (RDF/XML) as the describing metadata |
| 11 | [**XBRL**](xbrl.md) | Namespacing taken to the limit: a taxonomy of concepts in *your* namespace, built on `xbrli:` |

## The tour — working with XML in code

The vocabularies above are what gets *exchanged*; these pages are how programs
*process* it. Start with the concepts page, then dip into whichever runtime you
work in — they cover the same five tasks (parse, navigate with namespaces,
validate, transform, write) so you can compare them directly.

| Page | What it covers |
| --- | --- |
| [**Working with XML in code**](xml-in-code.md) | The three parsing models — **DOM**, **SAX**, **pull/streaming** — and the namespace API (`prefix → URI`) that every language reinvents. Read this first. |
| [**APIs: Java**](api-java.md) | JAXP (DOM/SAX/StAX), XPath with `NamespaceContext`, XSD validation, XSLT via Saxon, JAXB data binding |
| [**APIs: .NET**](api-dotnet.md) | LINQ-to-XML (`XDocument`), `XmlReader` streaming, XPath with `XmlNamespaceManager`, `XmlSchemaSet`, `XslCompiledTransform` |
| [**APIs: Python**](api-python.md) | `lxml` and `ElementTree`, namespace maps, `iterparse` streaming, XPath / XSLT / XML Schema, `xsdata` binding |
| [**APIs: Rust**](api-rust.md) | `quick-xml` streaming, `roxmltree`, XPath/XSD via libxml2 — and where Rust *stops* (no native XSLT 2/3). The language `unxml` itself is written in |

!!! note "The data is illustrative"
    The documents here are hand-written specimens — real *structure*, namespaces
    and schema shapes, made-up content. Where a snippet is trimmed from a larger
    real file, it says so. The code snippets are idiomatic and correct against
    current library APIs, but kept short — they show the *shape* of each API, not a
    complete program.
