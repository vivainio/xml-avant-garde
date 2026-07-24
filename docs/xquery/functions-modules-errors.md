---
icon: lucide/workflow
---

# Functions, modules, and error handling

A query grows into a program when repeated expressions become named functions,
shared code moves into library modules, and failures become part of the design.
XQuery provides all three without leaving the language: typed functions,
first-class function items, an import system, external parameters, and
`try`/`catch`.

## Declare a typed function

A function declaration belongs in the query prolog, before the main expression:

``` xquery
declare function local:price($cd as element(cd)) as xs:decimal {
  xs:decimal($cd/price)
};

for $cd in /catalog/cd
where local:price($cd) gt 10
return $cd/title
```

The parameter and return types are executable contracts. This declaration
requires exactly one `cd` element and promises exactly one decimal. Passing an
empty sequence, two elements, or returning a string raises a type error.

Cardinality belongs in the signature:

``` xquery
declare function local:total($cds as element(cd)*) as xs:decimal {
  sum($cds/price ! xs:decimal(.))
};

declare function local:year($cd as element(cd)) as xs:integer? {
  $cd/year ! xs:integer(.)
};
```

The first accepts any number of CDs; the second returns zero or one year.

!!! tip "Type at the boundary, infer inside"
    Strong parameter and return types catch bad calls early. Local `let`
    variables rarely need declarations because the processor can infer them.
    Add a local type when it expresses a real invariant, not merely to repeat
    what the expression already proves.

## Names and arity identify a function

Function identity is its expanded QName plus **arity**—the number of parameters.
The same name may have a one-argument and a two-argument form:

``` xquery
declare function local:label($cd as element(cd)) as xs:string {
  local:label($cd, " — ")
};

declare function local:label(
  $cd as element(cd),
  $separator as xs:string
) as xs:string {
  concat($cd/title, $separator, $cd/artist)
};
```

`local:label#1` and `local:label#2` are different function items. XQuery does not
overload solely by parameter type: two declarations with the same name and arity
conflict even if their types differ.

## Recursion replaces mutable state

Functions may call themselves:

``` xquery
declare function local:descendant-count($node as node()) as xs:integer {
  count($node/*) +
  sum($node/* ! local:descendant-count(.))
};

local:descendant-count(/catalog)
```

The more useful XML pattern is recursive construction:

``` xquery
declare function local:outline($element as element()) as element(item) {
  <item name="{ name($element) }">{
    $element/* ! local:outline(.)
  }</item>
};

local:outline(/catalog)
```

Prefer paths, FLWOR, and folds for ordinary iteration. Reach for recursion when
the data or algorithm is itself recursive: arbitrary-depth trees, graph walks,
or divide-and-conquer logic.

## Functions are values

An inline function can be stored, passed, and returned:

``` xquery
let $display := function($cd as element(cd)) as xs:string {
  concat($cd/title, " — ", $cd/artist)
}
return /catalog/cd ! $display(.)
```

A named function reference uses `#arity`:

``` xquery
let $prices := /catalog/cd/price ! xs:decimal(.)
return fold-left($prices, 0, function($total, $price) {
  $total + $price
})
```

Common higher-order functions include:

| Function | Does |
| --- | --- |
| `filter($items, $predicate)` | keeps items for which a function returns true |
| `for-each($items, $action)` | maps every item to a result |
| `fold-left($items, $zero, $combine)` | accumulates from left to right |
| `fold-right(...)` | accumulates from right to left |
| `sort($items, ..., $key)` | sorts using a key function |
| `map:for-each($map, $action)` | visits key/value pairs |

For example, parameterize a report by its grouping key:

``` xquery
declare function local:groups(
  $cds as element(cd)*,
  $key as function(element(cd)) as xs:anyAtomicType
) as element(group)* {
  for $cd in $cds
  group by $value := $key($cd)
  order by $value
  return <group key="{ $value }" count="{ count($cd) }"/>
};

local:groups(/catalog/cd, function($cd) { string($cd/@genre) })
```

An inline function is a **closure**: it retains variables from the scope where it
was created:

``` xquery
let $minimum := 10
let $expensive := function($cd) {
  xs:decimal($cd/price) ge $minimum
}
return filter(/catalog/cd, $expensive)/title
```

## Dynamic lookup

When the function name is data, `function-lookup` resolves a QName and arity:

``` xquery
let $name := QName("http://www.w3.org/2005/xpath-functions", "upper-case")
let $function := function-lookup($name, 1)
return
  if (exists($function))
  then $function("XQuery")
  else error(xs:QName("local:UNKNOWN-FUNCTION"))
```

Use dynamic lookup for plugin-like dispatch or configuration-driven pipelines.
For ordinary calls, a direct function reference is clearer and gives the compiler
more opportunities to check and optimize the query.

## Library modules

A library module groups declarations under its own namespace. It has no main
query expression:

