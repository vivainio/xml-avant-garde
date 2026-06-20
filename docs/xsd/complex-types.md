# Complex types

A **complex type** describes an element that has child elements, attributes, or
both. Everything in a UBL document — from the `Invoice` root down to a single
`cac:PartyName` — is an instance of some complex type. This page builds a
UBL-style invoice from its pieces, starting with content models and working up
to real aggregates.

The running example is a small UBL invoice. The three standard namespaces are:

``` xml title="invoice.xml" linenums="1"
<Invoice
    xmlns="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2"
    xmlns:cbc="urn:oasis:names:specification:ubl:schema:xsd:CommonBasicComponents-2"
    xmlns:cac="urn:oasis:names:specification:ubl:schema:xsd:CommonAggregateComponents-2">
  <cbc:ID>INV-001</cbc:ID>
  <cbc:IssueDate>2026-06-20</cbc:IssueDate>
  <cbc:DocumentCurrencyCode>EUR</cbc:DocumentCurrencyCode>
  <cac:AccountingSupplierParty>
    <cac:Party>
      <cac:PartyName>
        <cbc:Name>Example Seller Ltd</cbc:Name>
      </cac:PartyName>
    </cac:Party>
  </cac:AccountingSupplierParty>
  <cac:InvoiceLine>
    <cbc:ID>1</cbc:ID>
    <cbc:LineExtensionAmount currencyID="EUR">10.90</cbc:LineExtensionAmount>
  </cac:InvoiceLine>
</Invoice>
```

- `cbc:` (**Common Basic Components**) holds leaf values like `cbc:ID` and
  `cbc:Name`.
- `cac:` (**Common Aggregate Components**) holds containers like `cac:Party`
  and `cac:InvoiceLine` that nest other components.

## Content models

A complex type's children are governed by a **content model**: a compositor
that says which children may appear and in what order. There are three.

| Compositor | Meaning |
| --- | --- |
| `xsd:sequence` | the listed elements appear **in order** |
| `xsd:choice` | exactly **one of** the listed elements appears |
| `xsd:all` | each listed element appears **at most once**, in **any order** |

UBL uses `xsd:sequence` almost everywhere — document structure is positional, so
the order of `cbc:ID`, `cbc:IssueDate`, and the rest is fixed by the schema:

``` xml title="sequence.xsd" linenums="1"
<xsd:complexType name="MiniInvoiceType">
  <xsd:sequence>                                        <!-- (1)! -->
    <xsd:element ref="cbc:ID"/>
    <xsd:element ref="cbc:IssueDate"/>
    <xsd:element ref="cbc:DocumentCurrencyCode"/>
  </xsd:sequence>
</xsd:complexType>
```

1.  `xsd:sequence` fixes the order: an instance must list `ID`, then
    `IssueDate`, then `DocumentCurrencyCode`. Reordering them is invalid.

`xsd:choice` is for "one of these" — a payment that is either a card or a
transfer, never both:

``` xml title="choice.xsd" linenums="1"
<xsd:complexType name="PaymentType">
  <xsd:choice>
    <xsd:element ref="cbc:CardNumber"/>
    <xsd:element ref="cbc:IBAN"/>
  </xsd:choice>
</xsd:complexType>
```

`xsd:all` lets the children appear in any order, each at most once. It is rare
in UBL because document order matters, but it is the right tool when order
genuinely does not:

``` xml title="all.xsd" linenums="1"
<xsd:complexType name="ContactType">
  <xsd:all>
    <xsd:element ref="cbc:Telephone"/>
    <xsd:element ref="cbc:ElectronicMail"/>
  </xsd:all>
</xsd:complexType>
```

## Cardinality with minOccurs and maxOccurs

Each particle in a content model carries an occurrence range. The defaults are
`minOccurs="1"` and `maxOccurs="1"` — so a plain `<xsd:element>` means *exactly
one*. Two overrides cover almost everything:

- **Optional**: `minOccurs="0"` — may be absent.
- **Repeating**: `maxOccurs="unbounded"` — may appear any number of times.

``` xml title="cardinality.xsd" linenums="1"
<xsd:complexType name="InvoiceType">
  <xsd:sequence>
    <xsd:element ref="cbc:ID"/>                                   <!-- (1)! -->
    <xsd:element ref="cbc:IssueDate"/>
    <xsd:element ref="cbc:Note" minOccurs="0"/>                   <!-- (2)! -->
    <xsd:element ref="cac:InvoiceLine" maxOccurs="unbounded"/>    <!-- (3)! -->
  </xsd:sequence>
</xsd:complexType>
```

1.  No occurrence attributes, so the defaults apply: exactly one `cbc:ID`.
2.  `minOccurs="0"` makes `cbc:Note` optional — an instance may omit it.
3.  `maxOccurs="unbounded"` allows one or more `cac:InvoiceLine` children. (It
    still requires at least one, because `minOccurs` defaults to `1`.)

Mapping back to the instance: the single `<cbc:ID>INV-001</cbc:ID>` satisfies
the required element, an absent `cbc:Note` is fine because it is optional, and a
document may carry many `<cac:InvoiceLine>` blocks under the one repeating
particle.

