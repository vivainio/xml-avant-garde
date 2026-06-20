# External documents

So far every example has read a single source document. But a stylesheet often
needs to pull in *another* file — a shared code list, a configuration, a second
data set to join against. The `document()` function does exactly that: it loads
an XML file and hands you back its nodes, which you then navigate with XPath
just like the main input.

## Loading another file

`document('uri')` parses the file at `uri` and returns its node(s) — typically
the document root. From there, ordinary location paths reach inside it:

``` xml
<xsl:value-of select="document('catalog.xml')/catalog/cd[1]/title"/>   <!-- (1)! -->
```

1.  Load `catalog.xml`, then walk into it: the title of its first CD.

A **relative** URI is resolved against the location of the *stylesheet*, not the
source document and not the current working directory. So `document('codes.xml')`
finds a `codes.xml` sitting beside the `.xsl` file.

!!! note "The argument is an XPath expression"
    `document(...)` takes an *expression*, not just a literal. `document('codes.xml')`
    uses a string literal, but `document(@href)` would load whatever file the
    current node's `href` attribute names — handy for following references.

## The lookup-table join

The classic use is a **code-list join**: the main document stores terse codes,
and a small external file maps each code to a human-readable label. Suppose the
catalog tags each CD with a `genre` attribute holding a code:

``` xml title="catalog.xml" linenums="1"
<catalog>
  <cd genre="ROK"><title>Empire Burlesque</title><artist>Bob Dylan</artist><price>10.90</price></cd>
  <cd genre="POP"><title>Hide your heart</title><artist>Bonnie Tyler</artist><price>9.90</price></cd>
  <cd genre="CNT"><title>Greatest Hits</title><artist>Dolly Parton</artist><price>9.90</price></cd>
</catalog>
```

The labels live in their own file:

``` xml title="codes.xml" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<codes>
  <code id="ROK" label="Rock"/>
  <code id="POP" label="Pop"/>
  <code id="CNT" label="Country"/>
</codes>
```

Bind the loaded document to a [variable](variables.md) once, then look codes up
in it with an [XPath predicate](predicates.md):

``` xml title="genre-labels.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
<xsl:template match="/">
  <xsl:variable name="codes" select="document('codes.xml')"/>   <!-- (1)! -->
  <html><body>
    <xsl:for-each select="catalog/cd">
      <p>
        <xsl:value-of select="title"/> —
        <xsl:value-of select="$codes/codes/code[@id = current()/@genre]/@label"/>  <!-- (2)! -->
      </p>
    </xsl:for-each>
  </body></html>
</xsl:template>
</xsl:stylesheet>
```

1.  Load and parse `codes.xml` once; `$codes` now holds its node-set.
2.  Inside the loop, `current()` is the `cd` being processed. Find the `code`
    whose `id` matches this CD's `genre`, and read its `label`.

<div class="xslt-result" markdown>
Empire Burlesque — Rock

Hide your heart — Pop

Greatest Hits — Country
</div>

!!! tip "Why `current()` here?"
    Inside the predicate `[@id = ...]`, the context node is each `code` element,
    so a bare `@genre` would (wrongly) mean the *code's* `genre`. `current()`
    steps back out to the `cd` the `for-each` is iterating, so `current()/@genre`
    is the CD's code. Outside a predicate the two coincide; inside one they do
    not.

## `document('')` — the stylesheet itself

A string argument of the empty string is special: `document('')` refers to **the
stylesheet document itself**. This is the standard XSLT 1.0 idiom for keeping a
small lookup table *inside* the stylesheet — declared under your own namespace
so the processor ignores it as instructions — and then reading it back as data.

``` xml title="embedded-codes.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0"
                xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                xmlns:c="http://example.org/codes">       <!-- (1)! -->

<c:codes>                                                  <!-- (2)! -->
  <c:code id="ROK" label="Rock"/>
  <c:code id="POP" label="Pop"/>
  <c:code id="CNT" label="Country"/>
</c:codes>

<xsl:template match="/">
  <xsl:variable name="codes" select="document('')/xsl:stylesheet/c:codes"/>  <!-- (3)! -->
  <html><body>
    <xsl:for-each select="catalog/cd">
      <p>
        <xsl:value-of select="title"/> —
        <xsl:value-of select="$codes/c:code[@id = current()/@genre]/@label"/>
      </p>
    </xsl:for-each>
  </body></html>
</xsl:template>

</xsl:stylesheet>
```

1.  A custom namespace for the embedded data. `http://example.org/...` is a safe,
    made-up URI that will never clash with XSLT's own.
2.  The lookup table, sitting as a top-level child of `xsl:stylesheet`. Because
    it is in the `c:` namespace, the processor treats it as foreign data, not as
    an instruction to execute.
3.  `document('')` reparses *this* stylesheet; the path then dives into its
    `xsl:stylesheet` root and selects the embedded `c:codes` element.

<div class="xslt-result" markdown>
Empire Burlesque — Rock

Hide your heart — Pop

Greatest Hits — Country
</div>

Same result as the external file, but with the table and the logic shipped
together in one file — convenient for short, stable code lists.

!!! warning "Top-level foreign elements need a namespace"
    A top-level element in an XSLT 1.0 stylesheet that has **no** namespace is an
    error. The custom namespace (`c:` here) is what makes the embedded table
    legal — and it is also what your XPath must match (`c:code`, not `code`).

## Repeated calls and caching

Calling `document()` with the same URI more than once returns nodes from the
*same* parsed document — node identity is preserved, so comparisons behave
sensibly. Processors typically parse each distinct URI **once** and cache the
result, so binding the document to a variable up front (as above) is mostly a
matter of readability rather than performance. Even so, doing the load once and
naming it keeps the lookups tidy.

## Where next

That rounds out the core of **XSLT 1.0**: templates and `apply-templates`,
named templates and parameters, variables, control flow, XPath with predicates,
string handling, output control, sorting — and now joining in external data
with `document()`. With these you can express the great majority of everyday
transformations.

That completes the XSLT 1.0 toolkit. One topic still worth knowing in 1.0 is
**`xsl:key` and `key()`** — for large cross-references an indexed lookup beats
scanning a node-set with a predicate on every hit, and it is the scalable
version of the `document()` join shown above.

Beyond that, the language itself moved on. The next section,
[Moving to XSLT 2.0 and 3.0](moving-to-3.md), picks up where 1.0 leaves off —
sequences and types, real functions, native grouping, and regular expressions
that retire most of the 1.0 workarounds you have just learned.
