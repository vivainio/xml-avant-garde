---
icon: lucide/triangle-alert
---

# Error handling

In 1.0 a runtime error — a bad cast, a missing document, a failed
[`xsl:result-document`](result-documents.md) — simply aborted the transform.
XSLT 3.0 adds **`xsl:try`/`xsl:catch`** so you can recover, and **`xsl:assert`**
so you can fail *deliberately* with a useful message.

## `xsl:try` / `xsl:catch`

Wrap the risky work in `xsl:try`; if it raises a dynamic error, control jumps to
a matching `xsl:catch` instead of aborting:

``` xml
<xsl:try>
  <xsl:variable name="d" select="doc('prices.xml')"/>
  <total><xsl:value-of select="sum($d/prices/price)"/></total>
  <xsl:catch>
    <total error="prices unavailable">0</total>
  </xsl:catch>
</xsl:try>
```

Inside the `xsl:catch`, a set of `err:` variables describe what went wrong:

| Variable | Holds |
| --- | --- |
| `$err:code` | the error's QName, e.g. `err:FODC0002` |
| `$err:description` | the human-readable message |
| `$err:value` | any value the error carried |
| `$err:module` / `$err:line-number` | where it was raised |

``` xml
<xsl:catch>
  <error code="{$err:code}"><xsl:value-of select="$err:description"/></error>
</xsl:catch>
```

You can narrow a catch to specific codes with `errors=` (a list of QName
patterns), and have several `xsl:catch` blocks — the first match wins:

``` xml
<xsl:try>
  <xsl:sequence select="$x cast as xs:integer"/>
  <xsl:catch errors="err:FORG0001">…not a valid integer…</xsl:catch>
</xsl:try>
```

!!! note "`rollback-output`"
    If the `xsl:try` body had already written some result before failing,
    `rollback-output="yes"` discards that partial output so the `xsl:catch`
    starts clean. Without it, whatever was written before the error stays.
    Streaming bodies cannot be rolled back, so the attribute must be `no` there.

## Raising your own: `error()`

The `error()` function aborts with a code and message of your choosing — useful
for enforcing preconditions:

``` xml
<xsl:if test="not(catalog/cd)">
  <xsl:sequence select="error(xs:QName('local:empty'), 'Catalogue has no entries')"/>
</xsl:if>
```

It pairs naturally with `xsl:try` elsewhere: raise a precise error here, catch
and translate it into output there.

## `xsl:assert` — checked assumptions

`xsl:assert` (3.0) states an invariant. When **assertions are enabled** (a
processor option — off by default, so production runs pay nothing), a false
`test` aborts with your message; when disabled, the instruction is skipped
entirely:

``` xml
<xsl:assert test="every $p in //price satisfies $p castable as xs:decimal"
            select="'Every price must be numeric'"/>
```

Use it for "this should never happen" checks that document your assumptions and
catch bad input early during development, without slowing the shipped transform.

## Where to go next

- [Sequences and types](sequences-and-types.md) — `cast as` / `treat as`, the operations that most often raise the errors you catch here.
- [Multiple result documents](result-documents.md) — where `rollback-output` matters.
- [Streaming](streaming.md) — error handling under single-pass constraints.
