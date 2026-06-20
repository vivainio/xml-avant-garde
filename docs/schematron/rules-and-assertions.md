# Rules and assertions

Schematron is **assertion-based**, not grammar-based. Instead of describing the
shape a document must have (the job of XSD), you write a list of *claims that
should hold* and let the engine report any that don't. The vocabulary is tiny:
patterns group rules, rules pick the nodes to check, and asserts/reports state
the conditions.

## The running example

Every example on this page checks the same neutral UBL invoice instance:

``` xml title="invoice.xml" linenums="1"
<Invoice xmlns="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2"
         xmlns:cbc="urn:oasis:names:specification:ubl:schema:xsd:CommonBasicComponents-2"
         xmlns:cac="urn:oasis:names:specification:ubl:schema:xsd:CommonAggregateComponents-2">
  <cbc:ID>INV-001</cbc:ID>
  <cbc:IssueDate>2026-06-20</cbc:IssueDate>
  <cbc:DocumentCurrencyCode>EUR</cbc:DocumentCurrencyCode>
  <cac:InvoiceLine>
    <cbc:ID>1</cbc:ID>
    <cbc:LineExtensionAmount currencyID="EUR">100.00</cbc:LineExtensionAmount>
  </cac:InvoiceLine>
  <cac:InvoiceLine>
    <cbc:ID>2</cbc:ID>
    <cbc:LineExtensionAmount currencyID="EUR">50.00</cbc:LineExtensionAmount>
  </cac:InvoiceLine>
</Invoice>
```

## The building blocks

A Schematron schema declares its namespace prefixes once, then groups rules into
patterns. Because UBL elements live in namespaces, every context and test below
is written with the `cbc:`/`cac:` prefixes bound in the `<ns>` declarations:

``` xml title="rules.sch" linenums="1"
<schema xmlns="http://purl.oclc.org/dsdl/schematron" queryBinding="xslt2">
  <ns prefix="cbc" uri="urn:oasis:names:specification:ubl:schema:xsd:CommonBasicComponents-2"/>  <!-- (1)! -->
  <ns prefix="cac" uri="urn:oasis:names:specification:ubl:schema:xsd:CommonAggregateComponents-2"/>

  <pattern>                                                   <!-- (2)! -->
    <rule context="cac:InvoiceLine">                          <!-- (3)! -->
      <assert test="cbc:LineExtensionAmount">                 <!-- (4)! -->
        An invoice line must carry a line amount.
      </assert>
    </rule>
  </pattern>
</schema>
```

1.  `<ns>` binds a prefix to a namespace URI for the whole schema. The
    `queryBinding="xslt2"` attribute selects XPath 2.0 as the expression
    language.
2.  `<pattern>` groups related rules. A schema can hold many patterns; they are
    just organisational containers.
3.  `<rule context="…">` selects the nodes this rule applies to. The `context`
    is an XPath that works exactly like an XSLT `match` pattern — here, every
    `cac:InvoiceLine` element in the document.
4.  `<assert test="…">` states a condition that *should* be true for the context
    node. Its text is the message reported when the condition is **not** met.

### `<assert>` fires when the test is FALSE

This is the single most important — and most confusing — fact about Schematron.

!!! warning "An assert fires when its test is FALSE"
    You write the condition that **should** be true. The assertion FIRES — i.e.
    reports a violation — precisely when that condition evaluates to **false**.
    Read `<assert test="cbc:LineExtensionAmount">…</assert>` as *"I assert this
    line has an amount; complain if it doesn't."* Beginners often write the
    test as if it described the *error* — that inverts the logic and reports
    every valid document.

### `<report>` is the inverse

`<report test="…">` is the mirror image: it fires when its test is **true**. Use
it to flag something that is present but shouldn't be.

``` xml title="rules.sch" linenums="1"
<rule context="cac:InvoiceLine">
  <report test="cbc:LineExtensionAmount &lt; 0">             <!-- (1)! -->
    A line amount must not be negative.
  </report>
</rule>
```