``` xquery title="catalog-lib.xqm"
xquery version "3.1";

module namespace catalog = "https://example.org/catalog";

declare function catalog:price($cd as element(cd)) as xs:decimal {
  xs:decimal($cd/price)
};

declare function catalog:label($cd as element(cd)) as xs:string {
  concat($cd/title, " — ", $cd/artist)
};
```

Import it from a main module:

``` xquery title="report.xq"
xquery version "3.1";

import module namespace catalog = "https://example.org/catalog"
  at "catalog-lib.xqm";

for $cd in /catalog/cd
where catalog:price($cd) gt 10
return catalog:label($cd)
```

The namespace URI is the module's stable identity. The `at` URI is a location
hint; database engines may resolve a module namespace through their own
repository instead. Keep reusable functions in library modules and environment
or request-specific wiring in the main module.

Avoid circular imports. Even where a processor diagnoses them cleanly, mutually
dependent modules are a sign that shared types or helpers belong in a lower-level
module.

## Private functions and annotations

XQuery defines the annotation syntax `%prefix:name`; the meaning of many
annotations is processor-specific. The standard `%private` and `%public`
annotations control whether a module declaration is visible to importers:

``` xquery
declare %private function catalog:normalized-title(
  $cd as element(cd)
) as xs:string {
  normalize-space($cd/title)
};

declare %public function catalog:label($cd as element(cd)) as xs:string {
  concat(catalog:normalized-title($cd), " — ", $cd/artist)
};
```

RESTXQ route annotations and BaseX optimization annotations use the same syntax,
but are extensions. Treat them as an integration layer around portable core
functions.

## External variables: parameters without string substitution

Declare values supplied by the host application or command line as `external`:

``` xquery
declare variable $minimum as xs:decimal external;
declare variable $genre as xs:string? external := ();

for $cd in /catalog/cd
where xs:decimal($cd/price) ge $minimum
  and (empty($genre) or $cd/@genre = $genre)
return $cd
```

The optional default after `external :=` is used when the host supplies no value.
Binding variables keeps data separate from query source, preserves types, and
avoids the injection risk of building XQuery with string concatenation.

Library modules may expose external configuration too, but a small configuration
map passed into functions is often easier to test than a large set of global
variables.

## Raise a deliberate error

`error()` turns a failed invariant into a named dynamic error:

``` xquery
declare namespace app = "https://example.org/errors";

declare function local:required-artist(
  $cd as element(cd),
  $artists as element(artists)
) as element(artist) {
  let $matches := $artists/artist[@id = $cd/@artist-id]
  return
    if (count($matches) eq 1)
    then $matches
    else error(
      QName("https://example.org/errors", "app:ARTIST-CARDINALITY"),
      concat("Expected one artist for ", $cd/title),
      map { "artist-id": string($cd/@artist-id), "matches": count($matches) }
    )
};
```

The three arguments are an error QName, a human description, and an optional
application value. A namespace-qualified code prevents collisions with W3C and
processor error codes.

## Catch dynamic errors

Wrap the expression that may fail:

``` xquery
declare namespace app = "https://example.org/errors";

try {
  local:required-artist($cd, $artists)
} catch app:ARTIST-CARDINALITY {
  <warning code="{ $err:code }"
           message="{ $err:description }"
           artist-id="{ $err:value?artist-id }"/>
} catch err:FORG0001 {
  <warning message="A value could not be cast"/>
} catch * {
  <warning code="{ $err:code }" message="{ $err:description }"/>
}
```

Catch clauses are tried in order. A QName catches one code; `*` catches any
remaining dynamic error. The predefined `err:` variables can include:

- `$err:code` and `$err:description`;
- `$err:value`, the value passed to `error`;
- `$err:module`, `$err:line-number`, and `$err:column-number`;
- `$err:additional`, for processor-supplied detail.

Not every processor can provide every location field.

!!! warning "`try` cannot catch a query that does not compile"
    Static syntax and type errors are raised before evaluation reaches
    `try`/`catch`. Catch runtime failures—bad input values, unavailable documents,
    deliberate `error()` calls—not misspelled function names or malformed query
    source.

## Recover, enrich, or fail?

Use error handling deliberately:

- **Recover** when a fallback is a valid result, such as substituting cached
  metadata when an optional external document is unavailable.
- **Enrich and rethrow** when the lower-level error lacks business context.
- **Fail immediately** when continuing would produce a believable but incorrect
  report.

To rethrow with context:

``` xquery
try {
  doc($uri)
} catch * {
  error(
    QName("https://example.org/errors", "app:INPUT"),
    concat("Could not load catalog from ", $uri, ": ", $err:description),
    $err:value
  )
}
```

Returning `<error>` elements from every catch is not automatically safer. It
changes failure into ordinary data, which callers may overlook. Reserve that
pattern for APIs whose response vocabulary explicitly models errors.

## Where to go next

[Updating and indexing XML databases](updating-indexing.md) applies functions and
error behavior to persistent data, then shows how database indexes change query
performance.
