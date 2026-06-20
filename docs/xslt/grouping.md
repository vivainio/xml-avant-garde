# Grouping

A recurring task is to take a flat list and reorganise it under headings — all
the rock CDs together, then all the pop CDs. XSLT 1.0 had no grouping construct,
so people reached for **Muenchian grouping**: declare an
[`xsl:key`](keys.md#keys-for-grouping-muenchian), then select the first node in
each key bucket with a `generate-id()` comparison. It worked, but it was famously
opaque.

XSLT 2.0 replaced the whole trick with one instruction, `xsl:for-each-group`.

For these examples the [catalog](index.md#the-running-example) carries a `genre`
attribute on each CD:

``` xml title="catalog.xml" linenums="1"
<catalog>
  <cd genre="rock"><title>Empire Burlesque</title><artist>Bob Dylan</artist><price>10.90</price></cd>
  <cd genre="pop"><title>Hide your heart</title><artist>Bonnie Tyler</artist><price>9.90</price></cd>
  <cd genre="rock"><title>Blood on the Tracks</title><artist>Bob Dylan</artist><price>11.50</price></cd>
  <cd genre="pop"><title>Faster Than the Speed of Night</title><artist>Bonnie Tyler</artist><price>8.50</price></cd>
</catalog>
```

## Grouping by key

`xsl:for-each-group` iterates once per **group**. The `select` attribute chooses
the population of nodes; `group-by` computes a key for each one, and nodes that
share a key land in the same group.

``` xml linenums="1"
<xsl:for-each-group select="catalog/cd" group-by="@genre">   <!-- (1)! -->
  <h2><xsl:value-of select="current-grouping-key()"/></h2>    <!-- (2)! -->
  <xsl:for-each select="current-group()">                     <!-- (3)! -->
    <p><xsl:value-of select="title"/></p>
  </xsl:for-each>
</xsl:for-each-group>
```

1.  Group the CDs; the key is each one's `genre` attribute.
2.  Inside the body, `current-grouping-key()` is the key of the current group.
3.  `current-group()` is the sequence of members that share that key.

<div class="xslt-result" markdown>
**rock**

Empire Burlesque

Blood on the Tracks

**pop**

Hide your heart

Faster Than the Speed of Night
</div>

Note the two functions that are only meaningful inside the body:

| Function | Value within the group body |
| --- | --- |
| `current-grouping-key()` | the shared key of the current group |
| `current-group()` | the sequence of nodes belonging to it |

## Aggregating a group

Because `current-group()` is just a sequence, the usual aggregate functions
apply to it. This adds a count and a price total per genre:

``` xml linenums="1"
<xsl:for-each-group select="catalog/cd" group-by="@genre">
  <h2>
    <xsl:value-of select="current-grouping-key()"/>
    (<xsl:value-of select="count(current-group())"/> CDs,
     total <xsl:value-of select="sum(current-group()/price)"/>)   <!-- (1)! -->
  </h2>
</xsl:for-each-group>
```

1.  `current-group()/price` is the price of every member; `sum()` totals them.

<div class="xslt-result" markdown>
**rock** (2 CDs, total 22.4)

**pop** (2 CDs, total 18.4)
</div>

## Grouping adjacent runs

`group-by` collects matching nodes from across the *whole* population, no matter
how far apart they sit. `group-adjacent` instead starts a new group whenever the
key changes between **consecutive** items — it groups *runs*.

This matters when the input is already arranged and you want to fold together
neighbouring duplicates. Against the catalog above, where genres alternate,
`group-adjacent` produces four singleton groups rather than two:

``` xml linenums="1"
<xsl:for-each-group select="catalog/cd" group-adjacent="@genre">   <!-- (1)! -->
  <p>
    <xsl:value-of select="current-grouping-key()"/>:
    <xsl:value-of select="count(current-group())"/>
  </p>
</xsl:for-each-group>
```

1.  A new group begins each time the genre differs from the previous CD.

<div class="xslt-result" markdown>
rock: 1

pop: 1

rock: 1

pop: 1
</div>

!!! note "group-by vs group-adjacent"
    Use `group-by` to gather every node with a given key regardless of order;
    use `group-adjacent` to collapse consecutive runs that share a key. If the
    nodes are already sorted by the key, the two coincide.

## Grouping by position

Two further variants split the population by a **pattern** rather than a key,
which is handy for flat structures like a run of headings followed by content:

- `group-starting-with="..."` — a new group begins at each node that matches the
  pattern (the matching node leads its group).
- `group-ending-with="..."` — a new group ends at each node that matches the
  pattern (the matching node closes its group).

``` xml
<xsl:for-each-group select="*" group-starting-with="h2">
  ...
</xsl:for-each-group>
```

## Ordering the groups

Groups emerge in **order of first appearance** — the order in which each key was
first encountered in the population. To order them otherwise, drop an `xsl:sort`
in as the first child, exactly as in an ordinary loop. This emits the genres
alphabetically:

``` xml linenums="1"
<xsl:for-each-group select="catalog/cd" group-by="@genre">
  <xsl:sort select="current-grouping-key()"/>     <!-- (1)! -->
  <h2><xsl:value-of select="current-grouping-key()"/></h2>
</xsl:for-each-group>
```

1.  Sort the groups by their key before iterating.

<div class="xslt-result" markdown>
**pop**

**rock**
</div>

!!! tip "Sort the members too"
    The `xsl:sort` above orders the *groups*. To order the CDs *within* each
    group, add an `xsl:sort` inside the inner `xsl:for-each select="current-group()"`.
    See [Sorting](sorting.md) for the full set of sort keys and options.

## Next

Grouping keys are often derived from text — a prefix, a normalised case, a
substring. [Regular expressions and strings](regex.md) cover the matching and
tokenising tools that make those keys easy to compute.
