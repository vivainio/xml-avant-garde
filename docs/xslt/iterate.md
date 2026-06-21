---
icon: lucide/repeat
---

# Iteration with `xsl:iterate`

`xsl:for-each` visits every item, but each visit is **independent** — one
iteration cannot see what the previous one computed. Whenever you need to thread
a value *from one item to the next* — a running total, a previous row, a
counter, a "have I seen this yet?" flag — 1.0 forced you into recursive named
templates. 3.0 gives you `xsl:iterate`: an ordinary, readable loop that carries
state forward.

!!! note "This is a Saxon-HE feature — no licence needed"
    `xsl:iterate` is plain XSLT 3.0 and runs in the **free** Saxon-HE. It is
    often introduced alongside [streaming](streaming.md) because it is *also* the
    one loop that streams, but it is not streaming-only — reach for it on
    ordinary in-memory trees whenever a loop needs memory of its past.

## The shape

`xsl:iterate` selects a sequence, declares its carried state as `xsl:param`
children, and ends each iteration with `xsl:next-iteration` to supply the next
values:

``` xml linenums="1"
<xsl:iterate select="catalog/cd">              <!-- (1)! -->
  <xsl:param name="running" as="xs:decimal" select="0"/>   <!-- (2)! -->

  <xsl:variable name="now" select="$running + xs:decimal(price)"/>  <!-- (3)! -->
  <cd title="{title}" running-total="{$now}"/>             <!-- (4)! -->

  <xsl:next-iteration>                          <!-- (5)! -->
    <xsl:with-param name="running" select="$now"/>
  </xsl:next-iteration>
</xsl:iterate>
```

1.  The sequence to walk — same `select` you would give `xsl:for-each`.
2.  Carried state, with its starting value. The `xsl:param` children **must**
    come first, before any other instruction.
3.  Per-item work, reading the *incoming* `$running`.
4.  Output for this item — here, the cumulative total *including* this CD.
5.  Hand the updated value to the next turn. Omit a `with-param` and that
    parameter keeps its current value into the next iteration.

<div class="xslt-result" markdown>
``` xml
<cd title="Empire Burlesque" running-total="10.90"/>
<cd title="Hide your heart"  running-total="20.80"/>
<cd title="Greatest Hits"    running-total="30.70"/>
```
</div>

The running total is exactly the state that `xsl:for-each` cannot keep:
inside a `for-each`, every `cd` would see `$running` still at its initial `0`.

!!! warning "`xsl:param` first, and `select` must be a single sequence"
    The carried parameters must be the **first** children of `xsl:iterate`.
    `xsl:next-iteration` may appear only as the **last** thing executed on a
    path, and only inside the iteration (not in a called template). Every
    parameter you do not re-supply simply carries its present value forward.

## Stopping early with `xsl:break`

A `for-each` always runs to the end. `xsl:iterate` can quit the moment a
condition is met — and can emit a final result as it leaves:

``` xml linenums="1"
<!-- Add prices until the basket reaches 20, then stop. -->
<xsl:iterate select="catalog/cd">
  <xsl:param name="spent" as="xs:decimal" select="0"/>

  <xsl:variable name="now" select="$spent + xs:decimal(price)"/>
  <xsl:choose>
    <xsl:when test="$now gt 20">
      <xsl:break select="'budget reached'"/>          <!-- (1)! -->
    </xsl:when>
    <xsl:otherwise>
      <bought title="{title}"/>
      <xsl:next-iteration>
        <xsl:with-param name="spent" select="$now"/>
      </xsl:next-iteration>
    </xsl:otherwise>
  </xsl:choose>
</xsl:iterate>
```

1.  `xsl:break` ends the loop immediately. An optional `select` (or sequence
    constructor body) contributes a final value to the result, just like any
    other instruction.

Because it can short-circuit, `xsl:iterate` is the natural fit for "find the
first item where…" and "take items until…" — work that `for-each` can only fake
by visiting everything and filtering afterwards.

## Running once at the end with `xsl:on-completion`

When the loop finishes *normally* (not via `xsl:break`), an optional
`xsl:on-completion` block runs once, with the final carried state in scope —
ideal for a summary footer after the per-item rows:

``` xml linenums="1"
<xsl:iterate select="catalog/cd">
  <xsl:param name="total" as="xs:decimal" select="0"/>

  <xsl:on-completion>                                   <!-- (1)! -->
    <summary count="{count(catalog/cd)}" total="{$total}"/>
  </xsl:on-completion>

  <row title="{title}" price="{price}"/>
  <xsl:next-iteration>
    <xsl:with-param name="total" select="$total + xs:decimal(price)"/>
  </xsl:next-iteration>
</xsl:iterate>
```

1.  `xsl:on-completion` must be the first child *after* the `xsl:param`s. It
    fires only when the sequence is exhausted — `xsl:break` skips it.

## Why not just recurse, fold, or accumulate?

These all thread state. `xsl:iterate` is usually the clearest, but each has its
place:

| Tool | Threads state | Reads as | Best when |
| --- | --- | --- | --- |
| `xsl:for-each` | **no** | a loop | items are independent |
| `xsl:iterate` | yes | a loop | state flows item→item; you may stop early; you emit per-item output |
| Recursive named template | yes | recursion | the structure itself is recursive (a tree, not a flat list) |
| [`fold-left`](higher-order-functions.md) | yes | one expression | you want a *single* reduced value, no per-item nodes |
| [Accumulator](streaming.md) | yes | a declaration | the running value is needed in *many* templates, or you are streaming |

A `fold-left` is the tightest way to get one number out of a sequence; reach for
`xsl:iterate` when each step also **produces output** or you need to **break**.

!!! tip "`xsl:iterate` vs `fold-left` — same total, different shape"
    The cumulative-total example above could be a `fold-left` if you only wanted
    the final `30.70`. But here every CD emits a `<cd running-total="…">`
    element *as the loop runs* — that interleaving of output with threaded state
    is exactly what `xsl:iterate` expresses and `fold-left` does not.

## Where to go next

- [Higher-order functions](higher-order-functions.md) — `fold-left`/`fold-right`, the expression-level way to thread state.
- [Streaming](streaming.md) — where `xsl:iterate` and accumulators become mandatory, not optional.
- [Grouping](grouping.md) — `xsl:for-each-group`, the other structured replacement for hand-rolled loops.
