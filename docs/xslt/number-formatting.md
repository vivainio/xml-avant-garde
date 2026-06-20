# Number formatting

Raw numbers in XML rarely look the way you want them on screen: a price stored
as `10.9` should read `10.90`, a large total wants thousands separators, and a
ratio is friendlier as a percentage. `format-number` turns a numeric value into
a **string** shaped by a pattern, and `xsl:decimal-format` lets you adapt that
shaping to other locales.

## The running example

A small catalog with a `<price>` on each item:

``` xml title="catalog.xml" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<catalog>
  <item><name>Notebook</name><price>10.90</price></item>
  <item><name>Pen</name><price>9.90</price></item>
  <item><name>Stapler</name><price>1250.5</price></item>
</catalog>
```

## format-number

`format-number(value, pattern, decimal-format-name?)` formats `value` according
to `pattern` and returns the result as a **string** (never a number, so it is
ready to drop into text). The third argument is optional and names an
`xsl:decimal-format`; leave it off to use the default.

``` xml
<xsl:value-of select="format-number(10.9, '0.00')"/>
```

<div class="xslt-result" markdown>
10.90
</div>

## Pattern characters

The pattern is a template of placeholder characters. The two that do the heavy
lifting are `0` and `#`: a `0` is a digit position that is **always** shown
(padding with a zero if there is nothing there), while a `#` is shown **only**
if a digit is present.

| Symbol | Meaning | Example | Result |
| --- | --- | --- | --- |
| `0` | Required digit (pad with `0`) | `format-number(7, '000')` | `007` |
| `#` | Optional digit (shown only if present) | `format-number(7, '###')` | `7` |
| `.` | Decimal point | `format-number(7, '0.0')` | `7.0` |
| `,` | Grouping (thousands) separator | `format-number(1250.5, '#,##0')` | `1,251` |
| `%` | Percent — **multiplies the value by 100** | `format-number(0.25, '0%')` | `25%` |
| `;` | Separates positive and negative subpatterns | `format-number(-3, '0;(0)')` | `(3)` |

## Money: a fixed number of decimals

The pattern `'0.00'` forces exactly two decimal places. The integer `0` before
the point guarantees at least one leading digit, so values below 1 still show a
`0`:

``` xml
<xsl:for-each select="catalog/item">
  <xsl:value-of select="concat(name, ': ', format-number(price, '0.00'))"/>
  <xsl:text>&#10;</xsl:text>
</xsl:for-each>
```

<div class="xslt-result" markdown>
Notebook: 10.90

Pen: 9.90

Stapler: 1250.50
</div>

## Grouped numbers: thousands separators

Add a `,` to the integer part to switch on grouping. The conventional money
pattern combines grouping with two decimals as `'#,##0.00'`. The leading `#`s
are optional digit positions, while the final `0` keeps at least one integer
digit:

``` xml
<xsl:value-of select="format-number(1250.5, '#,##0.00')"/>
```

<div class="xslt-result" markdown>
1,250.50
</div>

!!! note "Where you put the comma does not fix the group size"
    The group size is taken from the distance between the `,` and the decimal
    point (or the end of the pattern). `'#,##0.00'` groups by three. You rarely
    need to write more than one `,` — one is enough to enable grouping.

## Percentages

A `%` in the pattern multiplies the value by 100 and appends the percent sign.
This means a stored ratio like `0.0825` formats directly, with no manual
arithmetic:

``` xml
<xsl:value-of select="format-number(0.0825, '0.0%')"/>   <!-- (1)! -->
```

1.  `0.0825 × 100 = 8.25`, then rounded to one decimal place by `0.0%`.

<div class="xslt-result" markdown>
8.3%
</div>

## Negative numbers: the two-subpattern form

A pattern may carry two subpatterns separated by `;`. The first is used for
positive (and zero) values, the second for negatives. Only the prefix and
suffix of the negative subpattern matter — its digits reuse the positive
layout. This is the standard way to render losses in parentheses:

