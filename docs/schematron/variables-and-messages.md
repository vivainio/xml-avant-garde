# Variables and messages

A bare `<assert test="…"/>` can already validate, but two things separate a
proof-of-concept Schematron from a maintainable one: not repeating the same
sub-expression in five rules, and emitting a message that tells the author
*what* failed and *which value* offended. This page covers `<let>` variables,
dynamic messages, reusable `<diagnostics>`, and the `id`/`see`/`role` attributes
that let the public **EN16931** rules carry stable identifiers like `BR-CO-10`.

The running document is the public UBL `Invoice` shape — an `cbc:ID`, a
`cbc:DocumentCurrencyCode`, one or more `cac:InvoiceLine` carrying a
`cbc:LineExtensionAmount`, and a `cac:LegalMonetaryTotal` that is supposed to
restate the sum of those lines:

``` xml title="invoice.xml" linenums="1"
<Invoice xmlns="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2"
         xmlns:cbc="urn:oasis:names:specification:ubl:schema:xsd:CommonBasicComponents-2"
         xmlns:cac="urn:oasis:names:specification:ubl:schema:xsd:CommonAggregateComponents-2">
  <cbc:ID>INV-001</cbc:ID>
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

## `<let>` — naming an expression

A `<let>` binds a name to the result of an XPath expression. Everywhere in
scope you then write `$name` instead of repeating the path — in a `test`, and
just as importantly inside a message.

``` xml title="variables.sch" linenums="1"
<let name="lineTotal" value="sum(cac:InvoiceLine/cbc:LineExtensionAmount)"/>
```

The value is *any* XPath expression, not just a constant: `$lineTotal` above is
the running sum of every line amount. Bind it once and the rule reads like the
business rule it encodes:

``` xml title="br-co-10.sch" linenums="1"
<pattern>
  <rule context="Invoice">
    <let name="lineTotal" value="sum(cac:InvoiceLine/cbc:LineExtensionAmount)"/>
    <assert test="cac:LegalMonetaryTotal/cbc:LineExtensionAmount = $lineTotal">
      [BR-CO-10]-Sum of Invoice line net amount (BT-106) = Σ Invoice line net amount (BT-131).
    </assert>
  </rule>
</pattern>
```

!!! tip "Factor the expression, not just the value"
    `$lineTotal` is recomputed against each `Invoice` the rule matches — it is a
    *named expression*, evaluated in the rule's context, not a snapshot taken
    once. The win is that `sum(cac:InvoiceLine/cbc:LineExtensionAmount)` now
    lives in one place; if the path changes, you fix it once.

### Scope

Where you place the `<let>` decides who can see `$x`:

| Placed inside | Scope | Visible to |
| --- | --- | --- |
| `<schema>` | global | every pattern and rule |
| `<pattern>` | pattern-scoped | every rule in that pattern |
| `<rule>` | rule-scoped | only that rule's asserts/reports |

``` xml title="scopes.sch" linenums="1"
<schema xmlns="http://purl.oclc.org/dsdl/schematron"
        queryBinding="xslt2">

  <let name="currency" value="//cbc:DocumentCurrencyCode"/>  <!-- (1)! -->

  <pattern>
    <let name="lines" value="//cac:InvoiceLine"/>            <!-- (2)! -->
    <rule context="Invoice">
      <let name="lineTotal"
           value="sum($lines/cbc:LineExtensionAmount)"/>     <!-- (3)! -->
      <assert test="cac:LegalMonetaryTotal/cbc:LineExtensionAmount = $lineTotal">
        [BR-CO-10]-Sum of Invoice line net amount (BT-106) = Σ Invoice line net amount (BT-131).
      </assert>
    </rule>
  </pattern>
</schema>
```

1.  **Global** — `$currency` works in any rule of any pattern.
2.  **Pattern-scoped** — `$lines` is visible to every rule in this pattern only.
3.  **Rule-scoped** — `$lineTotal` exists only inside this `rule`, and may
    reference variables from the enclosing scopes (`$lines`).

!!! note "Declare globals with a namespace-aware query binding"
    Once paths use prefixes like `cbc:` you need `queryBinding="xslt2"` (or
    `xslt`) plus `<ns prefix="cbc" uri="…"/>` declarations on the schema. The
    EN16931 schematron does exactly this. Scope rules are the same regardless of
    binding.

## Dynamic messages

A static message says *that* something is wrong; a dynamic one says *what*. Two
elements turn the offending node into text inside the message:

- `<value-of select="XPath"/>` — inserts the string value the expression
  evaluates to, relative to the rule context.
- `<name/>` — inserts the name of the context node (or of a selected node).

``` xml title="dynamic-message.sch" linenums="1"
<rule context="Invoice">
  <assert test="cbc:DocumentCurrencyCode = ('EUR','USD','GBP')">
    Currency '<value-of select="cbc:DocumentCurrencyCode"/>' on element
    &lt;<name/>&gt; is not allowed.
  </assert>