!!! note "`minOccurs="0" maxOccurs="unbounded"` is the fully optional list"
    Combine the two to allow *zero or more*: an element that may be missing
    entirely or repeated. This is the usual shape for optional collections.

## Referencing global elements with `ref`

So far the sequences have used `ref=` rather than `name=`. This is the heart of
how UBL composes documents. The `cbc:` and `cac:` schemas declare each component
once as a **global element**:

``` xml title="globals.xsd" linenums="1"
<xsd:element name="ID" type="cbc:IdentifierType"/>          <!-- in the cbc schema -->
<xsd:element name="InvoiceLine" type="cac:InvoiceLineType"/> <!-- in the cac schema -->
```

A document type then assembles itself by **referring** to those globals instead
of redeclaring them. The `ref` attribute points at a global element by its
qualified name; the referenced element brings its own name and type:

``` xml title="compose.xsd" linenums="1"
<xsd:complexType name="InvoiceType">
  <xsd:sequence>
    <xsd:element ref="cbc:ID"/>                <!-- (1)! -->
    <xsd:element ref="cbc:IssueDate"/>
    <xsd:element ref="cbc:DocumentCurrencyCode"/>
    <xsd:element ref="cac:AccountingSupplierParty"/>
    <xsd:element ref="cac:InvoiceLine" maxOccurs="unbounded"/>
  </xsd:sequence>
</xsd:complexType>
```

1.  `ref="cbc:ID"` reuses the global `ID` element from the `cbc` namespace —
    same name, same type, declared in one place and used everywhere.

!!! tip "`name` declares, `ref` reuses"
    Use `name=` to *define* a new element and `ref=` to *point at* one that
    already exists. UBL leans heavily on `ref`: the basic components are defined
    once and referenced by every document and aggregate that needs them.

## Attributes

Attributes are declared with `xsd:attribute`, after any content model. Each has
a `name`, a `type`, and an optional `use` that says whether it is required:

``` xml title="attributes.xsd" linenums="1"
<xsd:complexType name="ExampleType">
  <xsd:sequence>
    <xsd:element ref="cbc:ID"/>
  </xsd:sequence>
  <xsd:attribute name="schemeID" type="xsd:string" use="optional"/>  <!-- (1)! -->
  <xsd:attribute name="languageID" type="xsd:string" use="required"/>
</xsd:complexType>
```

1.  `use="optional"` is the default and may be omitted; `use="required"` forces
    the attribute to be present. (A third value, `use="prohibited"`, is used
    when restricting a type — see below.)

## simpleContent: a value plus an attribute

This is the pattern people forget, and it is everywhere in UBL. Sometimes you
want an element whose body is a **simple value** — a number or string — but
which still carries an **attribute**. A plain simple type cannot do this (it has
no attributes); a normal complex type cannot do it either (its body is child
elements, not text). The bridge is `xsd:simpleContent`.

The canonical case is a UBL **amount**: a decimal value with a `currencyID`
attribute, as in `<cbc:LineExtensionAmount currencyID="EUR">10.90</cbc:LineExtensionAmount>`.
You define it by *extending* a simple base type to bolt an attribute onto it:

``` xml title="amount.xsd" linenums="1"
<xsd:complexType name="AmountType">
  <xsd:simpleContent>                                <!-- (1)! -->
    <xsd:extension base="xsd:decimal">               <!-- (2)! -->
      <xsd:attribute name="currencyID"
                     type="xsd:string"
                     use="required"/>                <!-- (3)! -->
    </xsd:extension>
  </xsd:simpleContent>
</xsd:complexType>

<xsd:element name="LineExtensionAmount" type="cbc:AmountType"/>
```

1.  `xsd:simpleContent` says the element's body is a **simple value** (text),
    not child elements — but the type may still declare attributes.
2.  `xsd:extension base="xsd:decimal"` keeps the body typed as a decimal and
    *adds* to it. The element's text content must be a valid `xsd:decimal`.
3.  The added attribute. `currencyID` is required here, so an amount without a
    currency is invalid.

<div class="xslt-result" markdown>
**Valid** — decimal body, required attribute present:

`<cbc:LineExtensionAmount currencyID="EUR">10.90</cbc:LineExtensionAmount>`

**Invalid** — `currencyID` missing (it is `use="required"`):

`<cbc:LineExtensionAmount>10.90</cbc:LineExtensionAmount>`

**Invalid** — body is not a decimal:

`<cbc:LineExtensionAmount currencyID="EUR">free</cbc:LineExtensionAmount>`
</div>

!!! warning "Reach for `simpleContent` whenever a value needs an attribute"
    If an element has *both* text content and an attribute, you need
    `xsd:simpleContent` with `xsd:extension`. A bare simple type cannot hold the
    attribute, and ordinary complex content expects child elements instead of
    text. Every UBL amount, quantity, code, and measure that carries a unit or
    scheme attribute is built this way.

## complexContent: deriving one complex type from another

`xsd:simpleContent` extends a *simple* base. Its counterpart `xsd:complexContent`
derives from another *complex* type — either adding to it (`xsd:extension`) or
narrowing it (`xsd:restriction`).

