# Reading and writing JSON

XSLT grew up transforming XML into XML or HTML, but the data you are handed today
is just as likely to be **JSON** — a REST payload, a config file, a log line.
XSLT 3.0 makes JSON a first-class citizen: you can parse it into native **maps**
and **arrays**, walk it with a compact lookup syntax, and serialise structured
data straight back out as JSON. None of this exists in 1.0 or 2.0 — everything on
this page needs `version="3.0"` and a 3.0 processor such as Saxon.

There are two distinct ways to handle JSON, and it is worth knowing both:

| Approach | You get | Reach for it when |
| --- | --- | --- |
| **Maps & arrays** (`parse-json`, `json-doc`) | Native XDM maps/arrays, navigated with `?` | You want to *consume* JSON as data |
| **XML representation** (`json-to-xml`, `xml-to-json`) | An XML tree of `<map>`/`<array>`/`<string>` elements | You want to reuse your XPath/template skills on JSON |

## Reading JSON into maps and arrays

`parse-json($string)` turns a JSON string into native values: a JSON object
becomes a **map**, a JSON array becomes an **array**, and the scalars become
`xs:string`, `xs:double`, `xs:boolean`, or the empty sequence for `null`.

``` xml title="parse.xsl" linenums="1"
<xsl:variable name="raw" as="xs:string">                    <!-- (1)! -->
  { "artist": "Bob Dylan", "price": 10.90, "inStock": true }
</xsl:variable>

<xsl:variable name="cd" select="parse-json($raw)"/>         <!-- (2)! -->

<xsl:value-of select="$cd?artist"/>                         <!-- (3)! -->
```

1.  A literal JSON string. In real life this would come from a parameter, a file,
    or a web response.
2.  `parse-json` returns a `map(xs:string, item()*)` here — keys are strings,
    values are whatever the JSON held.
3.  The `?` **lookup operator** reads a map entry by key: `$cd?artist` is "look up
    key `artist` in map `$cd`". Far terser than `map:get($cd, 'artist')`.

<div class="xslt-result" markdown>
Bob Dylan
</div>

!!! tip "`json-doc()` reads straight from a URI"
    When the JSON lives in a file or at a URL, skip the read-then-parse two-step:
    `json-doc('catalog.json')` fetches *and* parses in one call, returning the
    same maps and arrays as `parse-json`. It is the JSON counterpart of
    [`document()`](external-documents.md).

## The lookup operator `?`

`?` is the workhorse for navigating parsed JSON. It indexes maps by key and
arrays by position (arrays are **1-based**, like everything else in XPath), and it
chains:

``` xml title="lookup.xsl" linenums="1"
<xsl:variable name="data" select="parse-json('
  { &quot;catalog&quot;: [
      { &quot;title&quot;: &quot;Empire Burlesque&quot;, &quot;price&quot;: 10.90 },
      { &quot;title&quot;: &quot;Hide your heart&quot;,  &quot;price&quot;: 9.90 }
  ] }')"/>

<xsl:value-of select="$data?catalog?1?title"/>   <!-- (1)! -->
<xsl:value-of select="$data?catalog?*?title"/>   <!-- (2)! -->
<xsl:value-of select="sum($data?catalog?*?price)"/>   <!-- (3)! -->
```

1.  Chain three lookups: key `catalog` → array index `1` → key `title`. → `Empire Burlesque`.
2.  The **wildcard** `?*` selects *every* member of the array, so `?*?title` is
    "the title of each entry" — a sequence of two strings.
3.  Because `?*?price` is a real sequence, ordinary XPath functions like `sum`
    apply directly. → `20.8`.

!!! note "When the key is not a name"
    `$map?artist` only works when the key is a valid name token. For keys with
    spaces or computed keys, use the parenthesised form `$map?('unit price')` or
    fall back to `map:get($map, 'unit price')`.

## Iterating over parsed JSON

Arrays are not sequences, so you do not `for-each` over them directly — you turn
them into a sequence first. `?*` is the usual way; `array:members` and the arrow
operator also work. Each member is itself a map you can look into:

``` xml title="iterate.xsl" linenums="1"
<xsl:for-each select="$data?catalog?*">     <!-- (1)! -->
  <cd>
    <title>{?title}</title>                 <!-- (2)! -->
    <price>{?price}</price>
  </cd>
</xsl:for-each>
```

1.  `?*` expands the array into a sequence of member maps, one per `for-each`
    iteration.
2.  Inside the loop the context item is one map, so a bare `?title` looks up the
    current entry. The `{ }` here is a **text value template** (see
    [Modern identity and text](modern-identity.md)); enable it with
    `expand-text="yes"` on the stylesheet.

<div class="xslt-result" markdown>
```
<cd><title>Empire Burlesque</title><price>10.9</price></cd>
<cd><title>Hide your heart</title><price>9.9</price></cd>
```
</div>

## Building JSON: maps and arrays

To *emit* JSON you build the native structures and then serialise them. Construct
maps and arrays inline with the `map { }` and `array { }` constructors, or
element-by-element with `xsl:map` / `xsl:map-entry`:

``` xml title="build.xsl" linenums="1"
<xsl:variable name="out" select="map {                      <!-- (1)! -->
  'count'  : count(catalog/cd),
  'titles' : array { catalog/cd/title/string() },          <!-- (2)! -->
  'total'  : sum(catalog/cd/price)
}"/>
```

1.  A map literal: `key : value` pairs separated by commas. Keys here are strings;
    values are an integer, an array, and a decimal.
