# Anatomy of a UBL invoice

**UBL** — the OASIS *Universal Business Language* — is one of the two XML syntaxes
that can carry an [EN16931](index.md) invoice (the other is UN/CEFACT CII). A UBL
invoice is a single XML document rooted at `<Invoice>`, and once you know how it is
organised it reads like a paper invoice: header fields, the two parties, the tax
and total blocks, then the lines.

This page walks a small but complete invoice and shows how the **business terms**
of the EN16931 model map onto the elements.

## Three namespaces

Every UBL invoice mixes three namespaces, and you will see their prefixes on
nearly every element:

| Prefix | Namespace | Holds |
| --- | --- | --- |
| *(default)* | `…:ubl:schema:xsd:Invoice-2` | the document root `<Invoice>` only |
| `cbc` | `…:CommonBasicComponents-2` | **leaf** fields — a single value (`cbc:ID`, `cbc:IssueDate`) |
| `cac` | `…:CommonAggregateComponents-2` | **aggregates** — groups of other elements (`cac:AccountingSupplierParty`) |

The split is the whole mental model: **`cbc` is a value, `cac` is a container.**
A `cac` element holds more `cac` and `cbc` children; a `cbc` element holds text.

## A complete invoice

``` xml title="invoice.xml" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<Invoice xmlns="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2"
         xmlns:cac="urn:oasis:names:specification:ubl:schema:xsd:CommonAggregateComponents-2"
         xmlns:cbc="urn:oasis:names:specification:ubl:schema:xsd:CommonBasicComponents-2">

  <cbc:CustomizationID>urn:cen.eu:en16931:2017</cbc:CustomizationID>   <!-- (1)! -->
  <cbc:ID>INV-001</cbc:ID>                                            <!-- (2)! -->
  <cbc:IssueDate>2026-06-20</cbc:IssueDate>
  <cbc:DueDate>2026-07-20</cbc:DueDate>
  <cbc:InvoiceTypeCode>380</cbc:InvoiceTypeCode>                      <!-- (3)! -->
  <cbc:DocumentCurrencyCode>EUR</cbc:DocumentCurrencyCode>

  <cac:AccountingSupplierParty>                                       <!-- (4)! -->
    <cac:Party>
      <cac:PartyName><cbc:Name>Northwind Traders Oy</cbc:Name></cac:PartyName>
      <cac:PostalAddress>
        <cbc:CityName>Tampere</cbc:CityName>
        <cac:Country><cbc:IdentificationCode>FI</cbc:IdentificationCode></cac:Country>
      </cac:PostalAddress>
    </cac:Party>
  </cac:AccountingSupplierParty>

  <cac:AccountingCustomerParty>
    <cac:Party>
      <cac:PartyName><cbc:Name>Contoso Wholesale AB</cbc:Name></cac:PartyName>
    </cac:Party>
  </cac:AccountingCustomerParty>

  <cac:TaxTotal>                                                      <!-- (5)! -->
    <cbc:TaxAmount currencyID="EUR">5.45</cbc:TaxAmount>
  </cac:TaxTotal>

  <cac:LegalMonetaryTotal>                                            <!-- (6)! -->
    <cbc:LineExtensionAmount currencyID="EUR">21.80</cbc:LineExtensionAmount>
    <cbc:TaxExclusiveAmount currencyID="EUR">21.80</cbc:TaxExclusiveAmount>
    <cbc:TaxInclusiveAmount currencyID="EUR">27.25</cbc:TaxInclusiveAmount>
    <cbc:PayableAmount currencyID="EUR">27.25</cbc:PayableAmount>
  </cac:LegalMonetaryTotal>

  <cac:InvoiceLine>                                                   <!-- (7)! -->
    <cbc:ID>1</cbc:ID>
    <cbc:InvoicedQuantity unitCode="C62">2</cbc:InvoicedQuantity>
    <cbc:LineExtensionAmount currencyID="EUR">21.80</cbc:LineExtensionAmount>
    <cac:Item><cbc:Name>Empire Burlesque (CD)</cbc:Name></cac:Item>
    <cac:Price><cbc:PriceAmount currencyID="EUR">10.90</cbc:PriceAmount></cac:Price>
  </cac:InvoiceLine>

</Invoice>
```

