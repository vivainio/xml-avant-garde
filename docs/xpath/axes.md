# Axes

Every location step in an XPath expression has three parts:

```
axis::node-test[predicate]
```

The **axis** sets the *direction* the step travels from the context node (down
to children, up to ancestors, sideways to siblings, …); the **node-test** says
*which kind* of node to keep along that axis; the optional **predicate**
filters what survives.

The paths you have written so far used the **abbreviated** syntax, where the
most common axes are implied. Every abbreviation expands to a full
`axis::node-test` form:

| Abbreviation | Full form |
| --- | --- |
| `cd` | `child::cd` |
| `@genre` | `attribute::genre` |
| `//` | `/descendant-or-self::node()/` |
| `..` | `parent::node()` |
| `.` | `self::node()` |

So `catalog/cd/@genre` is just a friendlier spelling of
`child::catalog/child::cd/attribute::genre`. The abbreviations cover the steps
you reach for daily; the explicit axes below are what you fall back to when they
do not.

## The running example

``` xml title="catalog.xml"
<?xml version="1.0" encoding="UTF-8"?>
<catalog>
  <cd genre="rock">
    <title>Empire Burlesque</title>
    <artist>Bob Dylan</artist>
    <price>10.90</price>
    <year>1985</year>
  </cd>
  <cd genre="pop">
    <title>Hide your heart</title>
    <artist>Bonnie Tyler</artist>
    <price>9.90</price>
  </cd>
  <cd genre="country">
    <title>Greatest Hits</title>
    <artist>Dolly Parton</artist>
    <price>9.90</price>
  </cd>
</catalog>
```

## Forward axes

Forward axes select nodes that come **at or after** the context node in
document order.

`child` — the immediate children (the default axis).

``` xpath
child::cd          <!-- same as: cd -->
```

<div class="xslt-result" markdown>
the three `<cd>` elements
</div>

`descendant` — children, grandchildren, and so on, but **not** the context node
itself.

``` xpath
descendant::title
```

<div class="xslt-result" markdown>
all three `<title>` elements, however deeply nested
</div>

`descendant-or-self` — the descendants plus the context node. This is the axis
hiding inside `//`.

``` xpath
descendant-or-self::node()
```

`following-sibling` — siblings that come after the context node, sharing the
same parent.

``` xpath
following-sibling::cd      <!-- from the first cd: the 2nd and 3rd -->
```

`following` — **every** node after the context node in document order, *except*
its own descendants (and excluding attribute and namespace nodes).

``` xpath
following::price           <!-- from the first title: all later prices -->
```

`attribute` — the attributes of the context element. This is what `@` abbreviates.

``` xpath
attribute::genre           <!-- same as: @genre -->
```

<div class="xslt-result" markdown>
`rock`, `pop`, `country`
</div>

`self` — the context node itself. This is what `.` abbreviates; it is most
useful for adding a node-test, as in `self::cd`.

``` xpath
self::node()               <!-- same as: . -->
```

`namespace` — the namespace nodes in scope on an element. Rarely used directly,
and dropped entirely in XPath 2.0+.

## Reverse axes

Reverse axes select nodes that come **at or before** the context node.

`parent` — the single parent node. This is what `..` abbreviates.

``` xpath
parent::node()             <!-- same as: .. -->
```

`ancestor` — the parent, grandparent, and so on up to the document root.

``` xpath
ancestor::catalog          <!-- from any title: the enclosing catalog -->
```

`ancestor-or-self` — the ancestors plus the context node.

``` xpath
ancestor-or-self::*
```

`preceding-sibling` — siblings that come before the context node, sharing the
same parent.

``` xpath
preceding-sibling::cd      <!-- from the third cd: the 1st and 2nd -->
```

`preceding` — every node before the context node in document order, *except*
its own ancestors (and excluding attribute and namespace nodes).

``` xpath
preceding::title           <!-- from the last price: all earlier titles -->
```

## `descendant` vs `following`, `child` vs `descendant`

Two distinctions trip people up.

`child` reaches **one level down**; `descendant` reaches **all levels down**.
For the flat catalog `child::title` from a `cd` finds its title, while
`descendant::title` from `catalog` finds *every* title in the document.

`descendant` stays **inside** the context node — only nodes it contains.
`following` looks **outside and after** it — everything later in the document
that is not one of its descendants. Standing on the first `<cd>`:

- `descendant::*` → its own `title`, `artist`, `price`, `year`.
- `following::*` → the second and third `<cd>` elements and all *their*
  children, but nothing inside the first `<cd>`.

The two axes never overlap: together with `ancestor`, `preceding`, and `self`
they partition the whole document.

## When you actually need explicit axes

The abbreviations handle "down" and "up one level". You write an explicit axis
when you need to move sideways, reach far up, or look backward.

``` xpath
following-sibling::cd[1]    <!-- the very next cd after this one -->
ancestor::catalog          <!-- the catalog this node lives in -->
preceding-sibling::*       <!-- every earlier sibling element -->
```

`following-sibling::cd[1]` is the idiomatic "next record" navigation: from one
`<cd>` it lands on the one immediately after. There is no abbreviation for any
of these — `..` only climbs one step, and `//` only descends.

!!! warning "Reverse axes number positions in reverse"
    On a **reverse** axis, a positional predicate counts outward from the
    context node, not in document order. So `preceding-sibling::cd[1]` is the
    *nearest* preceding sibling — the one **just before** the context node — and
    `ancestor::*[1]` is the immediate parent, the closest ancestor. This is the
    opposite of `following-sibling::cd[1]` (a forward axis), where `[1]` is the
    nearest *following* sibling. The rule: `[1]` always means "closest to the
    context node", which on a reverse axis is the document-*last* of the
    matches, not the document-first.

## Summary

| Axis | Direction | Selects | Abbreviation |
| --- | --- | --- | --- |
| `child` | forward | immediate children | *(none — the default)* |
| `descendant` | forward | all nodes below, excluding self | |
| `descendant-or-self` | forward | descendants plus self | `//` (as `/descendant-or-self::node()/`) |
| `following-sibling` | forward | later siblings | |
| `following` | forward | everything after, excluding descendants | |
| `attribute` | forward | the element's attributes | `@` |
| `self` | — | the context node | `.` |
| `namespace` | forward | in-scope namespace nodes | |
| `parent` | reverse | the single parent | `..` |
| `ancestor` | reverse | all nodes above | |
| `ancestor-or-self` | reverse | ancestors plus self | |
| `preceding-sibling` | reverse | earlier siblings | |
| `preceding` | reverse | everything before, excluding ancestors | |

!!! note
    `self`, `descendant-or-self`, and `ancestor-or-self` include the context
    node, so they are neither purely forward nor purely reverse.

## Next

A step is more than its axis: the node-test decides which nodes survive, and the
predicate filters them — see [Node tests and predicates](node-tests-predicates.md).
