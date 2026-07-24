---
icon: lucide/git-merge
---

# Joining XML documents

XQuery can bind data from several documents in one FLWOR expression and connect
related nodes by value. There is no special `JOIN` keyword: bind the candidate
rows with `for` or `let`, match their keys with `where` or a predicate, and
construct the result.

This is where XQuery starts to feel like SQL while still returning trees rather
than flat rows.

## Two documents, one relationship

Suppose the catalog stores only an artist identifier:

``` xml title="catalog.xml"
<catalog>
  <cd artist-id="dylan" genre="rock">
    <title>Empire Burlesque</title><price>10.90</price>
  </cd>
  <cd artist-id="tyler" genre="pop">
    <title>Hide your heart</title><price>9.90</price>
  </cd>
  <cd artist-id="parton" genre="country">
    <title>Greatest Hits</title><price>9.90</price>
  </cd>
</catalog>
```

The names and countries live in master data:

``` xml title="artists.xml"
<artists>
  <artist id="dylan"><name>Bob Dylan</name><country>US</country></artist>
  <artist id="tyler"><name>Bonnie Tyler</name><country>GB</country></artist>
  <artist id="parton"><name>Dolly Parton</name><country>US</country></artist>
</artists>
```

`@artist-id` and `@id` form the relationship. Load each document once and bind
matching pairs:

``` xquery title="catalog-with-artists.xq"
xquery version "3.1";

let $catalog := doc("catalog.xml")/catalog
let $artists := doc("artists.xml")/artists
return
  <catalog>{
    for $cd in $catalog/cd
    for $artist in $artists/artist
    where $cd/@artist-id = $artist/@id
    order by $artist/name, $cd/title
    return
      <album artist="{ $artist/name }"
             country="{ $artist/country }"
             title="{ $cd/title }"
             price="{ $cd/price }"/>
  }</catalog>
```

``` xml title="result"
<catalog>
  <album artist="Bob Dylan" country="US" title="Empire Burlesque" price="10.90"/>
  <album artist="Bonnie Tyler" country="GB" title="Hide your heart" price="9.90"/>
  <album artist="Dolly Parton" country="US" title="Greatest Hits" price="9.90"/>
</catalog>
```

The two `for` clauses first form every possible pair—a **cross join**. The
`where` clause keeps only pairs whose keys match, producing an inner join.

## Put the match in the binding

The same join is often clearer as a correlated lookup:

``` xquery
for $cd in $catalog/cd
let $artist := $artists/artist[@id = $cd/@artist-id]
return
  <album artist="{ $artist/name }" title="{ $cd/title }"/>
```

This reads as “for each CD, find its artist.” It also exposes the relationship's
cardinality: `$artist` may contain zero, one, or several matches.

If the identifier must be unique, make the contract executable:

``` xquery
let $artist as element(artist) := $artists/artist[@id = $cd/@artist-id]
```

The query then fails if the master data contains no matching artist or duplicate
IDs, instead of silently producing incomplete or repeated output.

## Inner, left outer, and anti-joins

An **inner join** returns only matching rows:

``` xquery
for $cd in $catalog/cd
let $artist := $artists/artist[@id = $cd/@artist-id]
where exists($artist)
return <album title="{ $cd/title }" artist="{ $artist/name }"/>
```

A **left outer join** keeps every CD and supplies a fallback for missing master
data:

``` xquery
for $cd in $catalog/cd
let $artist := $artists/artist[@id = $cd/@artist-id]
return
  <album title="{ $cd/title }"
         artist="{ ($artist/name, "[unknown artist]")[1] }"/>
```

The expression `($artist/name, "[unknown artist]")[1]` is a useful XQuery idiom:
make a sequence of the optional value followed by a default, then take the first
item.

An **anti-join** finds broken references—the left rows with no match:

``` xquery
for $cd in $catalog/cd
where empty($artists/artist[@id = $cd/@artist-id])
return <missing-artist id="{ $cd/@artist-id }" album="{ $cd/title }"/>
```

That form is valuable in data-quality checks: “find every invoice whose supplier
ID is absent from the supplier register.”

## One-to-many joins

`let` binds the *whole matching sequence*, so one-to-many relationships need no
special syntax. Add a label document:

``` xml title="labels.xml"
<labels>
  <label id="columbia"><name>Columbia Records</name></label>
  <label id="cbs"><name>CBS Records</name></label>
</labels>
```

