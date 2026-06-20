---
icon: lucide/book-marked
---

# New functions in 2.0 / 3.0 / 3.1

The 1.0 function library is small — `count`, `string`, `concat`, `substring`,
`translate`, and a handful more. Every version since has added to it. The big
additions get their own pages ([regex](regex.md), [dates](dates-and-times.md),
[higher-order](higher-order-functions.md), [maps & arrays](maps-and-arrays.md),
[JSON](json.md), [text I/O](text-and-parsing.md)); this page is a **reference**
for the smaller but constantly-useful functions that don't each need a chapter.

The version column is when the function appeared (`2.0`, `3.0`, `3.1`). All of
these work in the free [Saxon-HE](moving-to-3.md).

## Working with sequences

| Function | Does | Ver |
| --- | --- | --- |
| `distinct-values($seq)` | drop duplicate atomic values | 2.0 |
| `index-of($seq, $v)` | positions where `$v` occurs | 2.0 |
| `subsequence($seq, $start [, $len])` | a slice, 1-based | 2.0 |
| `reverse($seq)` | reverse order | 2.0 |
| `insert-before` / `remove($seq, $pos …)` | add / drop items by position | 2.0 |
| `head($seq)` / `tail($seq)` | first item / everything-but-first | 3.0 |
| `exists($seq)` / `empty($seq)` | is the sequence non-empty / empty | 2.0 |
| `deep-equal($a, $b)` | recursive value+structure equality | 2.0 |

``` xml
<xsl:value-of select="distinct-values(/catalog/cd/@genre)"/>   <!-- country pop rock -->
<xsl:value-of select="head(/catalog/cd)/title"/>               <!-- Empire Burlesque -->
```

## Aggregates and numbers

| Function | Does | Ver |
| --- | --- | --- |
| `avg($seq)` / `min($seq)` / `max($seq)` | average / smallest / largest | 2.0 |
| `abs($n)` | absolute value | 2.0 |
| `round-half-to-even($n [, $p])` | banker's rounding | 2.0 |
| `format-integer($n, $picture)` | `1` → `i`, `one`, `001`, … | 3.0 |
| `random-number-generator([$seed])` | a map yielding pseudo-random numbers | 3.1 |

``` xml
<xsl:value-of select="avg(/catalog/cd/xs:decimal(price))"/>    <!-- 10.233… -->
<xsl:value-of select="format-integer(4, 'I')"/>                <!-- IV -->
```

## Strings

Most string work lives on the [regex & strings page](regex.md); the essentials:

| Function | Does | Ver |
| --- | --- | --- |
| `string-join($seq [, $sep])` | concatenate with a separator | 2.0 |
| `upper-case` / `lower-case($s)` | case conversion | 2.0 |
| `normalize-unicode($s)` | Unicode normalisation (NFC, …) | 2.0 |
| `string-to-codepoints` / `codepoints-to-string` | string ⇄ code points | 2.0 |
| `tokenize` / `replace` / `matches` | [see regex](regex.md) | 2.0 |

## Nodes and documents

| Function | Does | Ver |
| --- | --- | --- |
| `doc($uri)` / `doc-available($uri)` | load an XML document / test it loads | 2.0 |
| `collection($uri)` / `uri-collection($uri)` | a set of documents / their URIs | 2.0 / 3.0 |
| `path($node)` | the node's XPath location, as a string | 3.0 |
| `has-children($node)` | does it have child nodes | 3.0 |
| `innermost($seq)` / `outermost($seq)` | strip ancestors / descendants from a node set | 3.0 |
| `unparsed-text` / `parse-xml` / `serialize` | [text & string I/O](text-and-parsing.md) | 2.0 / 3.0 |

``` xml
<xsl:if test="doc-available('prices.xml')">
  <xsl:value-of select="doc('prices.xml')/prices/total"/>
</xsl:if>
```

## Diagnostics

| Function | Does | Ver |
| --- | --- | --- |
| `trace($value, $label)` | log `$value` to the processor's trace output, return it unchanged | 2.0 |
| `error([$code [, $desc]])` | abort with an error — see [error handling](error-handling.md) | 2.0 |

`trace` is the printf-debugging of XSLT: wrap any sub-expression in it and watch
what flows through without altering the result.

## The bigger additions (their own pages)

For completeness, the major library growth covered elsewhere:

- **Regex & strings** — `tokenize`, `replace`, `matches` → [regex](regex.md)
- **Dates** — `format-date`, `current-dateTime`, the `*-from-*` accessors → [dates and times](dates-and-times.md)
- **Higher-order** — `for-each`, `filter`, `fold-left`/`-right`, `sort`, `apply` → [higher-order functions](higher-order-functions.md)
- **Maps & arrays** — the `map:*` and `array:*` namespaces → [maps and arrays](maps-and-arrays.md)
- **JSON** — `parse-json`, `json-doc`, `json-to-xml`, `xml-to-json` → [JSON](json.md)

## Where to go next

- [Higher-order functions](higher-order-functions.md) — passing these functions around as values.
- [Reading and writing non-XML text](text-and-parsing.md) — `unparsed-text`, `parse-xml`, `serialize` in depth.
