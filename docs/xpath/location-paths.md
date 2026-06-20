# Location paths

A **location path** is how XPath points at nodes in a document. It is a sequence
of **steps** separated by `/`. Each step selects a set of nodes relative to the
current *context*; the next step then runs once from each of those nodes, and
the results are gathered together. A path is read left to right, narrowing or
shifting the context one step at a time.

For these examples the running catalog gains a `genre` attribute on each `cd`,
and the first CD keeps an optional `<year>` child while the others omit it:

``` xml title="catalog.xml" linenums="1"
<catalog>
  <cd genre="rock"><title>Empire Burlesque</title><artist>Bob Dylan</artist><price>10.90</price><year>1985</year></cd>
  <cd genre="pop"><title>Hide your heart</title><artist>Bonnie Tyler</artist><price>9.90</price></cd>
  <cd genre="country"><title>Greatest Hits</title><artist>Dolly Parton</artist><price>9.90</price></cd>
</catalog>
```

## Absolute and relative paths

A leading `/` anchors the path at the **document root** — the invisible node
that sits above the outermost element. From there `catalog` selects the root
element, and `catalog/cd` reads as "every `cd` child of the `catalog` child of
the root".

``` xpath
/catalog/cd/title     <!-- (1)! -->
```

1.  Absolute: start at the root, walk down `catalog`, then `cd`, then `title`.

<div class="xslt-result" markdown>
Empire Burlesque

Hide your heart

Greatest Hits
</div>

Without the leading `/`, the path is **relative**: it begins at whatever the
current context node happens to be. If the context is a `cd`, then `title`
selects that CD's title and `cd/title` would look for a `cd` child of the
current `cd` (here, nothing).

``` xpath
cd/title              <!-- relative to the current node -->
```

!!! note "Two ways to say the same thing"
    `/catalog/cd` and `catalog/cd` select the same nodes only when the context
    is already the root. The leading slash removes that dependence on context —
    it always starts from the top.

## The `//` descendant shorthand

`//` is shorthand for "descendant-or-self": it matches at **any depth**, not
just the immediate child. `//title` finds every `title` anywhere in the
document, however deeply nested:

``` xpath
//title
```

<div class="xslt-result" markdown>
Empire Burlesque

Hide your heart

Greatest Hits
</div>

It can also appear in the middle of a path. `catalog//price` selects every
`price` at any depth beneath `catalog`:

``` xpath
catalog//price
```

<div class="xslt-result" markdown>
10.90

9.90

9.90
</div>

!!! warning "`//title[1]` is not the first title in the document"
    Because each step runs from *every* node reached so far, `//title[1]`
    applies the `[1]` predicate **per parent**: it keeps the first `title`
    within each element that has one — which here is all three titles. To get
    the single first `title` in the whole document, parenthesise the node-set
    first: `(//title)[1]`. See [Node tests and predicates](node-tests-predicates.md)
    and the XSLT page on [XPath predicates](../xslt/predicates.md).

!!! tip "`//` can be expensive"
    `//` searches the entire subtree, so a path like `//price` may scan every
    node in the document. When you know where the data lives, a specific path
    such as `/catalog/cd/price` is both clearer and cheaper.

## Self, parent, and attributes

Two steps refer to the context itself and its parent:

- `.` is the **current node** (self).
- `..` is the **parent** of the current node.

So from a `title`, `..` reaches its `cd`, and `../price` reaches that CD's
price. `.` is mostly used to pass the current node to a function or to root a
relative path explicitly, as in `.//title`.

Attributes are addressed with the `@` prefix. They are not children, so they
need their own step:

``` xpath
cd/@genre             <!-- (1)! -->
```

1.  The `genre` attribute of each `cd`.

<div class="xslt-result" markdown>
rock

pop

country
</div>

## The union operator

`|` forms the **union** of two node-sets — every node selected by either side.
`title | artist` selects both the titles and the artists:

``` xpath
catalog/cd/title | catalog/cd/artist
```

<div class="xslt-result" markdown>
Empire Burlesque

Bob Dylan

Hide your heart

Bonnie Tyler

Greatest Hits

Dolly Parton
</div>

The two operands need not be related — `title | @genre` is perfectly legal. The
result is a single node-set, with the usual node-set rules below.

## A step result is a node-set

In XPath 1.0 every location path evaluates to a **node-set**. Three properties
follow from that:

- It is conceptually **unordered** — a set, not a list.
- It contains **no duplicates**; a node selected by both sides of a `|` appears
  once.
- When you *consume* it — iterate, take `[1]`, or render it — the processor
  visits the nodes in **document order** (the order they appear in the source).

This is why `(//title)[1]` is well defined: the parentheses build the whole
node-set first, then positional `[1]` picks the document-order first node.

## Common path shapes

| Path | Selects |
| --- | --- |
| `/catalog` | the root `catalog` element |
| `/catalog/cd` | every `cd` child of `catalog` |
| `catalog/cd/title` | each CD's `title` (relative to the root context) |
| `//title` | every `title` anywhere in the document |
| `catalog//price` | every `price` at any depth under `catalog` |
| `cd/@genre` | the `genre` attribute of each `cd` |
| `.` | the current context node |
| `..` | the parent of the current node |
| `../price` | the `price` sibling, reached via the parent |
| `title | artist` | the union of all titles and artists |
| `(//cd)[1]` | the single first `cd` in the document |
| `//title[1]` | the first `title` within *each* parent (a gotcha) |

## Next

Every step so far — `cd`, `..`, `@genre`, `//` — is an abbreviation of a fuller
syntax that names a direction through the tree. [Axes](axes.md) unfolds those
abbreviations and shows the directions you cannot reach without them.