Extension appends particles to the base type's content model:

``` xml title="complexcontent-extension.xsd" linenums="1"
<xsd:complexType name="AddressType">
  <xsd:sequence>
    <xsd:element ref="cbc:StreetName"/>
    <xsd:element ref="cbc:CityName"/>
  </xsd:sequence>
</xsd:complexType>

<xsd:complexType name="PostalAddressType">
  <xsd:complexContent>
    <xsd:extension base="cac:AddressType">         <!-- (1)! -->
      <xsd:sequence>
        <xsd:element ref="cbc:PostalZone"/>        <!-- (2)! -->
      </xsd:sequence>
    </xsd:extension>
  </xsd:complexContent>
</xsd:complexType>
```

1.  `xsd:extension base="cac:AddressType"` inherits the base type's `StreetName`
    and `CityName` particles.
2.  The new particle is *appended after* the inherited ones, giving the
    sequence `StreetName`, `CityName`, `PostalZone`.

Restriction goes the other way, tightening the base — for example making an
optional element required or removing an attribute. The derived type must remain
a valid subset of the base.

!!! info "UBL prefers composition over deep inheritance"
    Although XSD supports it, UBL almost never builds tall derivation
    hierarchies. Instead it **composes**: a type is a `sequence` of `ref`s to
    other components. `cac:PartyType` references `cac:PartyName`, which
    references `cbc:Name`, and so on. When reading UBL, expect nesting by
    reference, not subclassing.

## Building up an aggregate

Composition is best seen end to end. Here is the supplier party, built from a
`cbc:` leaf upward into the `cac:` aggregates the instance uses.

Start with the basic component — a name, declared once as a global:

``` xml title="party.xsd" linenums="1"
<xsd:element name="Name" type="cbc:NameType"/>           <!-- cbc: a leaf string -->

<xsd:complexType name="PartyNameType">                   <!-- (1)! -->
  <xsd:sequence>
    <xsd:element ref="cbc:Name"/>
  </xsd:sequence>
</xsd:complexType>
<xsd:element name="PartyName" type="cac:PartyNameType"/>

<xsd:complexType name="PartyType">                       <!-- (2)! -->
  <xsd:sequence>
    <xsd:element ref="cac:PartyName" maxOccurs="unbounded"/>
  </xsd:sequence>
</xsd:complexType>
<xsd:element name="Party" type="cac:PartyType"/>

<xsd:complexType name="SupplierPartyType">               <!-- (3)! -->
  <xsd:sequence>
    <xsd:element ref="cac:Party"/>
  </xsd:sequence>
</xsd:complexType>
<xsd:element name="AccountingSupplierParty" type="cac:SupplierPartyType"/>
```

1.  `PartyNameType` wraps the basic `cbc:Name`.
2.  `PartyType` references `cac:PartyName` — an aggregate built from a smaller
    aggregate.
3.  `SupplierPartyType` references `cac:Party`. Each layer adds a level of
    nesting by reference, matching the instance:
    `cac:AccountingSupplierParty > cac:Party > cac:PartyName > cbc:Name`.

The invoice line is the other aggregate in the running example. It combines a
plain `cbc:ID` with the `simpleContent` amount type from earlier:

``` xml title="invoiceline.xsd" linenums="1"
<xsd:complexType name="InvoiceLineType">
  <xsd:sequence>
    <xsd:element ref="cbc:ID"/>                   <!-- (1)! -->
    <xsd:element ref="cbc:LineExtensionAmount"/>  <!-- (2)! -->
  </xsd:sequence>
</xsd:complexType>
<xsd:element name="InvoiceLine" type="cac:InvoiceLineType"/>
```

1.  A simple identifier for the line.
2.  The amount with its `currencyID` attribute — the `simpleContent` type at
    work inside an aggregate.

Pulling it together, the document type is just a sequence of references to these
components, with cardinality set per UBL's rules:

``` xml title="invoice.xsd" linenums="1"
<xsd:complexType name="InvoiceType">
  <xsd:sequence>
    <xsd:element ref="cbc:ID"/>
    <xsd:element ref="cbc:IssueDate"/>
    <xsd:element ref="cbc:DocumentCurrencyCode"/>
    <xsd:element ref="cac:AccountingSupplierParty"/>
    <xsd:element ref="cac:InvoiceLine" maxOccurs="unbounded"/>
  </xsd:sequence>
</xsd:complexType>
<xsd:element name="Invoice" type="InvoiceType"/>
```

<div class="xslt-result" markdown>
The `invoice.xml` at the top of this page is a **valid** instance of this
schema: required `ID`, `IssueDate`, and `DocumentCurrencyCode` in order, a
supplier party nested four levels deep, and one `InvoiceLine` whose
`LineExtensionAmount` carries the required `currencyID`.
</div>

## Next

Every leaf above bottomed out in a `cbc:` value typed as `xsd:decimal`,
`xsd:string`, or a date — and those base types can be constrained far more
tightly. [Simple types and restrictions](simple-types-restrictions.md) shows how
to bound, pattern-match, and enumerate the values that fill these elements.