2.  `array { ... }` collects a sequence into a JSON array — each `title`'s string
    value becomes one element.

The `xsl:map` form is handy when entries are conditional or built in a loop:

``` xml title="build-map.xsl" linenums="1"
<xsl:variable name="entry" as="map(*)">
  <xsl:map>
    <xsl:map-entry key="'title'" select="title/string()"/>
    <xsl:map-entry key="'price'" select="xs:double(price)"/>
    <xsl:if test="price &lt; 10">
      <xsl:map-entry key="'onSale'" select="true()"/>      <!-- (1)! -->
    </xsl:if>
  </xsl:map>
</xsl:variable>
```

1.  The entry only exists when the condition holds — easy conditional fields,
    awkward to do with a literal.

## Serialising to a JSON string

Once you hold a map or array, `serialize()` with the right options produces the
JSON text:

``` xml title="serialize.xsl" linenums="1"
<xsl:value-of select="serialize($out,
    map { 'method': 'json', 'indent': true() })"/>          <!-- (1)! -->
```

1.  The second argument is a serialisation-parameters map. `'method': 'json'` is
    what makes it JSON rather than XML; `'indent'` pretty-prints it.

<div class="xslt-result" markdown>
```
{ "count": 3,
  "titles": [ "Empire Burlesque", "Hide your heart", "Greatest Hits" ],
  "total": 30.7 }
```
</div>

!!! tip "Or let the whole result be JSON"
    Instead of `serialize`, you can declare `<xsl:output method="json"/>` and make
    JSON the *result document* itself — then an `xsl:sequence` of your map at the
    top level is serialised to JSON automatically, exactly as `method="xml"`
    serialises a node tree.

## The XML-representation route

The second approach skips maps entirely. `json-to-xml($string)` converts JSON
into a regular **XML tree** in a standard W3C vocabulary — `<map>`, `<array>`,
`<string>`, `<number>`, `<boolean>`, `<null>`, each carrying a `key` attribute
when it sits inside an object. The point: now all your existing XPath and template
machinery works on JSON.

``` xml title="json-to-xml.xsl" linenums="1"
<xsl:variable name="tree" select="json-to-xml($raw)"/>      <!-- (1)! -->

<xsl:value-of select="$tree//*:string[@key='artist']"/>     <!-- (2)! -->
```

1.  `$tree` is an ordinary document node in the `xpath-functions` namespace.
2.  Plain XPath against it — find the `string` element whose `key` is `artist`.
    (`*:string` ignores the namespace prefix for brevity.)

The output of `json-to-xml` for `{ "artist": "Bob Dylan", "price": 10.90 }`:

<div class="xslt-result" markdown>
```
<map xmlns="http://www.w3.org/2005/xpath-functions">
  <string key="artist">Bob Dylan</string>
  <number key="price">10.9</number>
</map>
```
</div>

The inverse, `xml-to-json($tree)`, takes a tree in that same vocabulary and
returns a JSON string — useful when you would rather *build* the structure as
familiar XML elements and convert at the end:

``` xml title="xml-to-json.xsl" linenums="1"
<xsl:variable name="built">
  <map xmlns="http://www.w3.org/2005/xpath-functions">
    <xsl:for-each select="catalog/cd">
      <map>
        <string key="title">{title}</string>
        <number key="price">{price}</number>
      </map>
    </xsl:for-each>
  </map>
</xsl:variable>

<xsl:value-of select="xml-to-json($built)"/>
```

## Which route should I use?

- **Maps and arrays** are the natural choice when JSON is *data* you compute over
  — sum a field, look up a key, reshape a payload. The `?` syntax is compact and
  the values are typed.
- **The XML representation** wins when you would rather lean on template rules,
  `apply-templates`, and XPath you already know — or when you need to round-trip
  JSON through the same pipeline that handles your XML.

They interoperate freely: parse with `parse-json`, and if a subtree is easier as
XML, there is nothing stopping you from handling the rest with `json-to-xml`.

## A complete round trip

Reading the running [catalog](index.md#the-running-example) and emitting JSON, end
to end:

``` xml title="catalog-to-json.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="3.0"
                xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                xmlns:xs="http://www.w3.org/2001/XMLSchema">

  <xsl:output method="json" indent="yes"/>                  <!-- (1)! -->

  <xsl:template match="/">
    <xsl:sequence select="map {                             <!-- (2)! -->
      'catalog': array {
        catalog/cd ! map {                                  <!-- (3)! -->
          'title':  title/string(),
          'artist': artist/string(),
          'price':  xs:double(price)
        }
      }
    }"/>
  </xsl:template>

</xsl:stylesheet>
```

1.  The result document *is* JSON — no `serialize` call needed.
2.  Top-level `xsl:sequence` of a map; with `method="json"` the processor
    serialises it.
3.  The **simple map operator** `!` applies the right-hand `map { }` constructor
    to each `cd` in turn, producing one map per disc — the array members.

<div class="xslt-result" markdown>
```
{ "catalog": [
    { "title": "Empire Burlesque", "artist": "Bob Dylan", "price": 10.9 },
    { "title": "Hide your heart", "artist": "Bonnie Tyler", "price": 9.9 },
    { "title": "Greatest Hits", "artist": "Dolly Parton", "price": 9.9 }
  ] }
```
</div>

## Next

You have now seen XSLT from 1.0 fundamentals to its modern, JSON-aware 3.0 form.
To revisit any topic, head back to the [Overview](index.md).
