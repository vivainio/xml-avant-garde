---
icon: lucide/file-code-2
---

# APIs: Python (lxml)

Python has two XML libraries worth knowing: the standard-library
`xml.etree.ElementTree` (always available, minimal) and **`lxml`** (a binding to
libxml2/libxslt that adds full [XPath](../xpath/index.md), [XSLT](../xslt/index.md)
1.0, [XSD](../xsd/index.md) and Schematron). For real work, use `lxml`. This page
works the [five tasks](xml-in-code.md) on the shared
[`invoice.xml`](xml-in-code.md#the-running-example).

!!! info "Install"
    `pip install lxml`. The API mirrors ElementTree (`lxml.etree` is largely a
    drop-in superset), so most snippets below work on stdlib `ElementTree` too —
    except XPath with prefixes, XSLT, and schema validation, which are lxml-only.

## 1. Parse — the element tree

``` python
from lxml import etree

doc = etree.parse("invoice.xml")          # ElementTree
root = doc.getroot()                      # <inv:invoice>
```

Python's model has one defining quirk: element tags are stored in **James Clark
notation**, `{namespace-uri}localname`:

``` python
print(root.tag)        # {urn:example:invoice}invoice   (1)
total = root.find("{urn:example:invoice}total")
print(total.text)      # 100.00
print(total.get("currency"))   # EUR  -- unprefixed attr, so no namespace (2)
```

1.  The prefix `inv` from the file is **gone** by the time you have a tree — it is
    replaced by the URI in braces. This is Python making the
    [namespace rule](xml-in-code.md#the-namespace-problem-in-every-language)
    physical: an element's identity *is* `{uri}local`, and the prefix was only ever
    a serialization detail.
2.  `get("currency")` with a bare name targets the no-namespace attribute, exactly
    like [.NET](api-dotnet.md). A namespaced attribute would be
    `get("{uri}currency")`.

## 2. Navigate — XPath with a namespace map

`{uri}local` is verbose, so for real navigation use `.xpath()` with a
`namespaces=` dict — lxml's [prefix → URI map](xml-in-code.md#the-namespace-problem-in-every-language):

``` python
ns = {"i": "urn:example:invoice",        # (1)!
      "p": "urn:example:party"}

total = doc.xpath("/i:invoice/i:total/text()", namespaces=ns)[0]   # "100.00"
ccy   = doc.xpath("string(/i:invoice/i:total/@currency)", namespaces=ns)  # "EUR"
name  = doc.xpath("//p:name/text()", namespaces=ns)[0]             # "Acme Records"
```

1.  `i` is *our* prefix bound to the document's URI. As everywhere, the file's own
    prefix (`inv`) is irrelevant — only the URI matches. lxml will raise if you use
    a prefix in the query that is not in `namespaces`, which is friendlier than
    silently matching nothing.

!!! warning "There is no default-prefix shortcut in XPath 1.0"
    A common trap: you cannot map the *empty* prefix `""` to a URI in
    `namespaces=` and then write `//total` — XPath 1.0 forbids it (the same rule
    as the [SVG page](svg.md#querying-namespaced-svg-with-xpath)). You must give
    the namespace a non-empty prefix in your map and use it. (libxml2 supports
    XPath 1.0 only.)

## 1b. Stream large files — `iterparse`

lxml's `iterparse` is the [pull/streaming model](xml-in-code.md#three-ways-to-read-xml):
it yields elements as they finish parsing, and you free them to keep memory flat:

``` python
INV = "urn:example:invoice"
for _, el in etree.iterparse("big.xml", tag=f"{{{INV}}}total"):   # (1)!
    print(el.text, el.get("currency"))
    el.clear()                                                    # (2)!
    while el.getprevious() is not None:
        del el.getparent()[0]                                     # (3)!
```

1.  `tag=` filters to just the elements you care about — note the triple braces:
    an f-string `{{{INV}}}` produces `{urn:example:invoice}` around the local name.
2.  `el.clear()` drops the element's children and text once you are done with it.
3.  The `getprevious()`/`del` dance also removes already-processed *siblings* —
    the canonical lxml idiom for processing a huge file in constant memory.

## 3. Validate against XSD

``` python
schema = etree.XMLSchema(etree.parse("invoice.xsd"))    # (1)!
doc = etree.parse("invoice.xml")

if not schema.validate(doc):
    for err in schema.error_log:                        # (2)!
        print(f"line {err.line}: {err.message}")
```

1.  `XMLSchema` resolves [`xs:import`/`xs:include`](../xsd/modular-schemas.md)
    relative to the schema file automatically. lxml also has `etree.Schematron`
    and `etree.RelaxNG` for the [Schematron](../schematron/index.md) and RELAX NG
    layers.
2.  `error_log` holds *every* failure with line numbers — the first layer of the
    [validation pipeline](../einvoicing/validation-pipeline.md), in three lines.

## 4. Transform — XSLT

libxslt is **XSLT 1.0**. For 1.0 work, lxml is excellent and — the key habit —
lets you compile the stylesheet once and reuse it:

``` python
transform = etree.XSLT(etree.parse("to-fo.xsl"))   # compile once
result = transform(etree.parse("invoice.xml"))     # run per input
result.write("invoice.fo")                          # -> XSL-FO, then Apache FOP
```

!!! note "Need XSLT 2.0 / 3.0 in Python?"
    libxslt stops at 1.0. For the [modern XSLT](../xslt/moving-to-3.md) this site
    teaches (grouping, `xsl:function`, JSON), call **Saxon** — there is a
    `saxonche` (Saxon-C/HE) Python package that runs 3.0 from Python.

## 5. Data binding

Python has no built-in XML data binding, but `xsdata` (and the older
`generateDS`) generate dataclasses from an XSD:

``` bash
xsdata generate invoice.xsd --package model     # XSD -> @dataclass model
```

``` python
from xsdata.formats.dataclass.parsers import XmlParser
from model import Invoice

inv = XmlParser().parse("invoice.xml", Invoice)   # XML -> dataclass
print(inv.total)                                  # 100.00
```

For loosely-structured or [extension-heavy](atom-feeds.md) documents, though, the
`lxml` element/XPath API above is usually the better fit than binding.

## Python cheat-sheet

| Task | API |
| --- | --- |
| Tree parse | `etree.parse` (`lxml`, or stdlib `ElementTree`) |
| Names | `{uri}local` (James Clark notation) |
| Streaming | `etree.iterparse` (+ `clear()` to free memory) |
| XPath | `.xpath(expr, namespaces={...})` (lxml only) |
| Validate | `etree.XMLSchema` (also `Schematron`, `RelaxNG`) |
| XSLT 1.0 | `etree.XSLT` |
| XSLT 2/3 | Saxon (`saxonche`) |
| Binding | `xsdata` / `generateDS` |

Compare with [Java](api-java.md), [.NET](api-dotnet.md) and [Rust](api-rust.md).
