# Context and tests

A Schematron rule has two halves that work together. The `context` of a
`<rule>` is an [XPath](../xpath/index.md) pattern that *selects the nodes to
check*; the `test` of each `<assert>` or `<report>` inside that rule is an XPath
**boolean** evaluated with the selected node as the **current node**. Because
the context node is the starting point, tests are written with **relative
paths** — `cbc:LineExtensionAmount`, `cac:InvoiceLine`, `@currencyID` — not
absolute paths from the document root.

## The running example

This section validates a small public UBL `Invoice`. We will return to it on
every page:

``` xml title="invoice.xml" linenums="1"
<Invoice xmlns="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2"
         xmlns:cbc="urn:oasis:names:specification:ubl:schema:xsd:CommonBasicComponents-2"
         xmlns:cac="urn:oasis:names:specification:ubl:schema:xsd:CommonAggregateComponents-2">
  <cbc:CustomizationID>urn:cen.eu:en16931:2017</cbc:CustomizationID>
  <cbc:ID>INV-0001</cbc:ID>
  <cbc:IssueDate>2026-01-15</cbc:IssueDate>
  <cbc:DocumentCurrencyCode>EUR</cbc:DocumentCurrencyCode>
  <cac:InvoiceLine>
    <cbc:LineExtensionAmount currencyID="EUR">100.00</cbc:LineExtensionAmount>
  </cac:InvoiceLine>
  <cac:InvoiceLine>
    <cbc:LineExtensionAmount currencyID="EUR">50.00</cbc:LineExtensionAmount>
  </cac:InvoiceLine>
  <cac:LegalMonetaryTotal>
    <cbc:LineExtensionAmount currencyID="EUR">150.00</cbc:LineExtensionAmount>
  </cac:LegalMonetaryTotal>
</Invoice>
```

## Context selects, test checks

A rule with `context="cac:InvoiceLine"` fires **once per invoice line**. Inside
it, every test runs with that line as the current node, so a bare
`cbc:LineExtensionAmount` means "the amount *of this line*":

``` xml title="line-rule.sch" linenums="1"
<rule context="cac:InvoiceLine">
  <assert test="cbc:LineExtensionAmount">          <!-- (1)! -->
    Each invoice line must carry a line extension amount.
  </assert>
  <assert test="cbc:LineExtensionAmount/@currencyID">   <!-- (2)! -->
    The line amount must state its currency.
  </assert>
</rule>
```

1. Relative path — the `cbc:LineExtensionAmount` child of *this* line.
2. `@currencyID` is the attribute on that child, addressed with the attribute
   axis. Both run with the matched `cac:InvoiceLine` as the current node.

<div class="xslt-result" markdown>
The rule fires twice (two `cac:InvoiceLine` nodes). Both lines have an amount
and a `currencyID`, so all four assertions hold — no message is reported.
</div>

!!! note "`assert` vs `report`"
    An `<assert>` is the normal form: it reports its message when the test is
    **false** (the expected condition failed). A `<report>` is the inverse — it
    reports when the test is **true** (an unwanted condition was found). The
    same XPath skills apply to both.

## XPath 2.0 with `queryBinding="xslt2"`

The default Schematron binding is XPath 1.0. Declaring
`queryBinding="xslt2"` on the root `<schema>` unlocks **XPath 2.0**: sequences,
`sum()`, `count()`, `string-length()`, `matches()`, quantified
`every`/`some $x in ... satisfies ...`, and `if/then/else`. This is what makes
cross-field calculation rules expressible.

A totals-equal-sum check (the idea behind the real BR-CO-10) compares the
document total against the sum of the line amounts — a single XPath 2.0
expression over a sequence:

``` xml title="totals.sch" linenums="1"
<schema xmlns="http://purl.oclc.org/dsdl/schematron"
        queryBinding="xslt2">
  <ns prefix="cbc" uri="urn:oasis:names:specification:ubl:schema:xsd:CommonBasicComponents-2"/>
  <ns prefix="cac" uri="urn:oasis:names:specification:ubl:schema:xsd:CommonAggregateComponents-2"/>
  <pattern>
    <rule context="cac:LegalMonetaryTotal">
      <assert test="xs:decimal(cbc:LineExtensionAmount)
                     = sum(../cac:InvoiceLine/cbc:LineExtensionAmount)">   <!-- (1)! -->
        The sum of line net amounts must equal the document line total.
      </assert>
    </rule>
  </pattern>
</schema>
```

