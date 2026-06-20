---
icon: lucide/boxes
---

# XQuery in the real world

XQuery's natural habitat is the **native XML database**. A standalone `.xq`
file run over one document is the textbook case, but the reason the language has
several independent, well-funded implementations is that an entire category of
database picked it as their query language — the way relational databases picked
SQL.

## Native XML databases

A native XML database stores XML documents directly (not shredded into tables),
indexes their structure and text, and exposes XQuery as the way to ask
questions. The `collection()` function is the doorway: where a standalone query
reads one `doc()`, a database query ranges over a whole indexed collection.

``` xquery title="across thousands of documents, using indexes"
for $cd in collection("music")//cd
where $cd/@genre = "jazz"
order by xs:decimal($cd/price) descending
return <hit>{ string($cd/title) }</hit>
```

The three you will meet:

| Engine | Niche |
| --- | --- |
| [**BaseX**](https://basex.org/) | Lightweight, open source, fast. Excellent CLI and GUI; the easiest way to *learn* XQuery against real data |
| [**eXist-db**](https://exist-db.org/) | Open source, application-platform leaning — people build entire web apps inside it with RESTXQ |
| **MarkLogic** | Commercial, enterprise scale; heavy use in publishing, government, and finance |

## RESTXQ — web apps in XQuery

eXist-db and BaseX implement **RESTXQ**: you annotate an XQuery function with the
HTTP route it serves, and the database becomes the web server. No application
tier, no ORM — the query *is* the endpoint.

``` xquery title="a REST endpoint"
declare
  %rest:GET
  %rest:path("/cds/{$genre}")
  %output:method("json")
function page:by-genre($genre as xs:string) {
  array {
    for $cd in collection("music")//cd[@genre = $genre]
    return map { "title": string($cd/title), "price": $cd/price }
  }
};
```

A request to `/cds/rock` runs the query and serializes the result — here as JSON,
thanks to XQuery 3.1's [maps and arrays](flwor.md#group-by-aggregate-30). This
is how document-publishing platforms (scholarly journals on JATS, archives on
TEI, legal corpora on Akoma Ntoso) are commonly built.

## The XQuery Update Facility

Plain XQuery is read-only — it *builds* new XML, it does not change stored
documents. The **XQuery Update Facility** (XQUF), supported by BaseX, eXist-db,
and others, adds in-place mutation, which only makes sense against a database:

``` xquery
for $cd in collection("music")//cd[@genre = "rock"]
return replace value of node $cd/price with xs:decimal($cd/price) * 1.1
```

`insert`, `delete`, `rename`, and `replace` node operations let XQuery serve as
the update language too — closing the loop that XSLT, a pure transformation
language, never tries to.

## Embedded XQuery in relational databases

You do not need a *native* XML database to meet XQuery — several relational
engines embed a subset for querying `xml`-typed columns:

- **SQL Server** exposes `.query()`, `.value()`, `.exist()`, and `.modify()`
  methods on its `xml` data type — a genuine, very widely deployed XQuery
  subset.
- **Oracle XML DB** offers `XMLQuery` / `XMLTable`, and **IBM DB2 pureXML**
  embeds XQuery in SQL.

These are partial implementations, but they put XQuery on an enormous number of
production machines that no one thinks of as "XML databases."

## Standalone and embedded processing

When the job is not a database query, [**Saxon**](https://www.saxonica.com/) is
the usual choice — run a query from the command line, or call it from Java,
.NET, Python, or C:

``` bash
saxon-xquery -q:expensive.xq -s:catalog.xml
```

BaseX has an equally good standalone CLI (`basex expensive.xq`), so you can work
through the [FLWOR](flwor.md) examples on this site with either, no server
required.

!!! note "The honest caveat"
    XQuery's strength is also its friction: most engines are *databases*, so the
    quickest way in is to install BaseX rather than reach for a one-line CLI you
    already have. But the multi-vendor reality is real — unlike
    [XSLT 3.0](../xslt/moving-to-3.md), where Saxon stands nearly alone, XQuery
    has several independent, conformant implementations precisely because the
    database world depends on it.

## Where to go next

- [FLWOR expressions](flwor.md) — the syntax behind every query above.
- [XQuery vs XSLT](xquery-vs-xslt.md) — choosing the right tool for the shape of the job.
- [Working with XML in code](../realworld/xml-in-code.md) — how programs parse and process XML around these queries.
