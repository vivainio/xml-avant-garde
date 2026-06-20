---
icon: lucide/coffee
---

# APIs: Java (JAXP & Saxon)

Java is where DOM, SAX, and StAX all became standards, so its XML story is the
most complete — and the most layered. The built-in API is **JAXP** (Java API for
XML Processing); for modern [XSLT](../xslt/index.md) (2.0/3.0) you add **Saxon**.
This page works the [five tasks](xml-in-code.md) on the shared
[`invoice.xml`](xml-in-code.md#the-running-example).

!!! info "Packages"
    Everything except Saxon is in the JDK: `javax.xml.parsers` (DOM/SAX),
    `javax.xml.stream` (StAX), `javax.xml.xpath`, `javax.xml.validation`,
    `javax.xml.transform`. Saxon (`net.sf.saxon:Saxon-HE`) is a separate
    dependency.

## 1. Parse — DOM

``` java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setNamespaceAware(true);                       // (1)!
Document doc = dbf.newDocumentBuilder()
                  .parse(new File("invoice.xml"));
Element root = doc.getDocumentElement();           // <inv:invoice>
```

1.  **The most important line on this page.** JAXP's factories default to
    namespace-*unaware* for backward compatibility. Forget `setNamespaceAware(true)`
    and `getLocalName()` returns `null`, namespace-aware XPath silently fails, and
    you lose an afternoon. Always set it.

## 1b. Parse — StAX (streaming pull)

For files too big for DOM, pull events one at a time at constant memory:

``` java
XMLInputFactory f = XMLInputFactory.newInstance();
XMLStreamReader r = f.createXMLStreamReader(new FileInputStream("invoice.xml"));
while (r.hasNext()) {
    if (r.next() == XMLStreamConstants.START_ELEMENT
            && "total".equals(r.getLocalName())) {        // (1)!
        String ccy = r.getAttributeValue(null, "currency");
        String amt = r.getElementText();
        System.out.println(amt + " " + ccy);
    }
}
```

1.  Streaming code compares `getLocalName()` (and, when it matters,
    `getNamespaceURI()`) — never the prefixed name, because the prefix is the
    author's choice, not yours. This is the [namespace rule](xml-in-code.md#the-namespace-problem-in-every-language)
    in its streaming form.

The [hybrid pattern](xml-in-code.md#three-ways-to-read-xml) — stream to each
record, then `XMLStreamReader`→DOM with a `Transformer` for just that subtree —
is the standard way to process multi-GB files with XPath convenience.

## 2. Navigate — XPath with a NamespaceContext

This is the part everyone gets wrong first. XPath needs a **prefix → URI** map,
supplied as a `NamespaceContext`:

``` java
XPath xp = XPathFactory.newInstance().newXPath();
xp.setNamespaceContext(new NamespaceContext() {         // (1)!
    public String getNamespaceURI(String prefix) {
        return switch (prefix) {
            case "i" -> "urn:example:invoice";          // (2)!
            case "p" -> "urn:example:party";
            default  -> XMLConstants.NULL_NS_URI;
        };
    }
    public String getPrefix(String uri) { return null; }
    public Iterator<String> getPrefixes(String uri) { return null; }
});

String total = xp.evaluate("/i:invoice/i:total", doc);  // "100.00"
String ccy   = xp.evaluate("/i:invoice/i:total/@currency", doc);  // "EUR"
```

1.  The interface is clunky (three methods, two usually unused). Most projects use
    a small reusable implementation, or Saxon's `s9api` which takes a plain map.
2.  `i` is *our* prefix bound to the document's URI. The document writes `inv:`;
    the query writes `i:`; they meet at the URI. Querying `/inv:invoice` here
    would throw — `inv` is unbound in *our* context.

## 3. Validate against XSD

``` java
SchemaFactory sf = SchemaFactory.newInstance(XMLConstants.W3C_XML_SCHEMA_NS_URI);
Schema schema = sf.newSchema(new File("invoice.xsd"));   // (1)!
Validator v = schema.newValidator();
try {
    v.validate(new StreamSource(new File("invoice.xml")));
    System.out.println("valid");
} catch (SAXParseException e) {
    System.out.println("line " + e.getLineNumber() + ": " + e.getMessage());  // (2)!
}
```

1.  `newSchema` can take an array of sources to load several
    [modular schemas](../xsd/modular-schemas.md) at once; `xs:import` between them
    resolves automatically.
2.  For a *list* of all errors rather than failing on the first, install an
    `ErrorHandler` on the `Validator` that collects instead of throwing.

## 4. Transform — XSLT with Saxon

The JDK's built-in transformer is XSLT **1.0 only**. For 2.0/3.0 (the
[modern XSLT](../xslt/moving-to-3.md) this site teaches) use Saxon — and **compile
once, run many**:

``` java
Processor proc = new Processor(false);                 // Saxon s9api, HE edition
XsltCompiler comp = proc.newXsltCompiler();
XsltExecutable exec = comp.compile(new StreamSource(new File("to-fo.xsl"))); // (1)!

Xslt30Transformer t = exec.load30();
t.transform(new StreamSource(new File("invoice.xml")),
            proc.newSerializer(new File("invoice.fo")));   // (2)!
```

1.  Compile the stylesheet **once** and reuse `exec` across thousands of inputs —
    compilation is the expensive step.
2.  The output here is the [XSL-FO](xsl-fo-fop.md) we met earlier; pipe it to
    Apache FOP to get a PDF. Saxon's `s9api` namespace handling is also far nicer
    than JAXP's: `XPathCompiler.declareNamespace("i", "urn:example:invoice")`.

!!! warning "What Saxon-HE leaves out"
    The snippet above uses **Saxon-HE** (Home Edition) — the free, open-source
    (MPL 2.0) build, and the right default. It runs the full **XSLT 3.0 / XPath 3.1
    / XQuery 3.1** languages, including maps, arrays, higher-order functions and
    packages. But three capabilities are gated behind the commercial **PE**
    (Professional) and **EE** (Enterprise) editions, and two of them are exactly
    the ones this site cares about:

    - **No streaming.** XSLT 3.0 *streaming* (`xsl:stream`, `xsl:mode
      streamable="yes"`) — the headline feature for transforming files too big for
      memory — is **EE-only**. In HE the whole input is built as a tree, so the
      [hybrid streaming pattern](xml-in-code.md#three-ways-to-read-xml) at the
      `XMLStreamReader` level is your fallback for huge inputs.
    - **No schema-awareness.** Validating against (and carrying the *types* from) an
      [XSD](../xsd/index.md) inside a transform — schema-aware XSLT/XQuery — is
      **EE-only**. HE transforms are always *untyped*; validate separately with
      JAXP's `Validator` (task 3 above).
    - **Fewer optimizations.** Bytecode generation, multi-threaded `xsl:for-each`,
      join optimization and document projection are **PE/EE**. For most workloads
      HE is plenty fast; these matter at scale.

    So: reach for HE first; you only need a paid edition when you hit *streaming
    transforms* or *schema-aware* processing specifically.

### Saxon beyond the JVM: Saxon-C / SaxonCHE

The same Saxon engine runs **outside Java**. *Saxon-C* is the Java codebase
**compiled to a native library** with GraalVM `native-image`, exposing a C/C++
API with bindings for Python (`saxonche` on PyPI — used on the
[Python page](api-python.md)), PHP, and, via FFI, languages like
[Rust](api-rust.md). **SaxonCHE** is the Home-Edition build of Saxon-C, so it
inherits exactly the HE limitations above — same XSLT 3.0 support, same
no-streaming / no-schema-aware ceiling — just without a JVM in the picture.

## 5. Data binding — JAXB

For stable schemas, skip nodes and bind to classes. Generate them from the XSD
(`xjc invoice.xsd`) or annotate by hand:

``` java
@XmlRootElement(namespace = "urn:example:invoice", name = "invoice")
@XmlAccessorType(XmlAccessType.FIELD)
class Invoice {
    @XmlElement(namespace = "urn:example:invoice") String id;
    @XmlElement(namespace = "urn:example:invoice") BigDecimal total;
}

JAXBContext ctx = JAXBContext.newInstance(Invoice.class);
Invoice inv = (Invoice) ctx.createUnmarshaller()
                           .unmarshal(new File("invoice.xml"));   // XML  -> object
ctx.createMarshaller().marshal(inv, System.out);                  // object -> XML
```

!!! note "JAXB moved out of the JDK"
    Since Java 11, JAXB is no longer bundled — add `jakarta.xml.bind:jakarta.xml.bind-api`
    plus a runtime (`org.glassfish.jaxb:jaxb-runtime`). The annotations also
    migrated from `javax.xml.bind.*` to `jakarta.xml.bind.*`.

## Java cheat-sheet

| Task | API |
| --- | --- |
| DOM parse | `DocumentBuilderFactory` (**`setNamespaceAware(true)`**) |
| Streaming | `XMLStreamReader` (StAX pull) / `SAXParser` (push) |
| XPath | `XPathFactory` + `NamespaceContext` (or Saxon `s9api`) |
| Validate | `SchemaFactory` + `Validator` |
| XSLT 1.0 | `TransformerFactory` (built-in) |
| XSLT 2/3 | **Saxon** `Processor` / `XsltCompiler` |
| Binding | JAXB (`xjc`, `@Xml*`) |

Compare with [.NET](api-dotnet.md), [Python](api-python.md) and [Rust](api-rust.md).
