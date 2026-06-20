---
icon: fontawesome/brands/rust
---

# APIs: Rust

Rust's XML story is different in kind from [Java's](api-java.md) or
[.NET's](api-dotnet.md). There is no single batteries-included `System.Xml`;
instead there is a set of focused crates, and the ecosystem is **streaming-first**
and **parsing-first**. Full [XSLT](../xslt/index.md) 2.0/3.0 and schema-aware
processing are *not* available in pure Rust — for those you reach across an FFI
boundary to libxml2/libxslt or to [Saxon-C](api-java.md#saxon-beyond-the-jvm-saxon-c-saxonche).
This page works the [five tasks](xml-in-code.md) on the shared
[`invoice.xml`](xml-in-code.md#the-running-example), and is honest about which
ones Rust does natively.

!!! info "This site's own tooling is Rust"
    The [`unxml`](https://github.com/vivainio/unxml-rs) renderings shown on every
    page in this section are produced by a Rust program built on **`quick-xml`** —
    the same crate in task 1 below. The flattened pseudocode you have been reading
    is a streaming pull-parser walking the document and re-emitting it. Real-world
    Rust XML, in other words, is exactly this page.

| Crate | Role |
| --- | --- |
| `quick-xml` | fast streaming **pull** reader + writer; serde support |
| `roxmltree` | read-only **tree**, namespace-aware navigation |
| `sxd-document` + `sxd-xpath` | pure-Rust DOM + **XPath 1.0** |
| `libxml` | bindings to **libxml2**: tree, XPath, **XSD validation** |
| `serde` + `quick-xml::de` | data **binding** to structs |

## 1. Parse — `quick-xml` (streaming pull)

The idiomatic Rust reader is a forward-only pull parser at constant memory. Use
the **namespace-resolving** variant so you match on URIs, not prefixes:

``` rust
use quick_xml::events::Event;
use quick_xml::name::ResolveResult::Bound;
use quick_xml::reader::Reader;

let mut reader = Reader::from_file("invoice.xml")?;
let mut buf = Vec::new();
loop {
    match reader.read_resolved_event_into(&mut buf)? {      // (1)!
        (Bound(ns), Event::Start(e))
            if ns.as_ref() == b"urn:example:invoice"        // (2)!
                && e.local_name().as_ref() == b"total" =>
        {
            for attr in e.attributes().flatten() {
                if attr.key.as_ref() == b"currency" {
                    println!("currency = {}", attr.unescape_value()?);
                }
            }
        }
        (_, Event::Eof) => break,
        _ => {}
    }
    buf.clear();                                            // (3)!
}
```

1.  `read_resolved_event_into` returns a `(ResolveResult, Event)` pair — the parser
    has already resolved the element's prefix to a namespace **URI** for you.
2.  So the match condition compares the *URI* (`urn:example:invoice`) and the
    *local name* (`total`) — never the prefixed `inv:total`. This is the
    [namespace rule](xml-in-code.md#the-namespace-problem-in-every-language) made
    unavoidable: in `quick-xml` the prefix is just bytes you were handed, the URI
    is the identity.
3.  Reusing and clearing `buf` is why this stays allocation-light — the engine
    behind `quick-xml`'s speed, and behind `unxml`.

## 2. Navigate — a tree, two ways

### `roxmltree` — read-only, namespace-aware

When you want random access rather than a stream, `roxmltree` parses into a
borrowed tree and exposes namespaces directly:

``` rust
let text = std::fs::read_to_string("invoice.xml")?;
let doc = roxmltree::Document::parse(&text)?;

let total = doc.descendants()
    .find(|n| n.tag_name().namespace() == Some("urn:example:invoice")  // (1)!
           && n.tag_name().name() == "total")
    .unwrap();

println!("{}", total.text().unwrap());          // 100.00
println!("{:?}", total.attribute("currency"));  // Some("EUR")
```

1.  `tag_name()` is a `(namespace, local-name)` pair — you filter on the URI, the
    same discipline as the streaming reader. `roxmltree` is fast and read-only by
    design; for *mutation* you would use `libxml`'s tree or build output with
    `quick-xml`'s writer.

### `sxd-xpath` — actual XPath (1.0)

If you want real [XPath](../xpath/index.md) expressions in pure Rust, `sxd-xpath`
provides them — with a namespace-prefix map, just like everywhere else:

``` rust
use sxd_document::parser;
use sxd_xpath::{Context, Factory};

let package = parser::parse(&text)?;
let document = package.as_document();

let xpath = Factory::new().build("/i:invoice/i:total")?.unwrap();
let mut ctx = Context::new();
ctx.set_namespace("i", "urn:example:invoice");      // (1)!

let value = xpath.evaluate(&ctx, document.root())?;
println!("{}", value.into_string());                // 100.00
```

1.  `i` is *our* prefix bound to the document's URI — the file's `inv:` is
    irrelevant, as on every other runtime. Note the ceiling: `sxd-xpath` (and
    libxml2) implement **XPath 1.0** only. There is no pure-Rust XPath 2.0/3.1.

## 3. Validate against XSD — via `libxml` (libxml2)

Pure Rust has no XSD validator. The practical route is the `libxml` crate, which
binds the same **libxml2** that backs [Python's lxml](api-python.md):

``` rust
use libxml::parser::Parser;
use libxml::schemas::{SchemaParserContext, SchemaValidationContext};

let doc = Parser::default().parse_file("invoice.xml")?;

let mut schema_parser = SchemaParserContext::from_file("invoice.xsd");
let mut validator = SchemaValidationContext::from_parser(&mut schema_parser)
    .expect("schema compiled");

match validator.validate_document(&doc) {
    Ok(()) => println!("valid"),
    Err(errors) => for e in errors {                    // (1)!
        println!("{}", e.message.unwrap_or_default());
    }
}
```

1.  Because this *is* libxml2 under the hood, it resolves
    [`xs:import`/`xs:include`](../xsd/modular-schemas.md) and reports the same
    diagnostics you would get from lxml — it is the first layer of the
    [validation pipeline](../einvoicing/validation-pipeline.md), reached through an
    FFI binding rather than a native implementation.

## 4. Transform — XSLT is *not* native

This is the honest gap. There is **no maintained pure-Rust XSLT engine**, for any
version. Two real options:

- **libxslt via FFI** — XSLT **1.0** only (libxslt never implemented 2.0+). Usable
  from the `libxml`/`libxslt` C bindings if 1.0 is enough.
- **Saxon-C / SaxonCHE** — to run the [modern XSLT 3.0](../xslt/moving-to-3.md) this
  site teaches, call Saxon's native library over its C API from Rust via FFI (or
  shell out to the Saxon CLI). Recall from the
  [Java page](api-java.md#saxon-beyond-the-jvm-saxon-c-saxonche) that SaxonCHE *is*
  the Java engine compiled to native code with GraalVM — so a Rust program linking
  it is really driving Saxon-HE, HE limitations and all (no streaming, no
  schema-aware transforms).

``` text
Rust program ──FFI──▶ SaxonCHE (native-compiled Java)  ──▶ XSLT 3.0 result
            └─FFI──▶ libxslt (C)                        ──▶ XSLT 1.0 result
```

For many Rust services the pragmatic answer is to **not** transform in Rust at all
— do the XSLT step in a Java/Python sidecar, and keep Rust for the high-throughput
parse/validate/route work it excels at.

## 5. Data binding — `serde` + `quick-xml`

Rust's binding story is `serde`, via `quick-xml`'s deserializer. Attributes use a
`@` prefix and element text uses `$text`:

``` rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Invoice {
    id: String,
    total: Total,
}
#[derive(Debug, Deserialize)]
struct Total {
    #[serde(rename = "@currency")]
    currency: String,
    #[serde(rename = "$text")]
    amount: String,
}

let inv: Invoice = quick_xml::de::from_str(&text)?;   // (1)!
println!("{} {}", inv.total.amount, inv.total.currency);
```

1.  Convenient, but with a real caveat that ties back to this whole section:
    `serde` matching keys on **local names** is awkward with namespaces. Unlike
    [JAXB](api-java.md#5-data-binding-jaxb) or
    [.NET's `XmlSerializer`](api-dotnet.md#5-data-binding-xmlserializer), there is
    no first-class "this field is in namespace *U*" annotation — multi-namespace or
    [extension-heavy](atom-feeds.md) documents push you back to the
    `quick-xml`/`roxmltree` element APIs. Binding shines for single-namespace,
    schema-stable documents.

## Rust cheat-sheet

| Task | Crate / approach |
| --- | --- |
| Streaming parse | `quick-xml` (`read_resolved_event_into`) |
| Tree navigate | `roxmltree` (read-only, ns-aware) |
| XPath 1.0 | `sxd-xpath` (prefix map) or `libxml` |
| XPath 2.0/3.1 | **none** (not available in Rust) |
| Validate XSD | `libxml` (libxml2 binding) |
| XSLT 1.0 | libxslt via FFI |
| XSLT 2/3 | Saxon-C / SaxonCHE via FFI (no native) |
| Binding | `serde` + `quick-xml::de` (weak on namespaces) |

!!! note "The shape of Rust XML"
    Rust gives you **the fastest, lowest-memory parse and write** of any runtime
    here, plus XPath 1.0 and XSD through libxml2 — but the *declarative* upper
    layers (XSLT 2/3, schema-aware processing) live behind FFI. The takeaway: use
    Rust for ingest, validation and routing at volume; cross to Saxon for
    transformation. That is the same division of labour `unxml` itself follows —
    a fast Rust front end over the XML, nothing more.

Compare with [Java](api-java.md), [.NET](api-dotnet.md) and [Python](api-python.md).