1. `sum(...)` adds every line's `cbc:LineExtensionAmount` as a sequence. `..`
   steps up from `cac:LegalMonetaryTotal` to the `Invoice` so the path reaches
   the sibling `cac:InvoiceLine` elements.

<div class="xslt-result" markdown>
`100.00 + 50.00 = 150.00` and the total is `150.00`, so the test is true and the
assertion passes. Change the total to `140.00` and the message fires.
</div>

## Common test idioms

Most business rules are built from a handful of XPath shapes. The current node
is always the rule's context node.

| Intent | XPath test |
| --- | --- |
| Existence | `cbc:ID` |
| Non-empty value | `normalize-space(cbc:CustomizationID) != ''` |
| Conditional requirement ("if A then B") | `not(A) or B` |
| Cardinality (exactly one) | `count(cac:LegalMonetaryTotal) = 1` |
| Code membership | `cbc:DocumentCurrencyCode = ('EUR','USD','GBP')` |

The non-empty idiom is exactly how the real **BR-01** ("An Invoice shall have a
Specification identifier") is bound — its test is
`normalize-space(cbc:CustomizationID) != ''`.

!!! tip "If A then B = `not(A) or B`"
    XPath has no `if … then` *requirement* operator, so a conditional rule is
    written as the logically equivalent `not(condition) or requirement`. Read it
    as: "either the condition does not apply, or the requirement is met." This
    one pattern is the heart of nearly every business rule.

## Severity with `flag`

Each assertion may carry a `flag` attribute naming its severity. `flag` is a
**conventional label** — Schematron does not assign it meaning; the validation
report simply carries it through so a consumer can sort fatal failures from
advisory ones. The EN16931 rules use `fatal` and `warning`.

A fatal calculation rule — the authentic **BR-CO-15**, the invoice total *with*
VAT equals the total *without* VAT plus the VAT amount:

``` xml title="br-co-15.sch" linenums="1"
<rule context="cac:LegalMonetaryTotal">
  <assert flag="fatal"
          test="xs:decimal(cbc:TaxInclusiveAmount)
                = xs:decimal(cbc:TaxExclusiveAmount) + xs:decimal(../cac:TaxTotal/cbc:TaxAmount)">
    [BR-CO-15]-Invoice total amount with VAT (BT-112) = Invoice total amount
    without VAT (BT-109) + Invoice total VAT amount (BT-110).
  </assert>
</rule>
```

A warning rule — the authentic **BR-51** — uses `<report>` because it flags an
*unwanted* condition, and marks it `warning` rather than `fatal`:

``` xml title="br-51.sch" linenums="1"
<rule context="cac:PaymentMeans/cac:CardAccount">
  <report flag="warning"
          test="string-length(normalize-space(cbc:PrimaryAccountNumberID)) &gt; 10">
    [BR-51]-In accordance with card payments security standards an invoice
    should never include a full card primary account number (BT-87).
  </report>
</rule>
```

<div class="xslt-result" markdown>
A `fatal` failure typically blocks acceptance of the document; a `warning` is
advisory and surfaces as a non-blocking note. Both appear in the same report,
distinguished only by their flag.
</div>

## The conditional-requirement pattern, worked

The `not(A) or B` shape is worth seeing in full, because it is how almost every
"only required *when* …" rule is expressed. Suppose a payment made by card must
carry an account identifier:

``` xml title="conditional.sch" linenums="1"
<rule context="cac:PaymentMeans">
  <assert flag="fatal"
          test="not(cbc:PaymentMeansCode = '48')
                or normalize-space(cac:CardAccount/cbc:PrimaryAccountNumberID) != ''">  <!-- (1)! -->
    If payment is by card, a card account identifier is required.
  </assert>
</rule>
```

1. `A` is "this is a card payment" (`cbc:PaymentMeansCode = '48'`); `B` is "an
   account identifier is present". When `A` is false the line passes
   automatically; when `A` is true, `B` must hold.

<div class="xslt-result" markdown>
- Non-card payment → `not(A)` is true → assertion passes regardless of `B`.
- Card payment **with** an account id → `A` true, `B` true → passes.
- Card payment **without** an account id → `A` true, `B` false → fails and the
  message is reported.
</div>

## Next

[Variables and messages](variables-and-messages.md) — name the values your
tests reuse with `<let>`, and turn a passing or failing test into a precise,
data-rich report message.
