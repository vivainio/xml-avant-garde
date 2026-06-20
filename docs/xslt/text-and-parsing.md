---
icon: lucide/file-text
---

# Reading and writing non-XML text

A persistent myth is that XSLT only reads and writes XML. Since 2.0 it has been a
capable text-processing language too: it can pull in a **plain-text file**, a
**CSV**, or a JSON document, turn a *string* into a navigable tree, and write a
tree back out as a string or a text file. Combined with
[regular expressions](regex.md), this makes XSLT a real option for
format-conversion jobs that have nothing to do with XML at either end.

This page covers the four corners:

| Direction | Resource (a file/URI) | A string in memory |
| --- | --- | --- |
| **In** | `unparsed-text` / `unparsed-text-lines` | `parse-xml` / `parse-json` |
| **Out** | `xsl:result-document method="text"` | `serialize` |

## Reading text: `unparsed-text`

`unparsed-text($href)` returns the entire contents of a resource as a single
`xs:string` — no XML parsing, no markup interpretation. `unparsed-text-lines`
returns it pre-split into a sequence of lines, which is almost always what you
want:

``` xml
<xsl:variable name="raw"   select="unparsed-text('notes.txt')"/>
<xsl:variable name="lines" select="unparsed-text-lines('data.csv')"/>
<xsl:value-of select="count($lines)"/>   <!-- number of lines -->
```

The `href` is resolved against the stylesheet's base URI; an optional second
argument names the encoding (`unparsed-text('x.txt', 'ISO-8859-1')`) when it is
not UTF-8 and not declared by the resource.

!!! tip "Guard with `unparsed-text-available`"
    Reading a missing file is a [recoverable error](error-handling.md). Test
    first when the input is optional:

    ``` xml
    <xsl:if test="unparsed-text-available('notes.txt')">
      <xsl:value-of select="unparsed-text('notes.txt')"/>
    </xsl:if>
    ```

### Worked example: CSV → XML

The canonical text job. Read the lines, drop the header, [`tokenize`](regex.md)
each row on commas, and build elements — the catalog you have been transforming
all along, but arriving as CSV:

``` xml title="catalog.csv"
genre,title,artist,price
rock,Empire Burlesque,Bob Dylan,10.90
pop,Hide your heart,Bonnie Tyler,9.90
```

``` xml
<xsl:template name="xsl:initial-template">
  <catalog>
    <xsl:for-each select="tail(unparsed-text-lines('catalog.csv'))">   <!-- (1)! -->
      <xsl:variable name="f" select="tokenize(., ',')"/>               <!-- (2)! -->
      <cd genre="{$f[1]}">
        <title>{$f[2]}</title>
        <artist>{$f[3]}</artist>
        <price>{$f[4]}</price>
      </cd>
    </xsl:for-each>
  </catalog>
</xsl:template>
```

1. [`tail`](new-functions.md) drops the header line; `xsl:initial-template` is
   the 3.0 entry point when there is no source document to match.
2. `tokenize` splits each line into fields. Real CSV with quoted commas needs
   [`xsl:analyze-string`](regex.md) instead — but for clean data this is enough.

(This assumes [text value templates](modern-identity.md) are on with
`expand-text="yes"`, hence the bare `{$f[2]}`.)

## Parsing a string into a tree: `parse-xml`

When the XML arrives *as a string* — embedded in a database column, returned by
an API, or read with `unparsed-text` — `parse-xml($string)` (3.0) turns it into a
**document node** you can navigate normally:

``` xml
<xsl:variable name="doc" select="parse-xml('&lt;cd&gt;&lt;title&gt;X&lt;/title&gt;&lt;/cd&gt;')"/>
<xsl:value-of select="$doc/cd/title"/>   <!-- X -->
```

`parse-xml-fragment` is the variant for a *fragment* — content without a single
document element, such as `<a/><b/>` or a run of mixed text and elements.

For JSON the equivalent is [`parse-json`](json.md) (string → maps/arrays) and
`json-doc($href)`, which reads JSON straight from a file in one step — the JSON
sibling of `unparsed-text` + `parse-json`.

## Writing a tree as a string: `serialize`

The inverse of `parse-xml`. `serialize($node)` (3.0) renders a node — or any
sequence — back into markup *as a string*, so you can embed it in text output,
store it, or post-process it:

``` xml
<xsl:value-of select="serialize(/catalog/cd[1])"/>
<!-- &lt;cd genre="rock"&gt;&lt;title&gt;Empire Burlesque&lt;/title&gt;… -->
```

A second argument controls the output — either a map or an
`output:serialization-parameters` element:

``` xml
<xsl:value-of select="serialize($node, map { 'method': 'xml', 'indent': true() })"/>
```

This is how you produce *escaped* markup (XML-inside-text), or serialise to JSON
with `map { 'method': 'json' }`.

## Writing text files: `method="text"`

To emit a non-XML *file*, combine [`xsl:result-document`](result-documents.md)
with `method="text"` — every node is flattened to its string value, no tags, no
escaping of `<` and `&` beyond what text needs.

### Worked example: XML → CSV

The round trip back out. Note `xsl:text` and `&#10;` to place the commas and
newlines exactly:

``` xml
<xsl:template match="/catalog">
  <xsl:result-document href="out.csv" method="text">
    <xsl:text>genre,title,artist,price&#10;</xsl:text>
    <xsl:for-each select="cd">
      <xsl:value-of select="(@genre, title, artist, price)" separator=","/>
      <xsl:text>&#10;</xsl:text>
    </xsl:for-each>
  </xsl:result-document>
</xsl:template>
```

`xsl:value-of` with `separator=","` joins the four fields in one step — the
[`string-join`](new-functions.md) idiom built into the instruction.

!!! warning "Character maps and escaping"
    `method="text"` does no markup escaping, so the output is literally the
    string values. If you need to substitute particular characters on the way
    out (say, replace a delimiter that appears in the data), an
    `xsl:character-map` referenced from `xsl:output` does it at serialization
    time. For structural CSV quoting, escape inside the transform with
    [`replace`](regex.md) before output.

## Where to go next

- [Regular expressions and strings](regex.md) — `tokenize`/`analyze-string`, the parsing half of every text job.
- [Reading and writing JSON](json.md) — `parse-json`, `json-doc`, and `serialize` with `method="json"`.
- [Multiple result documents](result-documents.md) — writing many text or XML files in one run.
- [New functions](new-functions.md) — `head`/`tail`, `string-join`, and the rest used above.