1.  Fires *when the test holds* — a negative amount is exactly what we want to
    catch. The `&lt;` is the escaped `<`, since the schema is itself XML.

So the two are duals: `assert test="X"` is equivalent to `report test="not(X)"`,
and vice versa. Pick whichever reads more naturally — assert for "this must
hold", report for "this must not happen".

## First matching rule wins

Within a single pattern, each node is handled by the **first** `rule` whose
`context` matches it. Once a node is claimed by a rule, later rules in the same
pattern are *not* also applied to it — exactly like XSLT template matching,
where one node activates one template. (See
[XSLT templates](../xslt/templates.md) for the same idea on the transform side.)

``` xml title="rules.sch" linenums="1"
<pattern>
  <rule context="cac:InvoiceLine[cbc:ID = '1']">             <!-- (1)! -->
    <assert test="cbc:LineExtensionAmount">First line needs an amount.</assert>
  </rule>
  <rule context="cac:InvoiceLine">                          <!-- (2)! -->
    <assert test="cbc:ID">Every line needs an ID.</assert>
  </rule>
</pattern>
```

1.  Line 1 matches *this* rule, so only its assert is evaluated for that node.
2.  Line 2 falls through to here. The first line never reaches this rule, even
    though its context (`cac:InvoiceLine`) would also match it. To check
    several independent conditions on the *same* node, put the asserts in one
    rule — or in separate patterns.

## A couple of direct rules

Direct (non-abstract) rules test the running invoice straight away. Here two
patterns require a line amount on every invoice line and an issue date on the
invoice:

``` xml title="rules.sch" linenums="1"
<pattern>
  <rule context="cac:InvoiceLine">
    <assert test="cbc:LineExtensionAmount">
      Invoice line <value-of select="cbc:ID"/> is missing a line amount.
    </assert>
  </rule>
</pattern>

<pattern>
  <rule context="/Invoice">                                 <!-- (1)! -->
    <assert test="cbc:IssueDate">
      An invoice must have an issue date.
    </assert>
  </rule>
</pattern>
```

1.  The context can be any XPath, including an absolute path to the document
    element.

Against the instance above, both lines carry a `cbc:LineExtensionAmount` and the
invoice carries a `cbc:IssueDate`, so **nothing fires** — the document passes:

<div class="xslt-result" markdown>
No assertions fired — document is valid.
</div>

Now remove the amount from the second line (delete its
`<cbc:LineExtensionAmount>`). The first rule's `test` is false for that node, so
the assert fires and emits its message — with `cbc:ID` filled in from the
context node:

<div class="xslt-result" markdown>
Invoice line 2 is missing a line amount.
</div>

## A real EN16931 rule

The same shapes appear in the public CEN EN16931 UBL Schematron (EUPL/Apache).
A direct rule on `cac:InvoiceLine` asserts the presence of the line net amount,
with a `flag` marking severity and a stable, bracketed business-rule message:

``` xml title="EN16931-UBL.sch" linenums="1"
<rule context="cac:InvoiceLine">
  <assert test="cbc:LineExtensionAmount" flag="fatal">      <!-- (1)! -->
    [BR-CO-10]-Sum of Invoice line net amount (BT-106) = &#931; Invoice line net amount (BT-131).
  </assert>
</rule>
```

1.  `flag="fatal"` is an arbitrary label the calling tool can act on (e.g. to
    distinguish blocking errors from warnings); Schematron does not interpret it
    itself. The message follows the EN16931 convention of a `[BR-…]` code
    followed by the human-readable rule text.

Other rules in the same ruleset read identically — for example the assert
behind the specification identifier:

<div class="xslt-result" markdown>
\[BR-01]-An Invoice shall have a Specification identifier (BT-24).
</div>

Note that even this "real" rule is just the building blocks from this page: a
`<rule>` with a `context`, an `<assert>` whose `test` states what must be true,
and a message emitted when it isn't.

## Next

[Context and tests](context-and-tests.md) — how the `context` XPath establishes
the node a rule runs against, and how `test` expressions are evaluated relative
to it.
