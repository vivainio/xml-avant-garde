# Simple types and restrictions

A **simple type** describes the text content of an element or attribute — no
child elements, no attributes of its own. XSD ships with built-ins like
`xsd:string`, `xsd:decimal`, `xsd:integer`, and `xsd:date`. You rarely use them
raw: instead you *derive* a tighter type by adding constraints, called
**facets**, on top of a base type.

The workhorse is `xsd:restriction`:

``` xml title="restriction-shape.xsd" linenums="1"
<xsd:simpleType name="ExampleType">
  <xsd:restriction base="xsd:string"> <!-- (1)! -->
    <xsd:maxLength value="20"/>        <!-- (2)! -->
  </xsd:restriction>
</xsd:simpleType>
```

1. `base` is the type you are narrowing — a built-in or another named simple type.
2. Each child element is a *facet*. Stack as many as you need; a value must satisfy all of them.

A value that matches the base type but violates any facet is invalid.

## Enumeration — closed value lists

`xsd:enumeration` is **the** code-list mechanism in XSD: list the permitted
values, and nothing else is allowed. UBL currency and document-type codes are
modelled exactly this way.

``` xml title="currency-code.xsd" linenums="1"
<xsd:simpleType name="CurrencyCodeType">
  <xsd:restriction base="xsd:string">
    <xsd:enumeration value="EUR"/>
    <xsd:enumeration value="USD"/>
    <xsd:enumeration value="GBP"/>
  </xsd:restriction>
</xsd:simpleType>
```

<div class="xslt-result" markdown>
`<cbc:DocumentCurrencyCode>EUR</cbc:DocumentCurrencyCode>` — **valid**

`<cbc:DocumentCurrencyCode>XYZ</cbc:DocumentCurrencyCode>` — **invalid** (not in the list)
</div>

!!! note
    The three codes above are an illustrative subset. Real code lists (ISO 4217
    currencies, UBL document-type codes) follow the same pattern with far more
    members, and are often kept in a separate code-list schema.

## Pattern — regular-expression constraint

`xsd:pattern` constrains the lexical form with a regular expression. The whole
value must match (the regex is implicitly anchored start-to-end). Use it for
structured identifiers.

``` xml title="invoice-id.xsd" linenums="1"
<xsd:simpleType name="InvoiceIdType">
  <xsd:restriction base="xsd:string">
    <xsd:pattern value="INV-[0-9]{3}"/>
  </xsd:restriction>
</xsd:simpleType>
```

<div class="xslt-result" markdown>
`<cbc:ID>INV-001</cbc:ID>` — **valid**

`<cbc:ID>INV-1</cbc:ID>` — **invalid** (needs exactly three digits)

`<cbc:ID>X-001</cbc:ID>` — **invalid** (wrong prefix)
</div>

!!! tip
    Multiple `xsd:pattern` facets on one type are combined with **OR** — a value
    matching any one of them passes.

## Length — `minLength`, `maxLength`, `length`

These constrain the number of characters of a string (or items of a list).

``` xml title="id-length.xsd" linenums="1"
<xsd:simpleType name="ShortIdType">
  <xsd:restriction base="xsd:string">
    <xsd:minLength value="3"/>
    <xsd:maxLength value="35"/>
  </xsd:restriction>
</xsd:simpleType>
```

Use `xsd:length` for a fixed, exact size:

``` xml title="country-code.xsd" linenums="1"
<xsd:simpleType name="CountryCodeType">
  <xsd:restriction base="xsd:string">
    <xsd:length value="2"/> <!-- (1)! -->
  </xsd:restriction>
</xsd:simpleType>
```

1. Exactly two characters — combine with `xsd:pattern` if you also need to restrict the alphabet.

<div class="xslt-result" markdown>
`FI` — **valid** against `CountryCodeType`

`FIN` — **invalid** (length must be exactly 2)
</div>

## Ranges — inclusive and exclusive bounds

For numeric and date/time types, the four bound facets set a permitted range.
`Inclusive` includes the boundary value; `Exclusive` excludes it.

``` xml title="quantity-range.xsd" linenums="1"
<xsd:simpleType name="LineQuantityType">
  <xsd:restriction base="xsd:decimal">
    <xsd:minExclusive value="0"/>   <!-- (1)! -->
    <xsd:maxInclusive value="1000"/> <!-- (2)! -->
  </xsd:restriction>
</xsd:simpleType>
```

1. Strictly greater than 0 — a zero quantity is rejected.
2. Up to and including 1000.

The same facets work on dates, e.g. constraining an `IssueDate` to not precede
a fixed start date:

``` xml title="issue-date.xsd" linenums="1"
<xsd:simpleType name="IssueDateType">
  <xsd:restriction base="xsd:date">
    <xsd:minInclusive value="2020-01-01"/>
  </xsd:restriction>
</xsd:simpleType>
```

