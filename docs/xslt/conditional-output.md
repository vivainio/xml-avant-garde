---
icon: lucide/filter
---

# Conditional output

A recurring annoyance in 1.0 is the **empty wrapper**: you build a `<discounts>`
element, the source happens to have no discounts, and you emit `<discounts/>`
anyway — or you wrap every constructor in an [`xsl:if`](conditionals.md) that
duplicates the selection logic. XSLT 3.0 adds three instructions that make output
*depend on whether other output was produced*, removing that duplication.

## `xsl:where-populated` — drop empty elements

`xsl:where-populated` evaluates the content inside it and **discards any element
it constructed that ended up with no attributes and no children**. Wrap a
maybe-empty element in it and the element simply disappears when there is nothing
to put inside:

``` xml
<xsl:where-populated>
  <expensive>
    <xsl:for-each select="cd[xs:decimal(price) > 10]">
      <cd>{title}</cd>
    </xsl:for-each>
  </expensive>
</xsl:where-populated>
```

If at least one CD is over 10, you get `<expensive><cd>…</cd></expensive>`. If
none are, the `<expensive>` wrapper is omitted entirely — no `<expensive/>`,
and no separate `xsl:if test="cd[...]"` repeating the predicate.

## `xsl:on-empty` / `xsl:on-non-empty` — react to the result

These two are evaluated *relative to the other items in the same sequence
constructor*. They let one constructor say "if the rest of this produced output,
add a header; if it produced nothing, say so instead":

``` xml
<table>
  <xsl:on-non-empty>
    <tr><th>Title</th><th>Price</th></tr>          <!-- header: only if rows exist -->
  </xsl:on-non-empty>

  <xsl:for-each select="cd[xs:decimal(price) > 10]">
    <tr><td>{title}</td><td>{price}</td></tr>
  </xsl:for-each>

  <xsl:on-empty>
    <tr><td>No expensive CDs.</td></tr>            <!-- fallback: only if no rows -->
  </xsl:on-empty>
</table>
```

The rule:

- **`xsl:on-non-empty`** contributes its content **only if** the *other* items in
  the constructor produced a non-empty result. Its own output does not count
  toward that test — so a table header never fools the check into thinking there
  are rows.
- **`xsl:on-empty`** contributes its content **only if** the other items produced
  *nothing*.

So the table above gets a header row *and* the data rows when there are matches,
and just the "No expensive CDs." row when there are none — the classic
report-with-fallback, without a guarding `xsl:choose`.

!!! tip "Which one to reach for"
    - Suppressing a **container** when its body is empty → `xsl:where-populated`.
    - Adding a **header/label** that should appear only alongside real content,
      or a **fallback** when there is none → `xsl:on-non-empty` /
      `xsl:on-empty`.

    They compose: a `xsl:where-populated` around a constructor that itself uses
    `xsl:on-non-empty` for a header is a common, fully declarative pattern.

## Where to go next

- [Conditionals](conditionals.md) — `xsl:if` and `xsl:choose`, when the test is on *input*, not output.
- [Modern identity and text](modern-identity.md) — text value templates, used in the examples above.
- [Grouping](grouping.md) — often the source of the maybe-empty groups these instructions tidy up.
