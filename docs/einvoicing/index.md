---
icon: lucide/receipt-text
---

# Real-world e-invoicing

The earlier sections teach the XML technologies one at a time: [XSD](../xsd/index.md)
for structure, [Schematron](../schematron/index.md) for business rules,
[XPath](../xpath/index.md) and [XSLT](../xslt/index.md) underneath both. This
section is where they meet a real problem — **European electronic invoicing** —
and stop being abstract.

Almost every example in this site has quietly used the same domain: a UBL invoice,
the EN16931 rule BR-01, a currency code. That is not a coincidence. E-invoicing is
the canonical modern XML application: a public semantic standard, two XML syntaxes,
a layered validation stack, and shared code lists — exactly the four technologies
this site covers, assembled into one pipeline.

## The cast

| Thing | What it is | Built on |
| --- | --- | --- |
| **EN16931** | The European *semantic* standard — the data model and business rules for a core invoice | the concepts; syntax-neutral |
| **UBL** | One XML *syntax* that carries an EN16931 invoice (OASIS) | [XSD](../xsd/index.md) |
| **CII** | The other XML syntax (UN/CEFACT Cross Industry Invoice) | [XSD](../xsd/index.md) |
| **Schematron rules** | The EN16931 business rules (BR-*) as runnable assertions | [Schematron](../schematron/index.md) → [XSLT](../xslt/index.md) |
| **Genericode** | The code lists (currency, country, VAT category…) as data | XML |
| **Peppol BIS** | A *profile* that narrows EN16931 for cross-border exchange | a CIUS of EN16931 |

## How the layers stack

A document is not "valid" or "invalid" against a single thing — it passes through
a **stack**, and each layer catches what the layer below cannot.

```mermaid
flowchart TD
  I["UBL / CII instance"] --> X["XSD<br/>structure & datatypes"]
  X --> E["EN16931 Schematron<br/>BR-* business rules"]
  E --> P["Peppol Schematron<br/>profile rules"]
  P --> C["Code-list checks<br/>(Genericode)"]
  C --> OK["Accepted"]
```

## The pages

1. [Anatomy of a UBL invoice](ubl-invoice.md) — walk a real invoice: namespaces,
   `cbc`/`cac`, and how EN16931's BT-/BG- business terms map onto the XML.
2. [The validation pipeline](validation-pipeline.md) — XSD, then EN16931
   Schematron, then profile and code-list checks: what each layer is for.
3. [Genericode code lists](genericode-codelists.md) — the `.gc` format, and
   looking codes up efficiently with [`xsl:key`](../xslt/keys.md) or a
   [`map`](../xslt/json.md).
4. [Peppol and CIUS profiles](peppol-cius.md) — how EN16931 is specialised for
   real networks, and why a profile may only *narrow*, never loosen.

!!! note "Neutral, invented data"
    Every invoice on these pages uses made-up parties and amounts. The
    *structure*, namespaces, rule identifiers and code lists are the real ones; the
    business data is not.
