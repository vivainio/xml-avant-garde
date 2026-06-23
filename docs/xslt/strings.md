# String functions

XPath 1.0 ships a small set of string functions, and they are the same ones you
use inside XSLT. They are enough for most text wrangling: joining, slicing,
testing, trimming and character-mapping. There is **no** regular-expression or
generic replace function in XSLT 1.0 — that toolbox is `translate` together with
`substring-before` / `substring-after`.

## At a glance

| Function | Does | Tiny example → result |
| --- | --- | --- |
| `concat(a, b, …)` | Joins all arguments into one string | `concat('a', '-', 'b')` → `a-b` |
| `substring(s, start, len?)` | Slice; **1-based**, `len` optional | `substring('abcdef', 2, 3)` → `bcd` |
| `substring-before(s, sep)` | Part before first `sep` | `substring-before('a=1', '=')` → `a` |
| `substring-after(s, sep)` | Part after first `sep` | `substring-after('a=1', '=')` → `1` |
| `string-length(s)` | Number of characters | `string-length('abc')` → `3` |
| `contains(s, sub)` | True if `sub` occurs in `s` | `contains('abc', 'b')` → `true` |
| `starts-with(s, prefix)` | True if `s` begins with `prefix` | `starts-with('abc', 'ab')` → `true` |
| `normalize-space(s)` | Trim ends, collapse internal runs | `normalize-space('  a   b ')` → `a b` |
| `translate(s, from, to)` | Per-character replacement | `translate('abc', 'ac', 'AC')` → `AbC` |
| `string(x)` | Coerce a value to a string | `string(10.90)` → `10.90` |

## Joining: concat

`concat` takes two or more arguments and glues them together. It is the usual
way to build a string out of element values and literals:

``` xml
<xsl:value-of select="concat(artist, ' — ', title)"/>
```