1.  **BT-24, Specification identifier.** The single most important field: it
    declares *which rulebook* the document claims to follow. `urn:cen.eu:en16931:2017`
    is plain EN16931; a Peppol invoice puts a longer Peppol URN here (see
    [Peppol and CIUS profiles](peppol-cius.md)). This is the field rule **BR-01**
    checks.
2.  **BT-1, Invoice number.**
3.  **BT-3, Invoice type code** — `380` is "commercial invoice", a value from
    code list UNCL1001 (see [Genericode](genericode-codelists.md)).
4.  **BG-4, Seller.** A `cac` aggregate wrapping a `cac:Party`, itself wrapping
    name and address aggregates — `cac` nesting all the way down to the `cbc`
    leaves.
5.  **BG-22 / BT-110, total VAT.** The `currencyID` attribute restates the
    currency on every amount.
6.  **BG-22, document totals.** The four amounts a rule like **BR-CO-10** ties
    together arithmetically.
7.  **BG-25, an invoice line** — quantity, line amount, the item, and a unit
    price.

## Business terms vs elements

EN16931 does not describe XML at all. It defines a **semantic model**: numbered
*business terms* (BT) and *business groups* (BG) — BT-1 is "Invoice number", BG-4
is "Seller" — with no mention of `cbc` or `cac`. The UBL *binding* is the mapping
from those numbers to concrete elements:

| EN16931 | Means | UBL element |
| --- | --- | --- |
| BT-24 | Specification identifier | `cbc:CustomizationID` |
| BT-1 | Invoice number | `cbc:ID` |
| BT-2 | Issue date | `cbc:IssueDate` |
| BT-5 | Currency | `cbc:DocumentCurrencyCode` |
| BG-4 | Seller | `cac:AccountingSupplierParty` |
| BG-25 | Invoice line | `cac:InvoiceLine` |
| BT-131 | Line net amount | `cac:InvoiceLine/cbc:LineExtensionAmount` |
| BT-106 | Sum of line net amounts | `cac:LegalMonetaryTotal/cbc:LineExtensionAmount` |

!!! note "The same model, a different syntax"
    In the CII syntax, BT-1 is `ram:ID` inside `rsm:ExchangedDocument`, not
    `cbc:ID`. The *business term is identical*; only the binding differs. This is
    exactly why EN16931's Schematron is written as an
    [abstract pattern bound twice](../schematron/abstract-patterns-en16931.md) —
    one rulebook, two syntaxes.

## Reading the invoice with XPath

Because it is just XML, every [XPath](../xpath/index.md) technique applies. The
totals rule BR-CO-10 — *document line total equals the sum of the lines* — is one
expression:

``` xml
sum(cac:InvoiceLine/cbc:LineExtensionAmount)
  = cac:LegalMonetaryTotal/cbc:LineExtensionAmount
```

And pulling a human-readable summary is ordinary [XSLT](../xslt/index.md):

``` xml
<xsl:value-of select="cbc:ID"/> —
<xsl:value-of select="cac:AccountingCustomerParty/cac:Party/cac:PartyName/cbc:Name"/>:
<xsl:value-of select="cac:LegalMonetaryTotal/cbc:PayableAmount"/>
<xsl:value-of select="cac:LegalMonetaryTotal/cbc:PayableAmount/@currencyID"/>
```

<div class="xslt-result" markdown>
INV-001 — Contoso Wholesale AB: 27.25 EUR
</div>

## Next

A well-formed invoice is only the start — it still has to *pass*. Next:
[The validation pipeline](validation-pipeline.md) — the layered checks every
invoice runs, and which layer catches what.