</rule>
```

If the invoice above carried `<cbc:DocumentCurrencyCode>SEK</cbc:DocumentCurrencyCode>`,
the assertion fails and the processor emits the substituted text:

<div class="xslt-result" markdown>
Currency 'SEK' on element &lt;Invoice&gt; is not allowed.
</div>

The same applies to `<report>` (which fires when its `test` is **true** — the
mirror image of `assert`). Reaching back into the data makes a failure
actionable: the author sees the bad value, not just a rule number.

!!! tip "`value-of` is evaluated in the rule context"
    The `select` is resolved against the same context node as the `test`, so you
    can surface intermediate values too — e.g. report the computed
    `$lineTotal` next to the declared total when `BR-CO-10` fails:
    `... declared <value-of select="cac:LegalMonetaryTotal/cbc:LineExtensionAmount"/> but lines sum to <value-of select="$lineTotal"/>.`

## `<diagnostics>` — reusable explanatory text

The message on an `assert` should stay short. When a rule needs a paragraph of
*why* and *how to fix it*, put that text in a `<diagnostic>` and point at it
from the assert with `diagnostics="…"`. The block lives once at schema level and
can be referenced by several asserts.

``` xml title="diagnostics.sch" linenums="1"
<pattern>
  <rule context="Invoice">
    <assert id="BR-CO-10" diagnostics="d-br-co-10"
            test="cac:LegalMonetaryTotal/cbc:LineExtensionAmount = sum(cac:InvoiceLine/cbc:LineExtensionAmount)">
      [BR-CO-10]-Sum of Invoice line net amount (BT-106) = Σ Invoice line net amount (BT-131).
    </assert>
  </rule>
</pattern>

<diagnostics>
  <diagnostic id="d-br-co-10">
    BT-106 (the document-level sum of line net amounts in
    cac:LegalMonetaryTotal/cbc:LineExtensionAmount) must equal the total of all
    BT-131 line net amounts. Here the lines sum to
    <value-of select="sum(cac:InvoiceLine/cbc:LineExtensionAmount)"/> but the
    declared total is
    <value-of select="cac:LegalMonetaryTotal/cbc:LineExtensionAmount"/>.
  </diagnostic>
</diagnostics>
```

A processor reports the assert message and, attached to it, the diagnostic:

<div class="xslt-result" markdown>
\[BR-CO-10]-Sum of Invoice line net amount (BT-106) = Σ Invoice line net amount (BT-131).

> BT-106 … must equal the total of all BT-131 line net amounts. Here the lines
> sum to 150.00 but the declared total is 140.00.
</div>

!!! info "`diagnostics` takes a list"
    The attribute is space-separated, so one assert can pull in several
    diagnostic blocks: `diagnostics="d-br-co-10 d-rounding"`. Diagnostics may use
    `<value-of>` and `<name/>` exactly like inline messages.

## Identifying and documenting rules

EN16931 rules are referenced by stable codes — `BR-01`, `BR-CO-10` — in
specifications, test suites, and bug reports. Schematron carries that identity
in the `id` attribute, and the convention in the public files is to repeat the
code as a bracketed prefix in the human message so it survives in plain-text
logs too.

``` xml title="ids-and-links.sch" linenums="1"
<rule context="Invoice"
      id="UBL-Invoice-rules"
      see="https://www.en16931.eu/"
      role="error">
  <assert id="BR-01"
          test="cbc:CustomizationID"
          see="https://www.en16931.eu/"
          role="fatal">
    [BR-01]-An Invoice shall have a Specification identifier (BT-24).
  </assert>
</rule>
```

- **`id`** — the rule's stable handle. Put it on the `assert` (`BR-01`) and,
  where useful, on the `rule`. Tooling groups results by it.
- **`see`** — a URI linking to the governing specification. Reporting tools turn
  it into a "more info" link.
- **`role`** — a free-text label, used by EN16931 to tag severity such as
  `error`, `fatal`, or `warning`, so a downstream tool can decide what blocks a
  document versus what merely warns.

!!! note "The id and the `[BR-01]` prefix are independent"
    The `id="BR-01"` is the machine identifier; the `[BR-01]-…` text is for
    humans reading a log. The public schematron keeps both in sync by hand —
    matching them is convention, not enforced, so keep them aligned when you add
    a rule.

## Documenting the schema itself

For prose aimed at readers rather than at a processor, Schematron has `<title>`
and `<p>`. They carry no assertion logic — they describe the schema, a pattern,
or a phase, and surface in generated documentation.

``` xml title="documentation.sch" linenums="1"
<schema xmlns="http://purl.oclc.org/dsdl/schematron" queryBinding="xslt2">
  <title>EN16931 model bound to UBL Invoice</title>

  <pattern id="totals">
    <title>Monetary total consistency (BR-CO group)</title>
    <p>Checks that document-level totals restate the sums of the line level,
       per EN16931 §6. Failures here usually mean a rounding or aggregation
       error upstream.</p>
    <rule context="Invoice">
      <!-- asserts … -->
    </rule>
  </pattern>
</schema>
```

!!! tip "Keep the rule's *why* in `<p>`, the *what* in the assert"
    A pattern `<title>`/`<p>` explains the group's intent once; each `assert`
    message states the single condition that failed. That split keeps messages
    short while the context stays discoverable.

## Next

[Abstract patterns and EN16931](abstract-patterns-en16931.md) — factor whole
rule shapes (not just sub-expressions) so the same check can be bound to UBL and
CII without duplication.
