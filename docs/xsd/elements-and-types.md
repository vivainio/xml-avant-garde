# Elements and types

An XSD schema describes what a valid document looks like. Almost everything you
write in a schema comes down to one decision, made over and over: *what type is
this element?* And every type in XSD falls into one of exactly two camps —
**simple** or **complex**. Get that split clear and the rest of the language
follows.

The running document is a UBL-style invoice. It carries a few plain values
(`cbc:ID`, `cbc:IssueDate`, `cbc:DocumentCurrencyCode`) and some structured
parts (`cac:Party`, `cac:InvoiceLine`):

``` xml title="invoice.xml" linenums="1"
<Invoice
    xmlns="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2"
    xmlns:cbc="urn:oasis:names:specification:ubl:schema:xsd:CommonBasicComponents-2"
    xmlns:cac="urn:oasis:names:specification:ubl:schema:xsd:CommonAggregateComponents-2">
  <cbc:ID>INV-001</cbc:ID>
  <cbc:IssueDate>2024-03-01</cbc:IssueDate>
  <cbc:DocumentCurrencyCode>EUR</cbc:DocumentCurrencyCode>
  <cac:AccountingSupplierParty>
    <cac:Party>
      <cbc:Name>Example Seller Ltd</cbc:Name>
    </cac:Party>
  </cac:AccountingSupplierParty>
  <cac:InvoiceLine>
    <cbc:ID>1</cbc:ID>
    <cbc:LineExtensionAmount currencyID="EUR">100.00</cbc:LineExtensionAmount>
  </cac:InvoiceLine>
</Invoice>
```

## Declaring an element

You declare an element with `xsd:element`. A `name` says what it is called; a
`type` says what may go inside it:

``` xml title="declaring an element" linenums="1"
<xsd:element name="IssueDate" type="xsd:date"/>
```

That single line means: *an element called `IssueDate` whose content must be a
valid date.* No children, no attributes — just a date value such as
`2024-03-01`. The type does all the work.

## The two kinds of type

Every type is either **simple** or **complex**, and the difference is exactly
about *structure*:

| Kind | May have children? | May have attributes? | Holds |
| --- | --- | --- | --- |
| **simple type** | no | no | a text/value, e.g. a string, number, or date |
| **complex type** | yes | yes | child elements and/or attributes |

A **simple type** describes a leaf: text or a value with nothing nested.
`cbc:IssueDate` above is simple — its content is a date and that is all. A
**complex type** describes structure: the `Invoice` element is complex because
it contains child elements, and `cbc:LineExtensionAmount` is complex because —
although its *content* is just a number — it also carries a `currencyID`
attribute, and **the moment an element has an attribute, its type must be
complex.**

!!! info "Attributes force a complex type"
    A simple type can hold a value *or* it can be reused as the content of a
    complex type, but it can never itself carry an attribute. `100.00` on its own
    is simple; `<cbc:LineExtensionAmount currencyID="EUR">100.00</...>` needs a
    complex type with *simple content* — a value plus an attribute. That pattern
    is covered in [Complex types](complex-types.md).

## Built-in datatypes tour

XSD ships with a large library of simple types you can use without defining
anything. These are the ones you meet constantly in invoice schemas:

| Type | Holds | Example |
| --- | --- | --- |
| `xsd:string` | any text, whitespace preserved | `Example Seller Ltd` |
| `xsd:normalizedString` | text with tabs/newlines collapsed to spaces | `INV 001` |
| `xsd:decimal` | an exact decimal number — use for **money** | `100.00` |
| `xsd:integer` | a whole number, no fraction | `42` |
| `xsd:boolean` | `true`/`false` (also `1`/`0`) | `true` |
| `xsd:date` | a calendar date, format **`YYYY-MM-DD`** | `2024-03-01` |
| `xsd:dateTime` | a date and time | `2024-03-01T09:30:00Z` |
| `xsd:anyURI` | a URI/IRI | `urn:oasis:names:specification:ubl:schema:xsd:Invoice-2` |

!!! tip "Use `xsd:decimal` for amounts, not a float"
    `xsd:decimal` represents the value exactly, so `100.00` stays `100.00`. The
    binary floating types (`xsd:float`, `xsd:double`) can introduce rounding
    error and are wrong for monetary values. Likewise prefer `xsd:date` for a
    calendar date like an issue date, and `xsd:dateTime` only when a time of day
    genuinely matters.

## Named vs anonymous types

A type can be **named** — declared once and referred to by name — or
**anonymous**, written inline inside the element that uses it.

