---
icon: lucide/calendar-clock
---

# Dates and times

XSLT 1.0 had no date type at all — you sliced date strings with `substring` and
hoped. 2.0/3.0 bring the full [XML Schema](../xsd/index.md) date/time types into
[XPath](../xpath/index.md), with real arithmetic and a flexible formatting
function. This is one of the clearest "impossible in 1.0, trivial now" wins.

## The types

`xs:date`, `xs:dateTime`, `xs:time`, plus the two duration types `xs:dayTimeDuration`
and `xs:yearMonthDuration`. You get the current values from functions, or cast a
string:

``` xml
<xsl:variable name="today"  select="current-date()"/>      <!-- 2025-03-14 -->
<xsl:variable name="now"    select="current-dateTime()"/>
<xsl:variable name="issued" select="xs:date(invoice/@date)"/>
```

## Arithmetic

Subtracting two dates gives a **duration**; adding a duration to a date gives a
date. It just works, time zones and leap years included:

``` xml
<!-- how many days until the invoice is due? -->
<xsl:variable name="age" select="current-date() - xs:date(invoice/@date)"/>
<xsl:value-of select="days-from-duration($age)"/>

<!-- 30 days after issue -->
<xsl:value-of select="xs:date(invoice/@date) + xs:dayTimeDuration('P30D')"/>
```

Comparisons (`<`, `>`, `=`) work on dates directly, so sorting and filtering by
date need no string trickery:

``` xml
<xsl:for-each select="cd[xs:integer(year) ge 1985]">…</xsl:for-each>
```

## Formatting: `format-date`

`format-date`, `format-dateTime`, and `format-time` turn a typed value into a
string using a **picture string** of `[component]` markers:

``` xml
<xsl:value-of select="format-date(current-date(), '[D1] [MNn] [Y0001]')"/>
<!-- 14 March 2025 -->

<xsl:value-of select="format-dateTime(current-dateTime(), '[H01]:[m01]:[s01]')"/>
<!-- 09:07:42 -->
```

The common components:

| Marker | Produces |
| --- | --- |
| `[Y]` / `[Y0001]` | year / zero-padded to 4 digits |
| `[M]` / `[M01]` | month number / 2-digit |
| `[MNn]` / `[MNn,3-3]` | month name / abbreviated to 3 letters |
| `[D]` / `[D1]` / `[D01]` | day of month |
| `[FNn]` | weekday name (Monday…) |
| `[H01]` / `[h]` | hour, 24- / 12-hour |
| `[m01]` `[s01]` | minutes, seconds |
| `[P]` | a.m./p.m. |

!!! tip "Language and calendars"
    A fifth argument selects the language: `format-date($d, '[FNn] [D] [MNn]',
    'fi', (), ())` yields Finnish weekday and month names where the processor
    supports them. The same picture-string mechanism powers
    [`format-number`](number-formatting.md) for numeric output.

## Components without formatting

When you just want one field as a number, the `*-from-*` accessors are simpler
than a picture string:

``` xml
year-from-date($d)    month-from-date($d)    day-from-date($d)
hours-from-dateTime($dt)   minutes-from-dateTime($dt)   seconds-from-dateTime($dt)
timezone-from-dateTime($dt)
```

For the weekday, there is no `*-from-*` accessor — use a picture string:
`format-date($d, '[F]')` gives the day-of-week number, `[FNn]` the name.

## Where to go next

- [Number formatting](number-formatting.md) — the same picture-string idea for numbers.
- [Regular expressions and strings](regex.md) — when the input is a non-standard date string you must parse first.