If an artist carries several label references, construct nested output directly:

``` xquery
for $artist in $artists/artist
let $labels-for-artist :=
  $labels/label[@id = $artist/label-ref/@id]
return
  <artist name="{ $artist/name }">{
    for $label in $labels-for-artist
    return <label>{ string($label/name) }</label>
  }</artist>
```

Relational SQL normally flattens that relationship into repeated rows. XQuery can
return the natural hierarchy: one `artist` containing any number of `label`
children.

## Join, group, aggregate

Bindings survive until `group by`, so a query can join first and summarize the
result:

``` xquery
<countries>{
  for $cd in $catalog/cd
  let $artist := $artists/artist[@id = $cd/@artist-id]
  where exists($artist)
  group by $country := string($artist/country)
  order by $country
  return
    <country code="{ $country }"
             albums="{ count($cd) }"
             total="{ sum($cd/price ! xs:decimal(.)) }"/>
}</countries>
```

After grouping, `$country` is the grouping key and `$cd` is the sequence of all
CDs in that group:

``` xml title="result"
<countries>
  <country code="GB" albums="1" total="9.90"/>
  <country code="US" albums="2" total="20.80"/>
</countries>
```

## Joining collections

The same query shape works when the left side is a database collection rather
than one file:

``` xquery
let $artists := doc("artists.xml")/artists
for $cd in collection("music")//cd
let $artist := $artists/artist[@id = $cd/@artist-id]
where $artist/country = "US"
return <album>{ $cd/title, $artist/name }</album>
```

A native XML database can use indexes for `@id = $cd/@artist-id`; a loose-file
processor may scan the artist document once per CD. The XQuery is portable, but
the query plan and index configuration are processor-specific.

## Avoid accidental quadratic work

The explicit cross-join form makes its cost visible:

``` xquery
for $cd in $catalog/cd
for $artist in $artists/artist
where $cd/@artist-id = $artist/@id
return ...
```

With 10,000 CDs and 10,000 artists, the logical candidate space is 100 million
pairs. Database optimizers can often rewrite an equality join to use an index,
but do not assume every standalone processor will.

Three practical rules help:

1. Declare indexes on stable join keys in a native XML database.
2. Inspect the processor's query plan for large collections.
3. For small in-memory master data, build a map once when portability matters:

``` xquery
let $artist-by-id :=
  map:merge(
    $artists/artist ! map:entry(string(@id), .)
  )
for $cd in $catalog/cd
let $artist := map:get($artist-by-id, string($cd/@artist-id))
return <album artist="{ $artist/name }" title="{ $cd/title }"/>
```

The map states the lookup intent directly and avoids repeatedly searching the
same tree. Duplicate keys need an explicit policy; `map:merge` options can choose
whether the first, last, or combined value wins.

## The e-invoicing version

The structure is identical with business documents. Query all UBL invoices,
extract the supplier identifier, and join it to a separate supplier register:

``` xquery
declare namespace inv =
  "urn:oasis:names:specification:ubl:schema:xsd:Invoice-2";
declare namespace cac =
  "urn:oasis:names:specification:ubl:schema:xsd:CommonAggregateComponents-2";
declare namespace cbc =
  "urn:oasis:names:specification:ubl:schema:xsd:CommonBasicComponents-2";

let $suppliers := doc("suppliers.xml")/suppliers
for $invoice in collection("invoices")/inv:Invoice
let $id := string(
  $invoice/cac:AccountingSupplierParty/cac:Party
          /cac:PartyLegalEntity/cbc:CompanyID
)
let $supplier := $suppliers/supplier[@id = $id]
return
  <invoice number="{ $invoice/cbc:ID }"
           supplier="{ ($supplier/name, "[unknown]")[1] }"
           payable="{ $invoice/cac:LegalMonetaryTotal/cbc:PayableAmount }"/>
```

From there, `group by` can summarize spend per supplier, while an anti-join can
report invoices that cannot be matched to master data. The mechanics are the same
as the catalog example; only the paths and namespaces are richer.

## Where to go next

- [Maps, arrays, and JSON](maps-arrays-json.md) turns the lookup map above into a
  general tool and joins JSON data directly to XML.
- [Constructing XML precisely](constructing-xml.md) explains exactly what happens
  to the nodes and values returned by a join.
