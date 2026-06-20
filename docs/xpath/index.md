---
icon: lucide/route
---

# XPath

**XPath** is a small expression language for *selecting* and *computing over*
parts of an XML document. You write a short expression that navigates the
document's tree and returns the nodes — or the number, string, or boolean —
that it picks out.

XPath is **not** a transformation language by itself. It has no way to produce
a new document; it only *addresses* and *queries* an existing one. The job of
turning those selections into output belongs to a **host** language that embeds
XPath.

## Where XPath shows up

If you have used the [XSLT Tutorial](../xslt/index.md), you have already written
XPath: every `select` and every `match` is an XPath expression, and XSLT is the
host most readers meet first. But the same language is the addressing layer in
many other places:

- **XSLT** — `select` (what to pull) and `match` (which template fires).
- **XQuery** — XPath is its navigation core, wrapped in a fuller query language.
- **XML Schema** — `xs:selector` and `xs:field` in identity constraints
  (`xs:key`, `xs:unique`).
- **Schematron** — assertion rules are XPath expressions over the document.
- **DOM APIs** — `document.evaluate(...)` in browsers evaluates XPath at runtime.
- **Library bindings** — Python's `lxml`, Java's `javax.xml.xpath`, and many
  others let code run XPath against a parsed tree.

The expression syntax is the same everywhere; only the surrounding host
differs.

## The data model: a tree of nodes

XPath sees an XML document not as text but as a **tree of nodes**. There are
seven node types:

| Node type | What it is |
| --- | --- |
| Root / document | the invisible top of the tree, *above* the document element |
| Element | a tagged element such as `<cd>` |
| Attribute | a name/value pair such as `genre="rock"` |
| Text | the character content inside an element |
| Comment | an `<!-- ... -->` node |
| Processing-instruction | a `<?target data?>` node |
| Namespace | a namespace declaration in scope on an element |

!!! note "Attributes and namespaces are not children"
    Although they are written *inside* an element's start tag, attribute and
    namespace nodes are **not** children of that element. They hang off the
    element specially, on their own axes. This trips people up constantly:
    `cd/*` and `cd/node()` will never reach a `genre` attribute — you address it
    deliberately with `@genre` (the attribute axis). The text inside `<title>`,
    by contrast, *is* a child of `title`.

## Context: every expression is relative

An XPath expression is always evaluated relative to a **context node**, plus a
context **position** and **size**. Together these are the *context* in which the
expression runs:

- The **context node** is where navigation starts. A relative path like
  `cd/title` is interpreted *from* that node.
- The **context position** is the node's index within the set currently being
  processed, returned by `position()`.
- The **context size** is how many nodes are in that set, returned by `last()`.

The host sets the context. In XSLT, the context node is the *current node* —
the node a template matched, or the node being visited inside an
`xsl:for-each`. The same expression therefore selects different things
depending on where the host points it.

## A first taste

Every page in this section queries the same small CD catalog, so you can focus
on the XPath rather than re-learning the data:

``` xml title="catalog.xml" linenums="1"
<catalog>
  <cd genre="rock"><title>Empire Burlesque</title><artist>Bob Dylan</artist><price>10.90</price><year>1985</year></cd>
  <cd genre="pop"><title>Hide your heart</title><artist>Bonnie Tyler</artist><price>9.90</price></cd>
  <cd genre="country"><title>Greatest Hits</title><artist>Dolly Parton</artist><price>9.90</price></cd>
</catalog>
```

Evaluated with the document root as the context node:

```
catalog/cd            (1)
//title               (2)
cd[1]/price           (3)
count(catalog/cd)     (4)
```

1. All three `<cd>` elements — the `cd` children of the top-level `catalog`.
2. Every `<title>` anywhere in the document, regardless of depth.
3. The `<price>` of the *first* `cd` (predicates are 1-based).
4. A *number*, not a node-set — `3`, the count of `cd` children.

<div class="xslt-result" markdown>
`catalog/cd` → the 3 `cd` elements

`//title` → Empire Burlesque, Hide your heart, Greatest Hits

`cd[1]/price` → 10.90

`count(catalog/cd)` → 3
</div>

Notice that the first three expressions return **nodes** while the last returns
a plain **number** — XPath computes values as well as locating nodes.

## XPath versions

- **XPath 1.0** is built around four data types, with the **node-set** as its
  central collection. It is universally supported, and is what XSLT 1.0 uses.
- **XPath 2.0 / 3.0** replace node-sets with ordered **sequences**, add a real
  type system tied to XML Schema, and ship a far larger function library
  (regular expressions, date arithmetic, `for`/`if`/`some`/`every`
  expressions, and more). These need a 2.0+ processor such as the one behind
  XSLT 2.0/3.0.

This section teaches **1.0**, with notes where 2.0+ changes the picture —
mirroring the [XSLT Tutorial](../xslt/index.md). The 1.0 foundations carry over
unchanged.

## Where to go next

1. [Location paths](location-paths.md) — steps, `/` and `//`, building a path.
2. [Axes](axes.md) — moving in any direction through the tree.
3. [Node tests and predicates](node-tests-predicates.md) — choosing what each step matches and filtering it.
4. [Functions and data types](functions-and-types.md) — the core function library and XPath's four 1.0 types.