A **named** type is defined at the top level and referenced with `type="..."`.
This is how UBL itself is built: `cbc:IssueDate` references a named, shared date
type, so every date element across the schema behaves identically:

``` xml title="named type, referenced by type=" linenums="1"
<!-- definition, written once -->
<xsd:simpleType name="DateType">           <!-- (1)! -->
  <xsd:restriction base="xsd:date"/>
</xsd:simpleType>

<!-- reference, reusable anywhere -->
<xsd:element name="IssueDate" type="DateType"/>   <!-- (2)! -->
```

1. A global, named simple type — defined once, given a name.
2. The element just *points* at the type by name. Any number of elements can
   share `DateType`.

An **anonymous** type is nested directly inside the element and has no `name`. It
exists only for that one element — there is nothing to reference it by:

``` xml title="anonymous type, nested inline" linenums="1"
<xsd:element name="IssueDate">
  <xsd:simpleType>                  <!-- (1)! -->
    <xsd:restriction base="xsd:date"/>
  </xsd:simpleType>
</xsd:element>
```

1. No `name`, no `type=` on the element — the type lives *inside* the element and
   applies to nothing else.

!!! note "When to use which"
    Reach for a **named** type when the same shape is used in more than one place
    (UBL's reusable `cbc:` and `cac:` components are all named). Reach for an
    **anonymous** type for a genuinely one-off, local structure that would only
    add clutter as a named definition.

The same choice applies to complex types. The top-level `Invoice` element has a
complex type, and a schema author may write it either way — as a named type:

``` xml title="Invoice with a named complex type" linenums="1"
<xsd:element name="Invoice" type="InvoiceType"/>

<xsd:complexType name="InvoiceType">
  <!-- child elements declared here -->
</xsd:complexType>
```

…or as an anonymous complex type nested inside the element:

``` xml title="Invoice with an anonymous complex type" linenums="1"
<xsd:element name="Invoice">
  <xsd:complexType>
    <!-- child elements declared here -->
  </xsd:complexType>
</xsd:element>
```

<div class="xslt-result" markdown>
Both forms accept the same `Invoice` documents. The named version lets another
schema reuse `InvoiceType`; the anonymous version keeps it private to this one
element.
</div>

## Global vs local declarations

*Where* an `xsd:element` sits also matters. An element declared as a direct child
of `xsd:schema` is **global**: it can be the document's root, and it can be
reused elsewhere by reference rather than by repeating its declaration. An
element declared inside a type is **local** to that type.

``` xml title="global vs local" linenums="1"
<xsd:schema xmlns:xsd="http://www.w3.org/2001/XMLSchema">

  <xsd:element name="ID" type="xsd:normalizedString"/>   <!-- (1)! -->

  <xsd:complexType name="InvoiceType">
    <xsd:sequence>
      <xsd:element ref="ID"/>                            <!-- (2)! -->
      <xsd:element name="IssueDate" type="xsd:date"/>    <!-- (3)! -->
    </xsd:sequence>
  </xsd:complexType>

</xsd:schema>
```

1. **Global** — a direct child of `xsd:schema`. Reusable and a candidate root.
2. **Reference** — `ref=` points at the global `ID` declaration instead of
   redeclaring it.
3. **Local** — declared inline, visible only inside `InvoiceType`.

The `ref=` mechanism, and how `minOccurs`/`maxOccurs` control how *many* times a
child may appear, belong with structure — see
[Complex types](complex-types.md).

## Grounding it in UBL

Putting the pieces against the running invoice:

- `cbc:ID` (`INV-001`) is a **simple-content** element — text, no children.
- `cbc:IssueDate` (`2024-03-01`) is based on **`xsd:date`**, format `YYYY-MM-DD`.
- `cbc:DocumentCurrencyCode` (`EUR`) is a simple string-based value.
- `cbc:LineExtensionAmount` (`100.00` with `currencyID="EUR"`) is **complex** —
  a decimal value *plus* an attribute.
- the top-level `Invoice` is a **complex type**, holding the `cbc:` and `cac:`
  children above.

<div class="xslt-result" markdown>
Read top to bottom, an invoice is mostly simple-typed leaves (`cbc:ID`,
`cbc:IssueDate`, `cbc:DocumentCurrencyCode`) gathered inside complex-typed
containers (`Invoice`, `cac:Party`, `cac:InvoiceLine`).
</div>

## Next

You have the type *system*; next comes putting elements *together*. [Complex
types](complex-types.md) shows how `xsd:sequence`, `ref=`, cardinality
(`minOccurs`/`maxOccurs`), and attributes assemble the simple leaves above into
the `Invoice` structure.
