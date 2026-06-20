# Node tests and predicates

A location step has three parts: `axis::nodetest[predicate]`. The axis chooses a
*direction* through the tree; the **node test** then decides *which nodes on
that axis* the step keeps; and any **predicates** filter that result further.
This page covers the last two — what a node test can match, and how predicates
narrow a node-set.

For the examples the catalog gains a `genre` attribute, an optional `<year>` on
the first CD, and a comment plus the usual whitespace between elements — so we
have something for the kind tests to bite on:

``` xml title="catalog.xml" linenums="1"
<catalog>
  <!-- imported 2024 -->
  <cd genre="rock"><title>Empire Burlesque</title><artist>Bob Dylan</artist><price>10.90</price><year>1985</year></cd>
  <cd genre="pop"><title>Hide your heart</title><artist>Bonnie Tyler</artist><price>9.90</price></cd>
  <cd genre="country"><title>Greatest Hits</title><artist>Dolly Parton</artist><price>9.90</price></cd>
</catalog>
```

## Node tests

A node test is written *after* the axis (or after the `@`, `//`, `/` shorthands,
which imply an axis). It comes in two flavours: **name tests**, which match by
name, and **kind tests**, which match by node *type*.

### Name tests

A bare name matches the nodes on the axis that have that name. On the default
child axis it means *elements*; on the attribute axis it means *attributes*.

```
cd                (1)
@genre            (2)
*                 (3)
@*                (4)
```

1.  Element children named `cd`.
2.  The `genre` attribute (the `@` selects the attribute axis).
3.  *All* element children, whatever their name.
4.  All attributes, whatever their name.

So `*` is the wildcard name test. Its meaning follows the axis: on the child
axis `*` is "any element node", but on the attribute axis `@*` is "any attribute
node".

!!! note "`*` is element-only on the child axis"
    `*` never matches text, comment, or processing-instruction nodes — it is a
    *name* test, and only elements and attributes have names. To reach the other
    node types you need a kind test (below).

#### Namespaced name tests

When the document uses namespace prefixes, a name test can be qualified. The
prefix is matched by the **namespace it is bound to**, not by the literal
prefix string — the expression's prefix and the document's prefix only have to
resolve to the same namespace URI, they need not be spelled the same.

```
cbc:price         (1)
cbc:*             (2)
*:price           (3)
```

1.  Elements named `price` in the namespace bound to `cbc`.
2.  Any element in the `cbc` namespace, whatever its local name.
3.  Any element with local name `price`, in *any* namespace (XPath 2.0+).

!!! info "`*:local` needs 2.0+"
    `prefix:*` is available in XPath 1.0, but the `*:local` form (wildcard
    namespace, fixed local name) is an XPath 2.0 addition. In 1.0 you match
    across namespaces with `local-name()` instead, e.g. `*[local-name()='price']`.

### Kind tests

A kind test selects by node *type* and is written with trailing parentheses, so
it never collides with a name test. There are four:

```
node()                      (1)
text()                      (2)
comment()                   (3)
processing-instruction()    (4)
```

1.  *Any* node on the axis — elements, text, comments, and PIs alike.
2.  Text nodes only (the character content of an element).
3.  Comment nodes (`<!-- ... -->`).
4.  Processing-instruction nodes (`<?target data?>`); pass a name to pin the
    target, e.g. `processing-instruction('xml-stylesheet')`.

#### `text()` versus selecting the element

Selecting the element and selecting its text are different nodes. `title`
returns the `<title>` *element*; `title/text()` returns the *text node* inside
it:

```
title             (1)
title/text()      (2)
```

1.  The `<title>` element node.
2.  The text node it contains — the characters `Empire Burlesque`.

<div class="xslt-result" markdown>
`title` → the `<title>` element (string value: `Empire Burlesque`)

`title/text()` → the text node `Empire Burlesque`
</div>

