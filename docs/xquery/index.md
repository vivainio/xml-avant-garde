---
icon: lucide/database
---

# XQuery

**XQuery** is a full query and transformation language for XML. Where
[XPath](../xpath/index.md) only *addresses* a document and
[XSLT](../xslt/index.md) transforms it with template rules, XQuery lets you
*query* one or many documents the way SQL queries tables — selecting, filtering,
joining, grouping, and building new XML from the result.

It is best understood as XPath's bigger sibling. XQuery 3.1 and XSLT 3.0 are
built on the **same foundation**:

- the same expression language — [XPath](../xpath/index.md) 3.1 is a strict
  subset of XQuery,
- the same data model — the **XDM** (sequences of nodes and atomic values),
- the same function library — `fn:` functions, `map:`/`array:`, higher-order
  functions, JSON support.

So almost everything you learned about [XPath](../xpath/index.md) carries over
unchanged. XQuery adds the surrounding *query* machinery: **FLWOR**
expressions, element constructors, and a module system.

## The shape of a query

A FLWOR expression (**F**or, **L**et, **W**here, **O**rder by, **R**eturn —
pronounced "flower") is the heart of the language. It reads top to bottom like a
loop, and like SQL it ends in a projection:

``` xquery title="expensive.xq"
xquery version "3.1";

<report>{
  for $cd in /catalog/cd
  let $price := xs:decimal($cd/price)
  where $price > 9.90
  order by $cd/title
  return <cd title="{ $cd/title }" price="{ $price }"/>
}</report>
```

Run against the [running CD catalog](#the-running-example), this produces a
brand-new document:

``` xml title="result"
<report>
  <cd title="Empire Burlesque" price="10.9"/>
</report>
```

Notice the two things XPath cannot do on its own: the `for…return` **loops and
filters**, and the literal `<report>…</report>` with `{ … }` holes is a **direct
element constructor** — XQuery builds output XML inline, no `xsl:element`
needed.

## XQuery is not Saxon-only

A common worry is that XQuery lives or dies with [Saxon](https://www.saxonica.com/).
The opposite is true — and it is the reverse of XSLT 3.0, where Saxon is
effectively the *only* complete implementation. XQuery has a **healthy
multi-vendor story**, because the native-XML-database world adopted it as its
query language:

| Implementation | What it is |
| --- | --- |
| [**BaseX**](https://basex.org/) | Open-source native XML database, full XQuery 3.1 + Update + Full-Text. The most active engine today; ships a good CLI |
| [**eXist-db**](https://exist-db.org/) | Open-source native XML database; XQuery 3.1, RESTXQ — build whole web apps in it |
| **MarkLogic** | Commercial enterprise XML database (now Progress); XQuery was its first-class query language |
| **Saxon** | The go-to *standalone/embedded* processor — Java, .NET, C, Python |
| **SQL Server** | A real XQuery *subset* embedded in the `xml` column type (`.query()`, `.value()`, `.modify()`) |
| **Oracle XML DB / DB2 pureXML** | XQuery embedded inside the relational engine |

The catch is only that most engines are *databases*: "trying XQuery" usually
means installing BaseX rather than piping a file through a CLI — though both
BaseX and Saxon run a standalone `.xq` file perfectly well. See
[XQuery in the real world](real-world.md) for what each is used for.

## XQuery versions

- **1.0** (2007) established FLWOR, constructors, and the XPath 2.0 type system.
- **3.0** (2014) added `group by`, `count`, the `=>` arrow operator, inline
  functions, and `try`/`catch`.
- **3.1** (2017) is the current version: **maps and arrays**, native JSON
  (`fn:parse-json`, `fn:json-doc`), and lockstep parity with XSLT 3.0's
  function library.

This section teaches 3.1, the version every current engine implements.

## The running example

Like the [XPath](../xpath/index.md) and [XSLT](../xslt/index.md) sections, every
page queries the same small CD catalog, so you can focus on the XQuery rather
than re-learning the data:

``` xml title="catalog.xml"
<catalog>
  <cd genre="rock"><title>Empire Burlesque</title><artist>Bob Dylan</artist><price>10.90</price><year>1985</year></cd>
  <cd genre="pop"><title>Hide your heart</title><artist>Bonnie Tyler</artist><price>9.90</price></cd>
  <cd genre="country"><title>Greatest Hits</title><artist>Dolly Parton</artist><price>9.90</price></cd>
</catalog>
```

!!! tip "Where does the document come from?"
    The paths above start from `/catalog`, which assumes the catalog is the
    processor's *context item*. Standalone queries usually load it explicitly
    with `doc("catalog.xml")/catalog/cd`; database queries pull from a whole
    indexed **collection** with `collection("music")//cd`. See
    [Querying collections](flwor.md#input-functions).

## Where to go next

1. [FLWOR expressions](flwor.md) — `for`/`let`/`where`/`order by`/`return`, `group by`, and building XML output.
2. [XQuery vs XSLT](xquery-vs-xslt.md) — two languages, one data model: when to reach for which.
3. [XQuery in the real world](real-world.md) — native XML databases, RESTXQ web apps, the Update Facility, and embedded SQL XQuery.
