# Genericode code lists

A coded field in an invoice — `cbc:DocumentCurrencyCode`, `cbc:InvoiceTypeCode`,
the VAT category code — is only valid if its value appears in the right *published
list*. Those lists are not hard-coded into the rules; they ship as data, in the
OASIS **Genericode** format (files with a `.gc` extension). This page shows what a
`.gc` file looks like and how to look codes up in it efficiently — the real-world
home of the [`xsl:key`](../xslt/keys.md) and [`map`](../xslt/json.md) techniques
from the XSLT section.

## What a `.gc` file is

Genericode (OASIS *Code List Representation*) expresses one code list as a typed,
self-describing table: a **column set** that declares the columns and names the key
column, then a **row per code**.

``` xml title="Currency-2.1.gc" linenums="1"
<gc:CodeList xmlns:gc="http://docs.oasis-open.org/codelist/ns/genericode/1.0/">

  <Identification>                                  <!-- (1)! -->
    <ShortName>CurrencyCode</ShortName>
    <Version>2.1</Version>
    <CanonicalUri>urn:xoev-de:kosit:codeliste:currency-codes</CanonicalUri>
  </Identification>

  <ColumnSet>                                       <!-- (2)! -->
    <Column Id="code" Use="required">
      <ShortName>Code</ShortName>
      <Data Type="normalizedString"/>
    </Column>
    <Column Id="name" Use="optional">
      <ShortName>Name</ShortName>
      <Data Type="string"/>
    </Column>
    <Key Id="codeKey"><ColumnRef Ref="code"/></Key>  <!-- (3)! -->
  </ColumnSet>

  <SimpleCodeList>                                   <!-- (4)! -->
    <Row>
      <Value ColumnRef="code"><SimpleValue>EUR</SimpleValue></Value>
      <Value ColumnRef="name"><SimpleValue>Euro</SimpleValue></Value>
    </Row>
    <Row>
      <Value ColumnRef="code"><SimpleValue>USD</SimpleValue></Value>
      <Value ColumnRef="name"><SimpleValue>US Dollar</SimpleValue></Value>
    </Row>
  </SimpleCodeList>

</gc:CodeList>
```

1.  **Identification** — the list's name, version and a canonical URI. A
    Schematron binding refers to the list by this identity.
2.  **ColumnSet** — declares the columns up front, with a datatype each. Here a
    `code` and a `name`; real lists often add `Status`, `Description`, dates, etc.
3.  **Key** — which column is the lookup key. Every row's `code` is unique.
4.  **SimpleCodeList** — the rows. Each `Row` carries one `Value` per column,
    referencing the column by `ColumnRef`.

## The lists you actually meet

EN16931 validation ships a folder of these. The common ones:

| List | Field it controls | Source |
| --- | --- | --- |
| Currency codes | `cbc:DocumentCurrencyCode` (BT-5) | ISO 4217 |
| Country codes | `cac:Country/cbc:IdentificationCode` | ISO 3166-1 |
| Invoice type codes | `cbc:InvoiceTypeCode` (BT-3) | UNCL1001 |
| VAT category codes | tax category `cbc:ID` | UNCL5305 |
| Payment means | `cbc:PaymentMeansCode` | UNCL4461 |
| Unit of measure | `@unitCode` on quantities | UN/ECE Rec 20 |

## Are code lists shared across formats?

Yes — and at several levels, because a code list attaches to the **business term**,
not to the element that carries it. It lives *above* the syntax layer.

**Across the two EN16931 syntaxes.** BT-5 (currency) must be an ISO 4217 code
whether the document writes it as UBL's `cbc:DocumentCurrencyCode` or CII's
`ram:InvoiceCurrencyCode`. The allowed-value set is identical; only the element
differs. So the *same* `.gc` files back both bindings — there are two separate
mappings, and only the first is syntax-specific:

| Mapping | Goes from | To | Per-syntax? |
| --- | --- | --- | --- |
| Syntax binding | business term | an *element* | yes (`cbc:…` vs `ram:…`) |
| Code-list binding | business term | a *list* | no — shared |

