---
icon: lucide/database-zap
---

# Updating and indexing XML databases

Core XQuery is functional: it reads XDM values and constructs new ones. The
optional **XQuery Update Facility** (XQUF) adds operations that change nodes in a
persistent XML database. Database engines then add transactions, indexes, query
plans, and maintenance policies around that standard language.

Keep those layers separate:

| Layer | Defines |
| --- | --- |
| XQuery 3.1 | selection, FLWOR, functions, construction |
| XQuery Update Facility | insert, delete, replace, rename, pending updates |
| database engine | storage, locking, commits, indexes, optimization commands |

XQUF 3.0 is a W3C Working Group Note rather than part of the XQuery 3.1
Recommendation. Support is optional. BaseX and eXist-db implement it; a standalone
processor may be read-only or expose different persistence APIs.

!!! info "Standards and implementation references"
    The portable semantics come from the W3C
    [XQuery Update Facility 3.0](https://www.w3.org/TR/xquery-update-30/).
    Database examples use the current BaseX documentation for
    [updates](https://docs.basex.org/main/Updates),
    [indexes](https://docs.basex.org/main/Indexes), and
    [transaction management](https://docs.basex.org/main/Transaction_Management).

## The four basic updates

Given a stored catalog, insert a child:

``` xquery
insert node <year>1985</year>
into db:get("music")/catalog/cd[title = "Empire Burlesque"]
```

Choose the location explicitly when order matters:

``` xquery
insert node <year>1985</year>
after db:get("music")/catalog/cd[title = "Empire Burlesque"]/price
```

Delete nodes:

``` xquery
delete nodes db:get("music")/catalog/cd[@discontinued = "true"]
```

Replace a whole node:

``` xquery
replace node
  db:get("music")/catalog/cd[title = "Hide your heart"]/price
with
  <price currency="EUR">9.50</price>
```

Replace only its value:

``` xquery
replace value of node
  db:get("music")/catalog/cd[title = "Hide your heart"]/price
with
  "9.50"
```

Rename an element or attribute:

``` xquery
rename node db:get("music")/catalog/cd/artist
as "performer"
```

The source and target expressions are ordinary XQuery. XQUF adds only the update
operation around the nodes they select.

!!! note "`db:get` is BaseX-specific"
    BaseX uses `db:get("music")` to open a database. Other engines use
    `collection()`, `doc()`, or vendor functions. The `insert`, `delete`,
    `replace`, and `rename` expressions are the portable XQUF part.

## Updating several targets

Portable code makes the iteration explicit:

``` xquery
for $price in db:get("music")/catalog/cd[@genre = "rock"]/price
return replace value of node $price
       with xs:decimal($price) * 1.10
```

Some engines accept several target nodes directly as an extension. The FLWOR form
states the cardinality clearly and travels better between implementations.

Before running a bulk update, execute the target expression alone:

``` xquery
db:get("music")/catalog/cd[@genre = "rock"]/price
```

Inspecting the exact target set is the database equivalent of previewing a
filesystem operation before changing files.

## Updates are pending, not immediate

An updating expression contributes primitives to a **pending update list**.
XQUF evaluates the query against one snapshot, checks the collected operations
for conflicts, then applies them at the end.

This does not behave like an imperative loop:

``` xquery
let $catalog := db:get("music")/catalog
return (
  insert node <cd><title>New</title></cd> into $catalog,
  count($catalog/cd)
)
```

In a strictly conforming configuration, mixing an updating expression and an
ordinary returned value may be rejected. Even where an engine extension permits
it, `count` sees the original snapshot—the insertion has not happened yet.

The pending-update model provides two useful properties:

- query results do not depend on the textual order in which updates happened to
  be discovered;
- conflicting operations can be detected before partially changing the data.

It also means “insert, then query the inserted node” normally requires a second
query or an engine-specific scripting facility.

## Persistent update versus modified copy

Use an updating expression on a database node when the stored document should
change. Use `copy`/`modify`/`return` when the source should remain untouched and
the query should return a transformed copy:

``` xquery
copy $copy := doc("catalog.xml")/catalog
modify (
  for $price in $copy/cd/price
  return replace value of node $price
         with format-number(xs:decimal($price), "0.00")
)
return $copy
```

The original document is unchanged; `$copy` has new node identity. This occupies
the middle ground between ordinary constructors and persistent mutation.

BaseX also offers a shorter `update { ... }` expression for main-memory copies,
but `copy`/`modify`/`return` expresses the standard facility.

## Updating functions

Move repeated mutations into an updating function:

``` xquery
declare %updating function local:set-price(
  $cd as element(cd),
  $price as xs:decimal
) {
  replace value of node $cd/price with $price
};

for $cd in db:get("music")/catalog/cd[@genre = "rock"]
return local:set-price($cd, xs:decimal($cd/price) * 1.10)
```

The `%updating` annotation tells the static checker that the body may contribute
to the pending update list. Updating and non-updating expressions cannot be mixed
arbitrarily; exact constraints depend on the XQUF version and extensions enabled
by the engine.

Keep target selection separate from the mutation when practical:

``` xquery
declare function local:rock-prices($db as document-node()) as element(price)* {
  $db/catalog/cd[@genre = "rock"]/price
};
```

The selector is pure, easy to test, and reusable in a preview report. The small
updating wrapper performs the state change.

## Transactions belong to the engine

XQUF defines snapshots and pending updates, but persistence and concurrency are
implementation-defined. In BaseX, the updates from one updating query are applied
atomically: if an update operation fails, the overall transaction is aborted.
BaseX allows concurrent readers and serializes writers through its transaction
management.

Do not generalize those exact locking rules to every XQuery implementation.
Document the guarantees of the database used in production, especially when a
workflow also writes files, calls HTTP services, or changes another database:
external effects may not participate in the XML database transaction.

## Why indexes matter

Without an index, this predicate may scan every `CompanyID` in every invoice:

``` xquery
db:get("invoices")//cbc:CompanyID[. = "FI12345678"]
```

A value index maps the searched value to matching nodes. The query stays
declarative; the optimizer decides whether an index can answer it.

BaseX maintains several distinct structures:

| Index | Accelerates |
| --- | --- |
| text | equality and range comparisons on element text |
| attribute | equality and range comparisons on attribute values |
| token | membership of whitespace-separated attribute tokens |
| full-text | word, phrase, stemming, wildcard, and fuzzy text search |
| path/name/statistics | navigation planning and cardinality estimates |

An index is not “an index on the XML document” in the abstract. Choose the
structure matching the operation: exact supplier IDs, numeric amount ranges,
token-valued classifications, or linguistic search.

## Write index-friendly predicates

A direct comparison gives the optimizer a recognizable shape:

``` xquery
db:get("invoices")//cbc:CompanyID[text() = "FI12345678"]
```

Wrapping every stored value in a function can hide that shape:

``` xquery
db:get("invoices")//cbc:CompanyID[
  lower-case(normalize-space(.)) = "fi12345678"
]
```

Some optimizers rewrite common expressions; others fall back to a scan. Prefer
normalizing stable identifiers before storage, or normalize the search parameter
once:

``` xquery
let $id := upper-case(normalize-space($supplied-id))
return db:get("invoices")//cbc:CompanyID[. = $id]
```

The same principle applies to joins: expose equality between stored keys rather
than burying it inside opaque user functions.

## Full-text is not `contains`

Substring search answers a character question:

``` xquery
contains(lower-case($description), "invoice")
```

XQuery Full Text answers a linguistic query and can use a full-text index:

``` xquery
$description contains text "invoice"
  using case insensitive
```

Full-text implementations may add phrases, stemming, stop words, fuzzy matching,
wildcards, distances, and scoring. Those features are a separate optional W3C
extension with processor-specific details. Use them for document search; keep
exact codes and identifiers on value indexes.

## Inspect the query plan

Do not infer index use from a fast test on small data. Inspect the optimized query
plan:

1. Run the query with the engine's query-info or plan view.
2. Look for an index-access operator or an “index rewrite” message.
3. Compare the estimated result count with reality.
4. Test on production-shaped data, with warm and cold caches distinguished.

BaseX exposes query information in its GUI and command-line tooling. If a
predicate is not rewritten for index access, simplify it and check that the
relevant index exists and includes the selected element or attribute names.

Do not force an index merely because one exists. For a value occurring on most
nodes, a sequential scan can be cheaper than millions of index hits; a capable
optimizer uses statistics to choose.

## Updates and stale indexes

Index maintenance is an operational policy. Current BaseX behavior distinguishes:

- rebuilding invalidated structures with `db:optimize`;
- `UPDINDEX`, which incrementally maintains value indexes;
- `AUTOOPTIMIZE`, which rebuilds remaining stale structures and statistics after
  updates.

Incremental maintenance costs write time; rebuilding later can leave queries slow
between update and optimization. Full-text indexes and statistics have different
maintenance constraints from value indexes. Choose based on workload:

| Workload | Typical policy |
| --- | --- |
| read-heavy, occasional batch import | update in bulk, then optimize |
| small database requiring fresh indexes | automatic optimization may be acceptable |
| frequent writes, exact-value lookups | incremental value-index maintenance |
| large full-text corpus | scheduled rebuilds and explicit freshness monitoring |

These option names are BaseX-specific and can change by version. Treat them as an
example of the decision every database requires, not portable XQuery syntax.

## A safe bulk-update workflow

For a change such as correcting VAT category codes across invoices:

1. Write a read-only query that selects targets and reports document URI, invoice
   number, old value, and proposed value.
2. Verify the count and sample records.
3. Back up the database or confirm restore capability.
4. Reuse the same selector in the updating query.
5. Run the update as one controlled transaction or bounded batches.
6. Run a postcondition query proving no old values remain and only expected new
   values appeared.
7. Rebuild or verify indexes and statistics according to database policy.
8. Run the ordinary validation pipeline over the changed invoices.

XQUF makes mutation compact; it does not remove the need for preview,
postconditions, backups, and domain validation.

## Portability boundary

Portable:

- `insert`, `delete`, `replace`, and `rename`;
- `copy`/`modify`/`return`;
- the pending-update snapshot model;
- `%updating` functions.

Engine-specific:

- database-opening functions such as `db:get`;
- transaction and locking guarantees;
- whether updating and simple results may be mixed;
- index types, configuration, plan display, and maintenance;
- scripting across several update snapshots.

Write the business selection and calculations as ordinary XQuery functions, then
isolate persistence calls and engine annotations in a thin module. That keeps the
largest part of the application testable and portable.

## Where to go next

- [Capstone: query a collection of UBL invoices](invoice-report.md) combines
  collections, joins, diagnostics, grouping, XML, and JSON in one reporting
  pipeline.
- [XQuery in the real world](real-world.md) places database querying and RESTXQ
  alongside standalone and embedded processors.
- [The validation pipeline](../einvoicing/validation-pipeline.md) shows the
  checks to run after modifying invoice XML.
