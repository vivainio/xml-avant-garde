# Keys and indexed lookup

The [external documents](external-documents.md) page joined a catalog to a code
list with an XPath predicate: `code[@id = current()/@genre]`. That works, but it
**re-scans the whole code list on every lookup** — for three codes it is
invisible, for thousands it is quadratic. `xsl:key` builds an **index** once, so
each lookup with `key()` is effectively constant-time. It is the scalable version
of the same join, and it has been in the language since 1.0 — everything here
runs in any processor.

## Declaring a key

`xsl:key` is a top-level element (a direct child of `xsl:stylesheet`). It names an
index and says *what to index* and *by which value*:

``` xml
<xsl:key name="cd-by-genre" match="cd" use="@genre"/>   <!-- (1)! -->
```

1.  Build an index called `cd-by-genre` over every `cd` element (`match`), keyed
    on each one's `genre` attribute (`use`).

- **`name`** — what you will pass to `key()`.
- **`match`** — which nodes go into the index (the same pattern language as
  `xsl:template match`).
- **`use`** — an XPath, evaluated relative to each matched node, giving the value
  to file it under.

The processor walks the document once, building a map from key value → the nodes
that have it.

## Looking things up with `key()`

`key('name', value)` returns **all** nodes in that index whose key equals
`value` — a node-set, just like a predicate would yield, but found by lookup
rather than by scanning:

``` xml title="by-genre.xsl" linenums="1"
<xsl:key name="cd-by-genre" match="cd" use="@genre"/>

<xsl:template match="/">
  <xsl:for-each select="key('cd-by-genre', 'ROK')">   <!-- (1)! -->
    <p><xsl:value-of select="title"/></p>
  </xsl:for-each>
</xsl:template>
```

1.  Every `cd` whose `@genre` is `ROK`, fetched from the index. Equivalent to
    `//cd[@genre = 'ROK']`, but it does not re-scan the tree.

!!! note "A key can map many nodes to one value, and one node to many keys"
    `key()` returns a *node-set*, so a key value shared by several nodes returns
    all of them — that is exactly what makes it useful for grouping. Conversely,
    if `use` evaluates to multiple values (e.g. `use="tokenize(@tags, ' ')"` in
    2.0, or a node-set in 1.0), the node is indexed under *each* of them.

## The code-list join, re-done with a key

Here is the [external-documents](external-documents.md) join again, but indexed.
The code list lives in its own file:

``` xml title="codes.xml" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<codes>
  <code id="ROK" label="Rock"/>
  <code id="POP" label="Pop"/>
  <code id="CNT" label="Country"/>
</codes>
```

We index the `code` elements by their `id`, then look each catalog code up:

``` xml title="genre-labels-keyed.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:key name="code-by-id" match="code" use="@id"/>          <!-- (1)! -->

<xsl:variable name="codes" select="document('codes.xml')"/>

<xsl:template match="/">
  <html><body>
    <xsl:for-each select="catalog/cd">
      <xsl:variable name="g" select="@genre"/>              <!-- (2)! -->
      <xsl:for-each select="$codes">                        <!-- (3)! -->
        <p><xsl:value-of select="key('code-by-id', $g)/@label"/></p>
      </xsl:for-each>
    </xsl:for-each>
  </body></html>
</xsl:template>

</xsl:stylesheet>
```

1.  The key indexes `code` elements wherever they occur — including inside the
    loaded `codes.xml`.
2.  Capture the catalog code into a variable *before* we change context (next
    line), because once we move into `$codes` the bare `@genre` would be gone.
3.  The crucial step — see the warning below.

<div class="xslt-result" markdown>
Rock

Pop

Country
</div>

!!! warning "`key()` only searches the *current* document"
    `key()` looks in the document that owns the **context node** at the moment you
    call it. Straight inside `for-each select="catalog/cd"` the context document is
    the *source*, so `key('code-by-id', …)` would search the catalog — and find
    nothing. The inner `for-each select="$codes"` exists solely to **switch the
    context node into `codes.xml`**, so the key resolves against the right
    document. This is the standard 1.0 idiom for keyed lookups into a loaded file:
    capture the value, switch context, call `key()`.

## Keys for grouping (Muenchian)

Before `xsl:for-each-group` (2.0), keys were also the engine of **grouping**. The
"Muenchian" technique keys every item by its grouping value, then keeps only the
*first* item of each value — the ones that are identical to the first node the key
returns for their own key:

``` xml title="muenchian.xsl" linenums="1"
<xsl:key name="cd-by-genre" match="cd" use="@genre"/>

<xsl:template match="/">
  <xsl:for-each select="catalog/cd[
        generate-id() = generate-id(key('cd-by-genre', @genre)[1])]">  <!-- (1)! -->
    <h2><xsl:value-of select="@genre"/></h2>
    <xsl:for-each select="key('cd-by-genre', @genre)">                 <!-- (2)! -->
      <p><xsl:value-of select="title"/></p>
    </xsl:for-each>
  </xsl:for-each>
</xsl:template>
```

1.  Keep a `cd` only if it is the *first* node its key returns — one
    representative per distinct genre. `generate-id()` compares node identity.
2.  For each representative, pull the whole group back out of the index.

It works, but it is famously oblique. In 2.0/3.0 this entire pattern collapses to
`xsl:for-each-group` — see [Grouping](grouping.md), which opens by retiring
exactly this trick.

## How keys relate to maps

Keys and the 3.0 [`map`](json.md) type both give indexed, near-constant-time
lookup. They are not the same tool:

| | `xsl:key` / `key()` | `map` (3.0) |
| --- | --- | --- |
| Version | 1.0+ | 3.0 |
| Indexes | nodes in a document | any values |
| Returns | a node-set | anything — strings, sequences, nested maps |
| Lives | bound to a document; `key()` searches the *context* document | a free value — passed to functions, returned, nested |
| Build | declarative, automatic, one top-level element | you construct it (often from loaded nodes) |
| Best when | indexing into a *source tree* and you want nodes back | you want a portable side table, typed keys, or non-node values |

A rough rule: reach for **`key()`** when the data you are indexing *is* the XML
you are already processing and you want the matching **nodes**; reach for a
**`map`** when you want a standalone lookup structure you can pass around, key by
typed values, or fill with records rather than nodes. See the
[codelist-as-a-map](json.md#reading-json-into-maps-and-arrays) discussion for the
3.0 alternative to the join above.

## Next

That completes the XSLT 1.0 toolkit, indexed lookups included. The next section,
[Moving to XSLT 2.0 and 3.0](moving-to-3.md), picks up where 1.0 leaves off —
sequences and types, real functions, native grouping, and regular expressions
that retire most of the 1.0 workarounds.