In practice you often *don't* write `text()`: taking the string value of the
element (as `string()` or a host's value-of does) gathers all descendant text
for you, whereas `text()` returns the immediate text-node children only — which
matters when an element has mixed content split by child elements.

#### Why `node()` is broader than `*`

`*` is an element name test; `node()` is the all-types kind test. Asked for the
children of a `cd`, they return different sets:

```
catalog/cd[1]/*           (1)
catalog/cd[1]/node()      (2)
```

1.  The element children: `title`, `artist`, `price`, `year`.
2.  *Every* child node — those same elements **plus** any text nodes (and
    comments) between and around them.

Asked for the children of `catalog`, `catalog/node()` also picks up the
`<!-- imported 2024 -->` comment and the whitespace text nodes, none of which
`catalog/*` would return. Neither test ever reaches `@genre`, though: attributes
are not children (see the data-model note on the [section index](index.md)) and
live only on the attribute axis.

### Node tests at a glance

| Node test | Matches |
| --- | --- |
| `cd` | elements named `cd` on the axis |
| `@genre` | the `genre` attribute |
| `*` | all element nodes (all attributes on the attribute axis) |
| `cbc:*` | all elements in the namespace bound to `cbc` |
| `*:price` | elements with local name `price` in any namespace (2.0+) |
| `node()` | any node — element, text, comment, or PI |
| `text()` | text nodes |
| `comment()` | comment nodes |
| `processing-instruction()` | processing-instruction nodes |

## Predicates

A **predicate** is a `[ ... ]` filter on a step. The step first produces a
node-set; the predicate is then tested against each node in turn and keeps only
those for which it is true. The kinds you will use — comparison, existence,
attribute, and positional — are worked through with XSLT-context examples on the
[XPath predicates](../xslt/predicates.md) page; this page looks at the same
mechanism from the *XPath-language* side.

### Each candidate is the context node

A predicate is evaluated once per candidate node, *with that node as the context
node*. The expression inside the brackets is therefore relative to the node
being tested. Within the step, the context **position** and **size** are set
from the step's node-set, so `position()` and `last()` count over the nodes the
step selected — not over the whole document.

```
catalog/cd[price > 10]      (1)
catalog/cd[position() = last()]   (2)
```

1.  For each `cd`, evaluate `price > 10` with that `cd` as context — keep the
    ones whose `<price>` child exceeds 10.
2.  The last `cd` in the set, since `position()` and `last()` range over the
    three `cd` nodes.

### A number is shorthand for `position()`

A predicate that is just a number is shorthand for "the node at that position":
a numeric predicate `[n]` means `[position() = n]`. Positions are **1-based**.

```
cd[2]                 (1)
cd[position() = 2]    (2)
```

1.  The second `cd`.
2.  Exactly the same thing — `[2]` expands to this.

<div class="xslt-result" markdown>
`cd[2]` → the *Hide your heart* CD
</div>

### Truthiness inside a predicate

The bracketed expression is coerced to a boolean, and the rule depends on its
type:

- A **node-set** is true when it is **non-empty** — so a bare name is an
  *existence* test.
- A **number** is true when it is neither `0` nor `NaN`.
- A **string** is true when it is non-empty.

```
cd[year]              (1)
cd[@genre]            (2)
cd[not(year)]         (3)
```

1.  CDs that *have* a `<year>` child — true because the `year` node-set is
    non-empty. Only the first CD qualifies.
2.  CDs that carry a `genre` attribute — here, all three.
3.  The inverse: CDs with no `<year>`.

<div class="xslt-result" markdown>
`cd[year]` → Empire Burlesque

`cd[not(year)]` → Hide your heart, Greatest Hits
</div>

!!! warning "A bare number is not an existence test"
    `cd[year]` tests *existence* (a node-set), but `cd[1]` tests *position* (a
    number expands to `position() = 1`). The two look similar but mean
    different things — a name in brackets asks "does it exist?", a literal
    number asks "is it in this position?".

### Multiple predicates filter left to right

Predicates chain. Each one is applied to the node-set the previous one produced,
and re-numbers it before the next runs — so order changes the result.

```
cd[@genre='rock'][1]      (1)
cd[1][@genre='rock']      (2)
```

1.  Keep the rock CDs, *then* take the first of those.
2.  Take the first CD, *then* keep it only if it is rock.

Here both yield the first CD (it is the rock one), but they diverge as soon as
the first CD is *not* rock: case 1 still returns the first rock CD, while case 2
returns nothing.

### Position and `//`

!!! warning "`//cd[1]` is not the first `cd` in the document"
    The predicate binds to its step, not to the whole path. In `//cd[1]` the
    `[1]` applies to the `cd` step, selecting **every `cd` that is the first
    `cd` among its own siblings** — possibly several nodes. To take the single
    global first node, parenthesise the path so the predicate applies to the
    *combined* result:

    ```
    (//cd)[1]     # the one first cd in the whole document
    //cd[1]       # each cd that is its parent's first cd
    ```

    See [XPath predicates](../xslt/predicates.md) for a worked walk-through of
    the distinction.

## Next

[Functions and data types](functions-and-types.md) — once you can select and
filter nodes, the core function library lets you *compute* over them and convert
between XPath's four 1.0 types.
