---
icon: lucide/braces
---

# Maps, arrays, and JSON

XQuery 3.1 is not limited to XML trees. A query can read JSON into native
**maps** and **arrays**, combine it with XML, and return either format. These
values are part of the XDM, so they pass through variables and functions without
being converted to temporary XML.

The three containers serve different jobs:

| Value | Best for | Access |
| --- | --- | --- |
| sequence | a flat stream of items | position or iteration |
| map | values addressed by key | `$map?key` |
| array | members whose position and nesting matter | `$array(1)` |

## Maps: immutable key-value lookups

Construct a map with key/value pairs:

``` xquery
let $album := map {
  "title": "Empire Burlesque",
  "artist": "Bob Dylan",
  "price": xs:decimal("10.90")
}
return ($album?title, $album?price)
```

The lookup operator is shorthand for `map:get`:

``` xquery
$album?title
map:get($album, "title")
$album?("title")                  (: computed key :)
```

The bare `$album?title` form works for name-like string keys. Use parentheses or
`map:get` for a computed key or one containing spaces.

Maps are immutable. “Adding” an entry returns a new map:

``` xquery
let $discounted := map:put($album, "onSale", true())
return (
  map:contains($album, "onSale"),       (: false :)
  $discounted?onSale                    (: true :)
)
```

The original `$album` is unchanged. The same rule applies to `map:remove` and
`map:merge`.

## Build an index from XML

The joins chapter repeatedly looked up artists by `@id`. Turn the master data
into a map once:

``` xquery
let $artists := doc("artists.xml")/artists/artist
let $artist-by-id :=
  map:merge(
    for $artist in $artists
    return map:entry(string($artist/@id), $artist)
  )
for $cd in doc("catalog.xml")/catalog/cd
let $artist := $artist-by-id?(string($cd/@artist-id))
return <album title="{ $cd/title }" artist="{ $artist/name }"/>
```

Each value is still an XML `artist` element. A map does not force the value into
a string; map values may be any sequence.

Duplicate keys need a deliberate policy. By default `map:merge` keeps one value;
for a one-to-many index, combine them:

``` xquery
map:merge(
  for $cd in /catalog/cd
  return map:entry(string($cd/@genre), $cd),
  map { "duplicates": "combine" }
)
```

Now `$by-genre?rock` can be a sequence of several `cd` elements.

## Arrays preserve boundaries

Sequences flatten:

``` xquery
(("rock", "pop"), ("country"))     (: three strings in one flat sequence :)
```

An array keeps members and nesting:

``` xquery
let $genres := ["rock", "pop", "country"]
return (
  $genres(1),              (: rock; array positions are 1-based :)
  array:size($genres),     (: 3 :)
  $genres?*                (: all members as a sequence :)
)
```

There are two array constructors with an important difference:

``` xquery
[("rock", "pop"), "country"]       (: two members; first holds two strings :)
array { ("rock", "pop", "country") } (: three members, one per input item :)
```

The square constructor preserves each comma-separated expression as one member.
The curly constructor turns every item produced by its expression into a member.
That makes `array { for ... return ... }` ideal for building JSON arrays.

Like maps, arrays are immutable:

``` xquery
array:append($genres, "jazz")
array:put($genres, 2, "synth-pop")
array:remove($genres, 3)
```

Each function returns a new array.

## Parse and navigate JSON

`parse-json` reads a string; `json-doc` reads a URI:

``` xquery
let $data := parse-json('
  {
    "catalog": [
      {"title": "Empire Burlesque", "price": 10.90},
      {"title": "Hide your heart", "price": 9.90}
    ]
  }')
return (
  $data?catalog(1)?title,
  sum($data?catalog?*?price)
)
```

The JSON object becomes a map, `catalog` becomes an array, and each album object
becomes another map:

``` text
map
└── "catalog" → array
    ├── 1 → map { "title": ..., "price": ... }
    └── 2 → map { "title": ..., "price": ... }
```

`?*` expands all array members into a sequence, so ordinary FLWOR and aggregate
functions take over:

``` xquery
for $album in $data?catalog?*
where $album?price gt 10
order by $album?title
return $album?title
```

JSON scalar values map to XDM strings, doubles, and booleans. JSON `null` maps to
the empty sequence, so use `map:contains` when you must distinguish a present
property whose value is `null` from an absent property.

## Join JSON to XML

Because both formats become XDM values, no conversion layer is needed. Suppose a
JSON API provides current stock:

``` json title="stock.json"
{
  "dylan": 4,
  "tyler": 0,
  "parton": 7
}
```

Join it directly to the XML catalog:

``` xquery
let $stock := json-doc("stock.json")
for $cd in doc("catalog.xml")/catalog/cd
let $quantity := $stock?(string($cd/@artist-id))
return
  <album title="{ $cd/title }"
         in-stock="{ $quantity gt 0 }"
         quantity="{ $quantity }"/>
```

The map lookup is the join. The result happens to be XML, but it can just as
easily be JSON.

## Build a JSON result

Construct maps for objects and an array for the result list:

``` xquery
array {
  for $cd in doc("catalog.xml")/catalog/cd
  order by $cd/title
  return map {
    "title": string($cd/title),
    "genre": string($cd/@genre),
    "price": xs:decimal($cd/price)
  }
}
```

Tell the serializer that the top-level array is JSON:

``` xquery
declare namespace output =
  "http://www.w3.org/2010/xslt-xquery-serialization";
declare option output:method "json";
declare option output:indent "yes";

array {
  for $cd in doc("catalog.xml")/catalog/cd
  return map {
    "title": string($cd/title),
    "price": xs:decimal($cd/price)
  }
}
```

The `output` prefix is bound explicitly to the W3C serialization namespace so
the query does not depend on a processor-specific predeclared prefix.

The result is JSON text:

``` json
[
  {"title":"Empire Burlesque","price":10.9},
  {"title":"Hide your heart","price":9.9},
  {"title":"Greatest Hits","price":9.9}
]
```

Alternatively, `serialize($value, map { "method": "json" })` returns the JSON as
a string inside a larger query.

!!! warning "JSON cannot serialize arbitrary XDM values"
    JSON has no XML node, QName, date, or decimal type of its own. Convert nodes
    to strings and choose representations for dates and QNames before
    serialization. Processors serialize ordinary numeric XDM values as JSON
    numbers, but unsupported values raise an error instead of guessing.

## Transform JSON through XML

Sometimes templates and tree navigation are more convenient than maps.
`json-to-xml` produces the W3C XML representation of JSON:

``` xquery
let $tree := json-to-xml('{"artist":"Bob Dylan","price":10.90}')
return $tree//*:string[@key = "artist"]
```

`xml-to-json` performs the inverse, but only for XML that follows that standard
representation. For ordinary query work, native maps and arrays are usually
shorter; the XML representation is useful when an existing XML pipeline must
process the data.

## Types for maps and arrays

Function signatures can state the expected structure:

``` xquery
declare function local:total($album as map(*)) as xs:decimal {
  xs:decimal($album?price)
};

declare function local:titles($albums as array(*)) as xs:string* {
  $albums?*?title
};
```

`map(*)` and `array(*)` accept any map or array. A more specific map type such as
`map(xs:string, item()*)` constrains its key and value types. XQuery's type system
can describe the container shape, but it does not provide a built-in schema for
required JSON property names; validate those explicitly or with
implementation-specific JSON Schema support.

## Where to go next

[Constructing XML precisely](constructing-xml.md) returns to XML output and
explains direct versus computed constructors, node copying, namespaces, and
whitespace.
