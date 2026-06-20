---
icon: lucide/blocks
---

# Combining XSLT and XQuery

A question that comes up the moment you have both languages on the same engine —
typically [Saxon](https://www.saxonica.com/), which implements XSLT 3.0 and
XQuery 3.1 side by side — is: *can I call XQuery from my XSLT?* The short answer
is **no, not by embedding it** — but the two were designed to interoperate, just
at a different level than nesting one inside the other.

## You cannot embed XQuery in XSLT

There is **no XSLT instruction that runs an XQuery query**, and you **cannot
`import module`** an XQuery library or call an XQuery `declare function` from an
[`xsl:function`](../xslt/functions.md) or an [XPath](../xpath/index.md)
expression. XSLT and XQuery are two separate *host* languages; they do not nest.

The reason you rarely need them to is that they share their entire foundation —
[XPath](../xpath/index.md) 3.1, the XDM data model, and the `fn:`/`map:`/`array:`
function library. Most of "XQuery" is therefore *already available* to you as
plain XPath inside a stylesheet.

## What carries over for free

These are **XPath** expressions, valid in any `select` or `xsl:variable`:

``` xml
<xsl:variable name="titles" select="for $cd in /catalog/cd return $cd/title"/>
<xsl:variable name="cheap"  select="$prices => sort() => head()"/>
<xsl:value-of select="let $n := count(/catalog/cd) return $n * 2"/>
```

`for … return`, `let … return`, `some … satisfies`, `every`, the arrow operator
`=>`, maps, arrays, and higher-order functions all behave identically to their
XQuery counterparts — because they *are* the same XPath.

## What does not — and its XSLT equivalent

What you **cannot** paste into XSLT is the rest of a [FLWOR](flwor.md)
expression. `where`, `order by`, `group by`, `count`, and the window clauses are
**XQuery-only clauses**, not part of XPath. In XSLT you express them with
instructions instead:

| XQuery FLWOR clause | XSLT 3.0 equivalent |
| --- | --- |
| `for … return` | XPath `for` (works as-is) or [`xsl:for-each`](../xslt/loops-and-output.md) |
| `let … :=` | XPath `let` (works as-is) or [`xsl:variable`](../xslt/variables.md) |
| `where` | a predicate `[…]` or [`xsl:if`](../xslt/conditionals.md) |
| `order by` | [`xsl:sort`](../xslt/sorting.md) |
| `group by` | [`xsl:for-each-group`](../xslt/grouping.md) |
| `declare function` | [`xsl:function`](../xslt/functions.md) |

So the practical answer to "I have an XQuery and want it in my stylesheet" is
usually: **rewrite the query logic in XSLT 3.0**, which can do essentially
everything XQuery 3.1 can — see [XQuery vs XSLT](xquery-vs-xslt.md) for where
each is the more natural fit.

### The same query, both ways

Here is one query that exercises four of the table's rows at once — `where`,
`group by`, `order by`, and an element constructor — written first as a FLWOR
and then as its line-for-line XSLT 3.0 translation, both over the
[running CD catalog](index.md#the-running-example):

=== "XQuery (FLWOR)"

    ``` xquery
    <genres>{
      for $cd in /catalog/cd
      where xs:decimal($cd/price) >= 9.90      (1)
      group by $g := $cd/@genre                (2)
      order by $g                              (3)
      return <genre name="{ $g }"              (4)
                    count="{ count($cd) }"/>
    }</genres>
    ```

    1. `where` clause.
    2. `group by` — `$cd` becomes the whole group in the `return`.
    3. `order by` the grouping key.
    4. a direct element constructor with `{ … }` holes.

=== "XSLT 3.0"

    ``` xml
    <xsl:template match="/catalog">
      <genres>
        <xsl:for-each-group select="cd[xs:decimal(price) >= 9.90]"   (1)
                            group-by="@genre">                       (2)
          <xsl:sort select="current-grouping-key()"/>                (3)
          <genre name="{current-grouping-key()}"                     (4)
                 count="{count(current-group())}"/>
        </xsl:for-each-group>
      </genres>
    </xsl:template>
    ```

    1. `where` → the predicate `[xs:decimal(price) >= 9.90]`.
    2. `group by` → `xsl:for-each-group/@group-by`; the key is
       `current-grouping-key()`, the group is `current-group()`.
    3. `order by` → [`xsl:sort`](../xslt/sorting.md).
    4. the constructor → a literal result element with
       [attribute value templates](../xslt/loops-and-output.md) `{…}`.

Both emit the same result:

``` xml title="result"
<genres>
  <genre name="country" count="1"/>
  <genre name="pop" count="1"/>
  <genre name="rock" count="1"/>
</genres>
```

The two remaining rows are simpler: `for … return` and `let … :=` need no
translation at all — they are [shown above](#what-carries-over-for-free) working
verbatim as XPath inside a `select`.

### Maps and arrays: identical in both

The convergence is total when the result is **maps and arrays** rather than
elements. The `map { … }` and `array { … }` constructors are
[XPath](../xpath/index.md) 3.1, not clauses of either host language — so a query
that builds JSON-shaped output, iterating with the XPath-level `for`, is the
*same expression* in XQuery and XSLT, down to the character:

=== "XQuery"

    ``` xquery
    array {
      for $cd in /catalog/cd
      return map { 'title': string($cd/title), 'price': xs:decimal($cd/price) }
    }
    ```

=== "XSLT 3.0"

    ``` xml
    <xsl:variable name="cds" select="
      array {
        for $cd in /catalog/cd
        return map { 'title': string($cd/title), 'price': xs:decimal($cd/price) }
      }
    "/>
    ```

Only the wrapper differs — a bare expression in XQuery, an `xsl:variable/@select`
in XSLT. Both build the identical XDM array, and both serialize it the same way
(`method="json"`, or [`fn:serialize`](../xslt/json.md) with a JSON output map).
The languages diverge again *only* if you add a `where`, `order by`, or
`group by` — at which point the [translation above](#the-same-query-both-ways)
applies.

## `fn:transform()` — the one-way bridge that exists

[XPath](../xpath/functions-and-types.md) 3.1 adds a built-in `fn:transform()`
that **runs an XSLT transformation**, and it is available from *both* XSLT and
XQuery. So you can invoke a stylesheet from a query (and chain XSLT from XSLT):

``` xquery title="XQuery driving an XSLT transform"
let $result := fn:transform(map {
  "stylesheet-location": "to-html.xsl",
  "source-node": doc("catalog.xml")
})
return $result?output
```

Note the asymmetry that answers the original question: there is **no
`fn:query()`** counterpart. You can call XSLT from XQuery, but **not XQuery from
XSLT** — which is exactly why embedding doesn't work in that direction.

## Wiring both engines together

When you genuinely need both languages, combine them *around* each other rather
than inside:

1. **At the API level (s9api).** Because both produce and consume the same XDM,
   you run them in one program and pass nodes between them — compile an
   `XQueryExecutable` and an `XsltExecutable` from the same Saxon `Processor`,
   run the query, and feed its `XdmNode` result straight into the transform (or
   the reverse). No serialization round-trip; the data model is identical. See
   the [Java API page](../realworld/api-java.md) for the s9api shape.
2. **An XProc pipeline.** XProc's `p:xslt` and `p:xquery` steps let you
   orchestrate "transform, then query, then transform" declaratively, as a
   pipeline outside both languages.

## Bottom line

Inside a single `.xsl` file: no XQuery. But you get its *expression* half for
free through shared XPath, its *clause* half through XSLT instructions, and — if
you must run both engines — you connect them through the Saxon API or
`fn:transform()`, never by nesting one in the other.

## Where to go next

- [XQuery vs XSLT](xquery-vs-xslt.md) — choosing which language owns a problem.
- [FLWOR expressions](flwor.md) — the query clauses that have no XPath form.
- [APIs: Java (JAXP & Saxon)](../realworld/api-java.md) — running both from s9api.