``` xml
<xsl:value-of select="format-number(1250.5, '#,##0.00;(#,##0.00)')"/>   <!-- (1)! -->
<xsl:value-of select="format-number(-1250.5, '#,##0.00;(#,##0.00)')"/>  <!-- (2)! -->
```

1.  Positive value uses the first subpattern.
2.  Negative value uses the second: parentheses instead of a minus sign.

<div class="xslt-result" markdown>
1,250.50

(1,250.50)
</div>

!!! tip "Default negative handling"
    With a single subpattern, negatives simply get a leading `-`. You only need
    the `;` form when you want a different prefix or suffix, such as parentheses
    or a trailing `CR`.

## Rounding

`format-number` **rounds** the value to the precision the pattern asks for — it
does not truncate. Half-way values round up:

``` xml
<xsl:value-of select="format-number(2.345, '0.00')"/>   <!-- (1)! -->
<xsl:value-of select="format-number(2.344, '0.00')"/>   <!-- (2)! -->
```

1.  Third decimal is `5`, so it rounds up to `2.35`.
2.  Third decimal is `4`, so it rounds down to `2.34`.

<div class="xslt-result" markdown>
2.35

2.34
</div>

## Other locales: xsl:decimal-format

Not every locale uses `.` for the decimal point and `,` for grouping — much of
Europe does the opposite. Rather than rewrite every pattern, declare a named
`xsl:decimal-format` at the **top level** of the stylesheet (a direct child of
`xsl:stylesheet`) that redefines the separators, then pass its name as the
third argument to `format-number`:

``` xml title="prices-eu.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
<xsl:output method="text"/>

<xsl:decimal-format name="euro" decimal-separator="," grouping-separator="."/>  <!-- (1)! -->

<xsl:template match="/">
  <xsl:for-each select="catalog/item">
    <xsl:value-of select="concat(name, ': ',
      format-number(price, '#.##0,00', 'euro'))"/>                              <!-- (2)! -->
    <xsl:text>&#10;</xsl:text>
  </xsl:for-each>
</xsl:template>
</xsl:stylesheet>
```

1.  In the `euro` format the roles swap: `,` is the decimal point and `.` is the
    grouping separator.
2.  The third argument selects the `euro` format. The pattern itself still uses
    the abstract symbols `.` (grouping) and `,` (decimal) **as redefined** by
    that format.

<div class="xslt-result" markdown>
Notebook: 10,90

Pen: 9,90

Stapler: 1.250,50
</div>

!!! warning "The pattern follows the format's symbols"
    Once a `decimal-format` swaps the separators, you write the pattern using
    the new meanings. Under `euro`, the grouped-money pattern becomes
    `'#.##0,00'` — `.` now groups and `,` is the decimal point. An unnamed
    `xsl:decimal-format` (no `name` attribute) redefines the **default** format
    used whenever the third argument is omitted.

## Related XPath functions

`format-number` produces display strings; these XPath 1.0 functions work on the
numbers themselves, before or instead of formatting.

| Function | Does | Example → result |
| --- | --- | --- |
| `number(s)` | Parse a string into a number | `number('10.90')` → `10.9` |
| `round(x)` | Nearest integer (half rounds up) | `round(2.5)` → `3` |
| `floor(x)` | Largest integer ≤ `x` | `floor(2.9)` → `2` |
| `ceiling(x)` | Smallest integer ≥ `x` | `ceiling(2.1)` → `3` |
| `sum(node-set)` | Total of the numeric values | `sum(catalog/item/price)` → `1271.3` |

A common pairing is `sum` to total a node-set, then `format-number` to present
it:

``` xml
<xsl:value-of select="format-number(sum(catalog/item/price), '#,##0.00')"/>
```

<div class="xslt-result" markdown>
1,271.30
</div>

!!! info "number() vs. automatic coercion"
    Arithmetic and the numeric functions coerce strings to numbers
    automatically, so explicit `number()` is mostly for clarity or to test
    whether a string is numeric (a non-numeric string yields `NaN`).

## Next

[Whitespace and xsl:text](whitespace.md) — formatted numbers are only as tidy as
the text around them, so the next step is controlling stray spaces and newlines.
