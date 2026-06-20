---
icon: lucide/braces
---

# Maps and arrays

XSLT 3.1 (with [XPath](../xpath/index.md) 3.1) adds two structured item types
that XML alone never had: the **map**, a key-to-value dictionary, and the
**array**, an ordered list that — unlike a [sequence](sequences-and-types.md) —
can nest. They are the in-memory data structures you reach for when a problem is
about *lookup* and *structure* rather than *documents*, and they are the bridge
to [JSON](json.md).

## Maps

A map associates keys with values. Build one with the `map { … }` constructor;
keys and values are any atomic value or sequence:

``` xml
<xsl:variable name="cd" select="map {
  'title': 'Empire Burlesque',
  'artist': 'Bob Dylan',
  'price': 10.90
}"/>
```

Look a value up with the `?` operator (or `map:get`):

``` xml
<xsl:value-of select="$cd?title"/>            <!-- Empire Burlesque -->
<xsl:value-of select="map:get($cd, 'price')"/> <!-- 10.9 -->
```

Maps are **immutable** — `map:put`, `map:remove`, and `map:merge` return a *new*
map rather than changing the original. The common idioms:

| Function / syntax | Does |
| --- | --- |
| `$m?key` / `map:get($m,'key')` | look up a value |
| `map:put($m, 'k', $v)` | a copy with one entry added/replaced |
| `map:merge(($m1, $m2))` | combine maps (later wins, by default) |
| `map:keys($m)` / `map:size($m)` | the keys / entry count |
| `map:for-each($m, $f)` | apply a 2-arg function to each entry |

### Maps as lookup tables

The classic use is an in-memory index — a faster, more local alternative to
[`xsl:key`](keys.md) when the data fits in memory. Build a map once, then look
up by key in O(1):

``` xml
<!-- index every cd by genre -->
<xsl:variable name="byGenre" as="map(xs:string, element(cd)*)">
  <xsl:sequence select="map:merge(
    /catalog/cd ! map { string(@genre): . },
    map { 'duplicates': 'combine' })"/>
</xsl:variable>

<xsl:value-of select="$byGenre?rock/title"/>   <!-- Empire Burlesque -->
```

## Arrays

An array is an ordered list of **members**, where each member is itself a
sequence. That nesting is the whole reason arrays exist: `(1, (2, 3))` collapses
to four-less `(1, 2, 3)` as a sequence, but `[1, [2, 3]]` keeps its shape as an
array.

``` xml
<xsl:variable name="a" select="[ 'rock', 'pop', 'country' ]"/>
<xsl:value-of select="$a(2)"/>                 <!-- pop  (1-based) -->
<xsl:value-of select="array:size($a)"/>        <!-- 3 -->
```

| Function / syntax | Does |
| --- | --- |
| `$a(2)` / `array:get($a,2)` | the member at position 2 (1-based) |
| `array:size($a)` | member count |
| `array:append($a, $v)` | a copy with `$v` added |
| `array:flatten($a)` | collapse to a flat sequence |
| `$a?*` | all members as a sequence |

## Why both, and when

- A **sequence** is the default for "a bunch of nodes/values" — flat, ordered,
  what every XPath returns.
- A **map** is for **named** access — config, lookup tables, "return several
  values from one function" without inventing a wrapper element.
- An **array** is for **positional, nested** structure — a matrix, a JSON array
  of objects, anything a flat sequence would smear together.

!!! tip "Returning several values cleanly"
    Before maps, a [function](functions.md) that needed to return "the count
    *and* the total" had to build a throwaway element. Now it returns a map:

    ``` xml
    <xsl:function name="local:stats" as="map(*)">
      <xsl:param name="cds" as="element(cd)*"/>
      <xsl:sequence select="map {
        'count': count($cds),
        'total': sum($cds/xs:decimal(price))
      }"/>
    </xsl:function>
    ```

## Where to go next

- [Reading and writing JSON](json.md) — maps and arrays *are* the JSON data model.
- [Higher-order functions](higher-order-functions.md) — `map:for-each` and friends take function values.
- [Sequences and types](sequences-and-types.md) — how `map(*)` and `array(*)` fit the type system.