## Decimal precision — `totalDigits` and `fractionDigits`

For monetary amounts, `totalDigits` caps the total number of significant digits
and `fractionDigits` caps the digits after the decimal point.

``` xml title="amount.xsd" linenums="1"
<xsd:simpleType name="AmountType">
  <xsd:restriction base="xsd:decimal">
    <xsd:totalDigits value="12"/>    <!-- (1)! -->
    <xsd:fractionDigits value="2"/>  <!-- (2)! -->
  </xsd:restriction>
</xsd:simpleType>
```

1. At most 12 significant digits in all.
2. At most 2 digits after the decimal point — the typical "minor unit" precision for money.

<div class="xslt-result" markdown>
`<cbc:LineExtensionAmount>10.90</cbc:LineExtensionAmount>` — **valid**

`<cbc:LineExtensionAmount>10.901</cbc:LineExtensionAmount>` — **invalid** (3 fraction digits)
</div>

!!! info
    In UBL the *numeric* part of an amount is a restricted `xsd:decimal` like
    this, while the currency is carried separately in a `currencyID` attribute
    typed by an enumeration such as `CurrencyCodeType` above.

## Whitespace handling — `whiteSpace`

`xsd:whiteSpace` controls how whitespace in the lexical value is processed
before other facets are checked:

- `preserve` — leave the value exactly as written.
- `replace` — turn each tab, newline, and carriage return into a space.
- `collapse` — apply `replace`, then trim leading/trailing spaces and collapse internal runs to a single space.

``` xml title="collapse.xsd" linenums="1"
<xsd:simpleType name="CodeType">
  <xsd:restriction base="xsd:string">
    <xsd:whiteSpace value="collapse"/>
  </xsd:restriction>
</xsd:simpleType>
```

!!! note
    String-derived built-ins like `xsd:token` already imply `collapse`; the
    other built-ins (`xsd:decimal`, `xsd:date`, …) imply it too. You mostly set
    this facet explicitly only when deriving directly from `xsd:string`.

## Facet summary

| Facet | Constrains | Example |
|-------|------------|---------|
| `xsd:enumeration` | Allowed values (closed list) | `value="EUR"` |
| `xsd:pattern` | Lexical form (regex) | `value="INV-[0-9]{3}"` |
| `xsd:length` | Exact string/list size | `value="2"` |
| `xsd:minLength` / `xsd:maxLength` | String/list size bounds | `min 3`, `max 35` |
| `xsd:minInclusive` / `xsd:maxInclusive` | Range, boundary included | `0 ≤ x ≤ 1000` |
| `xsd:minExclusive` / `xsd:maxExclusive` | Range, boundary excluded | `x > 0` |
| `xsd:totalDigits` | Total significant digits | `value="12"` |
| `xsd:fractionDigits` | Digits after the point | `value="2"` |
| `xsd:whiteSpace` | Whitespace processing | `preserve` / `replace` / `collapse` |

## Lists and unions

Two more ways to build a simple type, without `xsd:restriction`.

### `xsd:list` — a whitespace-separated sequence

A list type holds several values of one item type, separated by whitespace.

``` xml title="code-list.xsd" linenums="1"
<xsd:simpleType name="CurrencyCodeListType">
  <xsd:list itemType="CurrencyCodeType"/> <!-- (1)! -->
</xsd:simpleType>
```

1. Each item must independently satisfy `CurrencyCodeType` (the enumeration above).

<div class="xslt-result" markdown>
`EUR USD GBP` — **valid** (three list items)

`EUR XYZ` — **invalid** (`XYZ` is not a member)
</div>

### `xsd:union` — valid against any member type

A union accepts a value that matches *any* of its member types — handy when one
field can carry either of two distinct shapes.

``` xml title="reference-union.xsd" linenums="1"
<xsd:simpleType name="InvoiceReferenceType">
  <xsd:union memberTypes="InvoiceIdType CurrencyCodeType"/> <!-- (1)! -->
</xsd:simpleType>
```

1. A value is valid if it matches the `INV-NNN` pattern **or** is one of the currency codes.

<div class="xslt-result" markdown>
`INV-001` — **valid** (matches `InvoiceIdType`)

`EUR` — **valid** (matches `CurrencyCodeType`)

`abc` — **invalid** (matches neither member)
</div>

!!! tip
    `xsd:list` and `xsd:union` can themselves be wrapped in an `xsd:restriction`
    — for example, to bound a list's length or enumerate the allowed union
    values — letting you layer the three derivation styles.

## Next

Continue with [Modular schemas](modular-schemas.md) to see how these named
simple types are split across files and reused — the way UBL keeps its code
lists and component definitions in separate, importable schemas.
