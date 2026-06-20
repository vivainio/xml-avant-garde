---
icon: lucide/git-compare
---

# XQuery vs XSLT

XQuery and [XSLT](../xslt/index.md) overlap almost completely in *capability* —
they share [XPath](../xpath/index.md) 3.1, the XDM data model, and the entire
`fn:` function library, and either one can, in principle, do the other's job.
They differ in **shape**: how you express the transformation, and which problems
that shape makes easy.

## Two control models

The fundamental split is **push vs pull**.

=== "XSLT — push, rule-based"

    You write **template rules** and let the processor walk the tree, firing
    whichever rule *matches* each node. You describe how to handle each *kind*
    of node; the engine drives the traversal.

    ``` xml
    <xsl:template match="cd">
      <album><xsl:value-of select="title"/></album>
    </xsl:template>
    ```

=== "XQuery — pull, query-based"

    You write an expression that **asks for** the nodes you want and builds the
    result around them. You drive the traversal explicitly with FLWOR.

    ``` xquery
    for $cd in /catalog/cd
    return <album>{ string($cd/title) }</album>
    ```

The push model shines when the input is **irregular or recursive** — documents
where the same element can appear anywhere and must be handled the same way
regardless of depth. The pull model shines when you know the **shape you want**
and are selecting, joining, and aggregating toward it.

## What each is best at

| Task | Reach for |
| --- | --- |
| Document-to-document transform, irregular/recursive structure | **XSLT** — `apply-templates`, [modes](../xslt/modes.md), the [identity transform](../xslt/modern-identity.md) |
| Mixed content (prose with inline tags) → HTML | **XSLT** — recursive rule matching is built for it |
| Select / filter / join / aggregate across a **collection** | **XQuery** — FLWOR + `collection()` over an indexed database |
| Query stored in and run by a database | **XQuery** — it *is* the database query language |
| Reporting: group, count, sum, top-N | **XQuery** — `group by`, `order by`, `count` read like SQL |
| Pipeline of named, composable steps | **XSLT** — modes and `xsl:import` precedence |

## The identity transform tells the story

The clearest dividing line is "copy the document, changing one thing." In XSLT
this is idiomatic — the [identity transform](../xslt/modern-identity.md) plus one
override:

``` xml
<xsl:mode on-no-match="shallow-copy"/>
<xsl:template match="price">
  <price><xsl:value-of select=". * 1.1"/></price>
</xsl:template>
```

XQuery has no template-matching equivalent built in; you write a recursive
**typeswitch** function that rebuilds the tree node by node. It works, but it is
visibly more code. Conversely, "give me the three priciest albums grouped by
genre" is a one-screen FLWOR in XQuery and a fiddlier exercise in XSLT.

!!! tip "Rule of thumb"
    *Reshaping a whole document?* XSLT. *Asking a question of a corpus?* XQuery.
    When the work is genuinely both, note that **Saxon and several engines run
    both languages** in the same pipeline — transform with XSLT, query with
    XQuery, no impedance mismatch because the data model is identical. See
    [Combining XSLT and XQuery](combining-xslt-xquery.md) for how.

## They are closer than they look

Because the foundation is shared, skills transfer directly:

- Every [XPath](../xpath/index.md) expression you write is valid in both.
- `let … return`, inline functions, maps, arrays, and JSON support exist in
  both XSLT 3.0 and XQuery 3.1 with the same semantics — XSLT in fact
  [borrowed `let`](../xslt/moving-to-3.md) from XQuery.
- A `declare function` in XQuery and an [`xsl:function`](../xslt/functions.md) in
  XSLT compile to the same kind of callable.

Learning one makes the other mostly a matter of syntax and control model — not a
new language.

## Where to go next

- [Combining XSLT and XQuery](combining-xslt-xquery.md) — using both on one engine.
- [FLWOR expressions](flwor.md) — the query syntax in depth.
- [XQuery in the real world](real-world.md) — where the pull model pays off.
