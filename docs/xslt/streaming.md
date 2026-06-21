---
icon: lucide/waves
---

# Streaming

The headline feature of XSLT **3.0** is **streaming**: processing a document
that is far too large to hold in memory by reading it once, top to bottom, and
discarding each part after use. A 50 GB log or data export that would blow the
heap as a tree becomes tractable — at the cost of working under strict rules
about what your code may do.

!!! info "Saxon edition"
    Streaming is a Saxon-**EE** feature (and only some processors implement it at
    all). The non-streaming versions of everything below work everywhere; reach
    for streaming specifically when input size forces it.

## The core constraint: one forward pass

A tree lets you go anywhere — `following-sibling`, a second `xsl:for-each` over
the same nodes, `//x` from the root. A stream cannot: once a node has gone past,
it is gone. The processor therefore checks your code for **guaranteed
streamability** and rejects anything that would need to look backward or visit a
node twice. Two ideas describe what is allowed:

- A **motionless** construct touches no input (a literal, `position()`).
- A **consuming** construct reads input — but each part of the stream may be
  consumed **once**. Two expressions consuming the same nodes is the cardinal
  error.

In practice you process each top-level record as it arrives, extract what you
need from it in a single downward pass, emit output, and move on.

## Opening a stream

`xsl:source-document` with `streamable="yes"` reads a document as a stream; a
matching template fires per record. The classic shape — summarise a huge catalog
without ever holding it all:

``` xml
<xsl:template name="xsl:initial-template">
  <xsl:source-document href="huge-catalog.xml" streamable="yes">
    <report>
      <xsl:for-each select="catalog/cd">
        <!-- one downward pass over each cd; nothing kept afterwards -->
        <cd genre="{@genre}"><xsl:value-of select="title"/></cd>
      </xsl:for-each>
    </report>
  </xsl:source-document>
</xsl:template>
```

You can equally set `streamable="yes"` on an [`xsl:mode`](modes.md) and drive it
with `apply-templates`, as long as every matching template is itself streamable.

## Accumulators — carrying state forward

Because you cannot re-scan, 3.0 adds **accumulators**: a named value that the
processor updates *as* the stream flows past, which you read at any node with
`accumulator-before()` / `accumulator-after()`. This is how you compute running
totals or counts in a single pass:

``` xml
<xsl:accumulator name="total" as="xs:decimal" initial-value="0">
  <xsl:accumulator-rule match="price" select="$value + xs:decimal(.)"/>
</xsl:accumulator>

<!-- later, streamably: -->
<xsl:value-of select="accumulator-after('total')"/>
```

`$value` is the accumulator's running value; the rule fires each time a `price`
goes by, so the total is ready by the end of the document without a second pass.

## `xsl:iterate` — the streamable loop

`xsl:for-each` cannot thread state from one item to the next. `xsl:iterate` can:
it carries `xsl:param` values across iterations (`xsl:next-iteration`) and can
stop early (`xsl:break`), all within the single-pass rules. It is also the one
loop that is *allowed* in a stream, because it never looks backward. Its full
treatment — `xsl:on-completion`, `xsl:break`, and when to prefer it over
`fold-left` or recursion — is on the [iteration page](iterate.md); here is the
streaming-relevant shape:

``` xml
<xsl:iterate select="catalog/cd">
  <xsl:param name="running" select="0.0"/>
  <xsl:variable name="running" select="$running + xs:decimal(price)"/>
  <cd title="{title}" running-total="{$running}"/>
  <xsl:next-iteration>
    <xsl:with-param name="running" select="$running"/>
  </xsl:next-iteration>
</xsl:iterate>
```

## When the rules bite

The compiler will reject, for example, `cd[last()]` in a stream (needs to know
the count before it has seen them all) or referencing
`preceding-sibling::cd` (already discarded). The fixes are idiomatic: use an
**accumulator** instead of looking back, `xsl:iterate` instead of cross-item
state, and `copy-of` to ground a small subtree into memory when you genuinely
need random access to *that record* (just not the whole document).

!!! tip "Don't stream by default"
    Streaming trades flexibility for scale. For documents that fit in memory —
    the overwhelming majority — the ordinary tree model is simpler and often
    faster. Adopt streaming when a real size limit demands it, not pre-emptively.

## Where to go next

- [Grouping](grouping.md) — `xsl:for-each-group` has a streamable subset (`group-adjacent`).
- [Multiple result documents](result-documents.md) — splitting a massive input into many files as it streams.
- [Error handling](error-handling.md) — why `rollback-output` cannot apply to streamed output.