**Across document types.** Within UBL, an Invoice, a CreditNote and an Order all
draw currency from the same ISO 4217 list. The list does not know which document
references it.

**Across unrelated standards.** The base lists *are* external global standards —
ISO 4217, ISO 3166, the UNCL lists, UN/ECE Rec 20 — reused far beyond invoicing.
EN16931 does not own them; it *references* them. Genericode is a format-neutral
table keyed by code, so any consumer can load the same file, and the
[lookup below](#looking-a-code-up) is identical regardless of the document being
validated.

!!! warning "A profile narrows a shared list, it does not redefine it"
    A [CIUS such as Peppol](peppol-cius.md) may restrict a shared list to a
    *subset* (consistent with "narrow, never loosen"), and add its own scheme lists
    (EAS, ICD) on top. The base list stays shared; the profile just applies a
    tighter allowed set. So: **shared, format-neutral value sets — only the "this
    field uses that list" binding is restated per syntax.**

## Looking a code up

The job is the [codelist join](../xslt/external-documents.md) from the XSLT
section, on real data: the invoice says `EUR`, and validation must confirm `EUR`
is a row in `Currency-2.1.gc`. Because a `.gc` file declares its key column, it
maps directly onto an [`xsl:key`](../xslt/keys.md):

``` xml title="lookup with xsl:key" linenums="1"
<xsl:key name="currency" match="Row" use="Value[@ColumnRef='code']/SimpleValue"/>  <!-- (1)! -->

<xsl:variable name="codes" select="document('Currency-2.1.gc')"/>

<xsl:template match="/">
  <xsl:variable name="ccy" select="//cbc:DocumentCurrencyCode"/>
  <xsl:for-each select="$codes">                       <!-- (2)! -->
    <xsl:if test="not(key('currency', $ccy))">          <!-- (3)! -->
      <error>Currency '<xsl:value-of select="$ccy"/>' is not in ISO 4217.</error>
    </xsl:if>
  </xsl:for-each>
</xsl:template>
```

1.  Index every `Row` by its `code` cell — the same column the `.gc` `<Key>`
    names.
2.  Switch the context into the loaded `.gc` document so `key()` searches *its*
    index — the gotcha explained in
    [Keys and indexed lookup](../xslt/keys.md#why-the-context-switch-is-needed).
3.  `key()` returns the matching row, or empty if the code is unknown. A
    predicate scan would re-read every currency row on every lookup; the key reads
    an index.

In XSLT 3.0 the same list reads naturally as a [`map`](../xslt/json.md) — load
once, index by code, then a constant-time membership test:

``` xml title="lookup with a map (3.0)" linenums="1"
<xsl:variable name="currency" as="map(xs:string, xs:string)"
  select="map:merge(
            document('Currency-2.1.gc')//Row
              ! map:entry( string(Value[@ColumnRef='code']/SimpleValue),
                           string(Value[@ColumnRef='name']/SimpleValue) ))"/>

<xsl:if test="not(map:contains($currency, $ccy))">
  <error>Currency '{$ccy}' is not in ISO 4217.</error>
</xsl:if>
```

!!! tip "Why the index matters here specifically"
    UN/ECE Rec 20 (units) has *thousands* of rows; a multi-line invoice checks a
    `@unitCode` on every line. A predicate scan is rows × lines work per
    document; an `xsl:key` or `map` makes it rows + lines. This is the textbook
    case from [Keys and indexed lookup](../xslt/keys.md), with real numbers behind
    it.

## How validation actually uses them

In the EN16931 artefacts you rarely write the lookup yourself — the code-list
checks are **compiled into the Schematron**. A binding file associates each coded
field with a list, and the toolchain generates assertions equivalent to "the value
of this field exists in list X." The mechanism underneath is exactly the keyed
lookup above; the standard just authors it declaratively so the rule, the field,
and the list stay in separate, maintainable files — mirroring the
[`include` split](../schematron/abstract-patterns-en16931.md#splitting-big-schemas-include)
of the rule model itself.

## Next

The last piece is how EN16931 gets specialised for a real network — added fields,
narrowed lists, and the one thing a profile is forbidden to do:
[Peppol and CIUS profiles](peppol-cius.md).