For the first CD in the [catalog](index.md#the-running-example) that produces:

<div class="xslt-result" markdown>
Bob Dylan — Empire Burlesque
</div>

## Slicing: substring

`substring(s, start, len)` returns part of a string. Two things to remember:
positions are **1-based** (the first character is at position 1, not 0), and the
`len` argument is **optional** — leave it off to take the rest of the string.

``` xml
<xsl:value-of select="substring('Empire Burlesque', 1, 6)"/>   <!-- (1)! -->
<xsl:value-of select="substring('Empire Burlesque', 8)"/>      <!-- (2)! -->
```

1.  Start at character 1, take 6 → `Empire`.
2.  Start at character 8, take the rest → `Burlesque`.

<div class="xslt-result" markdown>
Empire

Burlesque
</div>

## Splitting around a delimiter

`substring-before` and `substring-after` cut a string at the **first**
occurrence of a separator. They are the closest thing XSLT 1.0 has to a split.

``` xml
<xsl:value-of select="substring-before('key=value', '=')"/>   <!-- (1)! -->
<xsl:value-of select="substring-after('key=value', '=')"/>    <!-- (2)! -->
```

1.  Everything before the first `=` → `key`.
2.  Everything after the first `=` → `value`.

<div class="xslt-result" markdown>
key

value
</div>

!!! warning "Missing delimiter returns empty string"
    If the separator is **not** present, both functions return the empty string
    — not the original string. So `substring-before('plain', '=')` is `''`, and
    so is `substring-after('plain', '=')`. Guard with `contains` first if a
    string might not hold the delimiter.

## Length and tests

`string-length` counts characters. `contains` and `starts-with` return booleans,
which makes them ideal in predicates and in `xsl:if`:

``` xml
<xsl:if test="contains(artist, 'Dylan')">
  <p>Found a Dylan record: <xsl:value-of select="title"/></p>
</xsl:if>

<xsl:value-of select="catalog/cd[starts-with(artist, 'Bo')]/artist"/>
```

The predicate `[starts-with(artist, 'Bo')]` keeps the CDs by *Bob Dylan* and
*Bonnie Tyler*:

<div class="xslt-result" markdown>
Found a Dylan record: Empire Burlesque

Bob DylanBonnie Tyler
</div>

## Cleaning whitespace: normalize-space

`normalize-space` strips leading and trailing whitespace **and** collapses every
internal run of spaces, tabs and newlines down to a single space. It is the
standard way to tidy up the messy whitespace that comes with mixed content, and
it is one of the most heavily used functions in real stylesheets.

``` xml
<xsl:value-of select="normalize-space('   Bob    Dylan
   ')"/>
```

<div class="xslt-result" markdown>
Bob Dylan
</div>

!!! tip "Use it defensively"
    Source XML is often pretty-printed, leaving stray newlines and indentation
    inside text nodes. Wrapping a value in `normalize-space(...)` before you
    compare, concat or output it saves a lot of grief.

## Character mapping: translate

`translate(s, from, to)` replaces characters one at a time: every character in
`s` that appears at position *n* of `from` is swapped for the character at
position *n* of `to`. It does **not** replace substrings — it works
character-by-character.

The classic XSLT 1.0 idiom is case conversion, since 1.0 has no `upper-case()`
function (that arrived in 2.0):

``` xml
<xsl:variable name="lower" select="'abcdefghijklmnopqrstuvwxyz'"/>
<xsl:variable name="upper" select="'ABCDEFGHIJKLMNOPQRSTUVWXYZ'"/>

<xsl:value-of select="translate(artist, $lower, $upper)"/>   <!-- (1)! -->
```

1.  Each lowercase letter is mapped to its uppercase counterpart.

<div class="xslt-result" markdown>
BOB DYLAN
</div>

`translate` is also the 1.0 way to **strip** characters: when `to` is shorter
than `from`, any `from` character with no partner in `to` is deleted. Mapping
the unwanted characters to nothing removes them:

``` xml
<xsl:value-of select="translate('(0)20-555 1234', ' ()-', '')"/>   <!-- (1)! -->
```

1.  Spaces, parentheses and hyphens all have no counterpart, so they vanish →
    `0205551234`.

<div class="xslt-result" markdown>
0205551234
</div>

## Coercion: string

`string(x)` converts any value — a number, a boolean, a node-set — into its
string form. Most XSLT contexts coerce automatically, but `string()` is useful
when you want to be explicit, for example forcing a number into text:

``` xml
<xsl:value-of select="string(10.90)"/>
```

<div class="xslt-result" markdown>
10.90
</div>

## Reformat an artist name

Suppose you want to turn `Bob Dylan` into `DYLAN, Bob` — surname first, in caps,
then the given name. Splitting on the space gives both halves; `translate`
uppercases the surname; `concat` reassembles:

``` xml title="reformat-artist.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
<xsl:output method="text"/>

<xsl:variable name="lower" select="'abcdefghijklmnopqrstuvwxyz'"/>
<xsl:variable name="upper" select="'ABCDEFGHIJKLMNOPQRSTUVWXYZ'"/>

<xsl:template match="/">
  <xsl:for-each select="catalog/cd">
    <xsl:variable name="name" select="normalize-space(artist)"/>     <!-- (1)! -->
    <xsl:variable name="given" select="substring-before($name, ' ')"/>  <!-- (2)! -->
    <xsl:variable name="surname" select="substring-after($name, ' ')"/> <!-- (3)! -->
    <xsl:value-of select="concat(translate($surname, $lower, $upper),
                                 ', ', $given)"/>                     <!-- (4)! -->
    <xsl:text>&#10;</xsl:text>
  </xsl:for-each>
</xsl:template>
</xsl:stylesheet>
```

1.  Tidy whitespace first, so the split is reliable.
2.  Text before the first space — the given name.
3.  Text after the first space — the surname.
4.  Uppercase the surname, then join `SURNAME, Given`.

Run against the [catalog](index.md#the-running-example):

<div class="xslt-result" markdown>
DYLAN, Bob

TYLER, Bonnie

PARTON, Dolly
</div>

!!! info "XSLT 2.0 makes some of this easier"
    2.0 adds `upper-case()`, `lower-case()`, `replace()` (regex-based) and
    `tokenize()` (splitting into a sequence). If you are on a 1.0 processor,
    `translate` plus `substring-before` / `substring-after` cover the same
    ground with a little more work.

## Next

[Producing XML output](output.md) — now that you can shape text, the next step
is controlling the structure and format of what your stylesheet emits.
