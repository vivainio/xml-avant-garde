---
icon: lucide/shield-check
---

# XSD

**XSD** (XML Schema Definition) is a language for *describing the allowed
structure and data types* of a class of XML documents, and for *validating*
instances against that description. A schema answers questions like: which
elements may appear, in what order, how many times, what attributes they carry,
and what kind of data their content holds.

Unlike a DTD, an XSD is **itself an XML document**. That brings three concrete
advantages:

- **Rich data types** — `xs:integer`, `xs:date`, `xs:decimal` and dozens more,
  plus the ability to derive your own with length, range, pattern and
  enumeration constraints.
- **Namespace support** — schemas declare exactly which namespace their
  vocabulary belongs to, so vocabularies can be mixed without name clashes.
- **Tooling** — because a schema is plain XML, the same parsers, editors and
  transformation tools you already use apply to it.

!!! note "A schema does not validate by itself"
    The schema is a *description*. The actual checking is done by a **validating
    parser** that reads both the instance and the schema and reports whether the
    instance conforms. The schema is data, not a program.

## The schema namespace and the `<xsd:schema>` root

Every XSD is built from elements in the W3C schema namespace
`http://www.w3.org/2001/XMLSchema`, conventionally bound to the prefix `xs:`
(or `xsd:`). The document root is `<xsd:schema>`.

``` xml title="schema-skeleton.xsd"
<xsd:schema
    xmlns:xsd="http://www.w3.org/2001/XMLSchema"
    targetNamespace="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2"  <!-- (1)! -->
    xmlns="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2"
    elementFormDefault="qualified">                                          <!-- (2)! -->
  <!-- element and type declarations go here -->
</xsd:schema>
```

1.  **`targetNamespace`** is the namespace of the *vocabulary you are defining* —
    the namespace that conforming instances will use. Here it is the UBL Invoice
    namespace.
2.  **`elementFormDefault="qualified"`** means locally declared elements must be
    namespace-qualified in instances (they live in the target namespace). UBL
    schemas use `qualified`, which is why instance elements are bound to the UBL
    namespaces rather than appearing unqualified.

!!! tip "`xs:` vs `xsd:`"
    Both prefixes are pure convention — only the namespace URI matters. This
    section uses `xsd:` in schema documents and `xsi:` for the instance helper
    namespace described below.

## How an instance associates with a schema

XML instances can *point* at the schema that describes them using attributes
from the XML Schema **instance** namespace
`http://www.w3.org/2001/XMLSchema-instance`, conventionally `xsi:`:

- **`xsi:schemaLocation`** — pairs of `namespace schema-file` for vocabularies
  that *have* a target namespace.
- **`xsi:noNamespaceSchemaLocation`** — a single schema file for documents with
  no namespace.

``` xml title="instance-with-hint.xml"
<Invoice
    xmlns="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2 UBL-Invoice-2.1.xsd">
  ...
</Invoice>
```

These attributes are only **hints**. In practice validators are very often told
which schema to use **out-of-band** (configured in the tool or pipeline) and
ignore any in-document hint — which is the norm for UBL processing, where the
official schema set is fixed and known in advance.

## Motivating example: a UBL Invoice

This section uses **OASIS UBL** (Universal Business Language) as its running
real-world example. A UBL Invoice is a genuine, large, XSD-defined vocabulary —
exactly the kind of thing schemas exist to govern. Rather than meet it all at
once, we build up from the small pieces: individual elements and types, then
aggregates, then constrained values, then how the whole thing is assembled from
modules.

Here is a small UBL-shaped invoice instance. **This section explains the schema
that makes this valid.**

``` xml title="invoice.xml"
<Invoice xmlns="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2"
         xmlns:cac="urn:oasis:names:specification:ubl:schema:xsd:CommonAggregateComponents-2"
         xmlns:cbc="urn:oasis:names:specification:ubl:schema:xsd:CommonBasicComponents-2">
  <cbc:ID>INV-001</cbc:ID>
  <cbc:IssueDate>2026-06-20</cbc:IssueDate>
  <cbc:DocumentCurrencyCode>EUR</cbc:DocumentCurrencyCode>
  <cac:AccountingSupplierParty>
    <cac:Party>
      <cac:PartyName><cbc:Name>Acme Records</cbc:Name></cac:PartyName>
    </cac:Party>
  </cac:AccountingSupplierParty>
  <cac:InvoiceLine>
    <cbc:ID>1</cbc:ID>
    <cbc:LineExtensionAmount currencyID="EUR">10.90</cbc:LineExtensionAmount>
  </cac:InvoiceLine>
</Invoice>
```

Notice the three namespaces already at work:

- the **Invoice** namespace for the document element,
- **`cbc:`** — *Common Basic Components*, the small leaf values like
  `cbc:ID`, `cbc:IssueDate` and `cbc:LineExtensionAmount`,
- **`cac:`** — *Common Aggregate Components*, the structured groupings like
  `cac:Party` and `cac:InvoiceLine`.

That split between *basic* and *aggregate* components is the heart of how UBL is
layered, and we return to it in [Modular schemas](modular-schemas.md).

<div class="xslt-result" markdown>
**Validation outcome:** the instance above is *valid* against the UBL Invoice
schema set — every element is declared, appears in the expected order and
cardinality, and every value matches its declared type (`cbc:IssueDate` is a
date, `cbc:LineExtensionAmount` is a decimal carrying a `currencyID`
attribute).
</div>

!!! info "What makes it *invalid*?"
    Swap `<cbc:IssueDate>2026-06-20</cbc:IssueDate>` for
    `<cbc:IssueDate>20 June 2026</cbc:IssueDate>` and a validator rejects it:
    the content no longer matches the `xs:date` lexical form. Reorder elements so
    `cbc:IssueDate` precedes `cbc:ID`, and it fails too — the schema fixes the
    sequence.

## A note on XSD versions

This section teaches **XSD 1.0**, which is effectively universal and is what the
UBL schema set is written in. **XSD 1.1** is a later, backward-compatible
revision that adds — among other things — **assertions** (`xs:assert`, embedding
XPath rules inside a type) and **conditional type assignment** (choosing a type
based on attribute values). They are powerful but unevenly supported; everything
in this tutorial works in plain 1.0.

## Related sections

XSD describes and constrains documents; its sibling standards work on the same
documents from other angles. Once a document is known to be valid, you typically
**query** it with [XPath](../xpath/index.md) and **transform** it with the
[XSLT Tutorial](../xslt/index.md) — and both of those rely on exactly the
namespace discipline introduced here.

## Where to go next

1.  [Elements and types](elements-and-types.md) — declaring elements; simple vs
    complex; built-in types.
2.  [Complex types](complex-types.md) — sequences, cardinality, attributes,
    building aggregates.
3.  [Simple types and restrictions](simple-types-restrictions.md) — facets,
    enumerations, patterns, code lists.
4.  [Modular schemas](modular-schemas.md) — `import`/`include` and how UBL is
    layered.
