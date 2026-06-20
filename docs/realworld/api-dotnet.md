---
icon: lucide/hash
---

# APIs: .NET

.NET has two generations of XML API living side by side. The older `XmlDocument`
is a classic [DOM](xml-in-code.md#three-ways-to-read-xml); the newer **LINQ-to-XML**
(`XDocument`, from `System.Xml.Linq`) is what you reach for today. Streaming is
`XmlReader`/`XmlWriter`. This page works the [five tasks](xml-in-code.md) on the
shared [`invoice.xml`](xml-in-code.md#the-running-example), in C#.

!!! info "Namespaces (the C# kind)"
    `System.Xml.Linq` — `XDocument`, `XElement`, `XName`, `XNamespace`.
    `System.Xml` — `XmlReader`, `XmlWriter`, `XmlDocument`, `XmlNamespaceManager`.
    `System.Xml.XPath` — XPath extensions. `System.Xml.Schema` — XSD validation.

## 1. Parse — LINQ-to-XML

``` csharp
XDocument doc = XDocument.Load("invoice.xml");
XElement root = doc.Root;                       // <inv:invoice>
```

LINQ-to-XML is namespace-aware automatically — no flag to remember (unlike
[Java](api-java.md)). The trick is that `XName` *bundles* the namespace into the
name. You declare an `XNamespace` and combine it with `+`:

``` csharp
XNamespace inv = "urn:example:invoice";          // (1)!
XNamespace p   = "urn:example:party";

string id    = (string)  root.Element(inv + "id");        // "INV-42"
decimal total = (decimal) root.Element(inv + "total");     // 100.00
string ccy   = (string)  root.Element(inv + "total").Attribute("currency"); // (2)!
string name  = (string)  root.Element(p + "supplier").Element(p + "name");
```

1.  `inv + "id"` builds the fully-qualified `XName` `{urn:example:invoice}id`.
    There are **no prefixes** in this model at all — you always work with the URI,
    which sidesteps the whole [prefix-binding problem](xml-in-code.md#the-namespace-problem-in-every-language)
    by never using prefixes for lookup.
2.  The `currency` attribute is *unprefixed* in the document, so it is in **no
    namespace** — `Attribute("currency")` with a bare string is correct. A
    prefixed attribute would need `Attribute(someNs + "currency")`.

## 1b. Parse — XmlReader (streaming pull)

For large files, `XmlReader` is the forward-only, constant-memory pull reader:

``` csharp
using XmlReader r = XmlReader.Create("invoice.xml");
while (r.Read()) {
    if (r.NodeType == XmlNodeType.Element
            && r.LocalName == "total"
            && r.NamespaceURI == "urn:example:invoice") {   // (1)!
        string ccy = r.GetAttribute("currency");
        string amt = r.ReadElementContentAsString();
        Console.WriteLine($"{amt} {ccy}");
    }
}
```

1.  Match on `LocalName` **+** `NamespaceURI`, never `r.Name` (the prefixed form).
    The [hybrid pattern](xml-in-code.md#three-ways-to-read-xml): when you hit a
    record start, call `XNode.ReadFrom(r)` to materialize just that element as an
    `XElement`, query it, and move on — XPath convenience at streaming memory.

## 2. Navigate — XPath with XmlNamespaceManager

LINQ queries (above) are idiomatic, but when you want actual XPath you use the
`System.Xml.XPath` extensions plus an `XmlNamespaceManager` — .NET's
[prefix → URI map](xml-in-code.md#the-namespace-problem-in-every-language):

``` csharp
var ns = new XmlNamespaceManager(new NameTable());
ns.AddNamespace("i", "urn:example:invoice");      // (1)!
ns.AddNamespace("p", "urn:example:party");

XElement total = doc.XPathSelectElement("/i:invoice/i:total", ns);
string ccy = (string)doc.XPathEvaluate("string(/i:invoice/i:total/@currency)", ns);
```

1.  Again, `i` is *our* nickname for the document's `inv:` namespace. Bind any
    prefix you like to the URI; the document's own prefix is irrelevant to the
    query.

## 3. Validate against XSD

Attach an `XmlSchemaSet` and validate, collecting errors via a callback:

``` csharp
var schemas = new XmlSchemaSet();
schemas.Add("urn:example:invoice", "invoice.xsd");        // (1)!

XDocument doc = XDocument.Load("invoice.xml");
doc.Validate(schemas, (sender, e) =>
    Console.WriteLine($"{e.Severity} {e.Exception.LineNumber}: {e.Message}"));  // (2)!
```

1.  `Add(targetNamespace, file)` — the set resolves
    [`xs:import`/`xs:include`](../xsd/modular-schemas.md) between member schemas.
2.  The handler is called **once per error** and lets validation continue, so you
    get the full list with line numbers — the same first layer of the
    [validation pipeline](../einvoicing/validation-pipeline.md).

## 4. Transform — XSLT

.NET ships `XslCompiledTransform`. Note: it is **XSLT 1.0** only. For 2.0/3.0 you
need a third-party engine (Saxon has a .NET build, `Saxon-HE` via `SaxonCS`).
As always, **compile once, run many**:

``` csharp
var xslt = new XslCompiledTransform();
xslt.Load("to-fo.xsl");                       // compile once, reuse
xslt.Transform("invoice.xml", "invoice.fo");  // run per input -> XSL-FO
```

The output is the [XSL-FO](xsl-fo-fop.md) document; hand it to Apache FOP for PDF.

## 5. Data binding — XmlSerializer

Generate classes from the XSD (`xsd.exe invoice.xsd /classes`) or annotate:

``` csharp
[XmlRoot("invoice", Namespace = "urn:example:invoice")]
public class Invoice {
    [XmlElement("id",    Namespace = "urn:example:invoice")] public string Id;
    [XmlElement("total", Namespace = "urn:example:invoice")] public decimal Total;
}

var ser = new XmlSerializer(typeof(Invoice));
using var fs = File.OpenRead("invoice.xml");
var inv = (Invoice)ser.Deserialize(fs);       // XML  -> object
ser.Serialize(Console.Out, inv);              // object -> XML
```

!!! tip "`XmlSerializer` vs `System.Text.Json` instincts"
    Coming from JSON, the namespace attributes feel heavy — but they are doing
    real work: without `Namespace =`, the serializer emits/expects *unqualified*
    elements and silently fails to match our `inv:`-qualified document. The
    namespace is not decoration; it is part of the element's identity.

## .NET cheat-sheet

| Task | API |
| --- | --- |
| Tree parse | `XDocument.Load` (LINQ-to-XML) / `XmlDocument` (legacy DOM) |
| Streaming | `XmlReader` (pull) / `XmlWriter` |
| Names | `XNamespace` + `XName` (no prefixes), via `+` |
| XPath | `XPathSelectElement` + `XmlNamespaceManager` |
| Validate | `XmlSchemaSet` + `XDocument.Validate` |
| XSLT 1.0 | `XslCompiledTransform` |
| XSLT 2/3 | Saxon for .NET |
| Binding | `XmlSerializer` (`xsd.exe`) |

Compare with [Java](api-java.md), [Python](api-python.md) and [Rust](api-rust.md).
