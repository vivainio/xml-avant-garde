---
icon: lucide/blocks
---

# Constructing XML precisely

XQuery's literal XML syntax makes simple output pleasantly direct:

``` xquery
<album title="Empire Burlesque"/>
```

The details matter once names are dynamic, existing nodes are inserted, or
attributes and namespaces are assembled conditionally. Constructors do more than
print markup: they create new XDM nodes with identity, type annotations,
namespaces, and content-normalization rules.

## Direct constructors: XML with expression holes

A **direct constructor** looks like the XML it creates. Enclosed expressions in
`{ ... }` compute content and attribute values:

``` xquery
for $cd in /catalog/cd
return
  <album genre="{ $cd/@genre }">
    <title>{ string($cd/title) }</title>
    <artist>{ string($cd/artist) }</artist>
  </album>
```

The XML parts obey XML syntax; the expression holes obey XQuery syntax. To output
a literal brace in constructor text or an attribute, double it as `{{` or `}}`.

Inside element content, the result of an enclosed expression is a sequence:

- nodes are copied into the new element;
- atomic values become text;
- adjacent atomic values are separated by spaces;
- empty sequences add nothing;
- adjacent text nodes are merged.

That space rule is easy to miss:

``` xquery
<values>{ ("rock", "pop", 1985) }</values>
```

``` xml title="result"
<values>rock pop 1985</values>
```

Use `string-join($values, ", ")` when punctuation rather than spaces should
separate the values.

## Attribute value templates become strings

In a direct attribute, the enclosed expression is atomized and joined with
spaces:

``` xquery
let $genres := ("rock", "pop")
return <filter genres="{ $genres }"/>
```

``` xml title="result"
<filter genres="rock pop"/>
```

An empty sequence produces an empty attribute, not no attribute:

``` xquery
<album year="{ () }"/>       (: <album year=""/> :)
```

To omit an optional attribute, construct it conditionally:

``` xquery
element album {
  if ($cd/year) then attribute year { $cd/year } else (),
  attribute title { $cd/title }
}
```

## Computed constructors: when the shape is dynamic

Use computed constructors when an element or attribute name is calculated, or
when attributes are conditional:

``` xquery
let $name := "album"
return
  element { $name } {
    attribute genre { "rock" },
    element title { "Empire Burlesque" }
  }
```

The available constructors are:

``` xquery
element { $name } { $content }
attribute { $name } { $value }
text { $value }
comment { $value }
processing-instruction { $target } { $value }
document { $content }
namespace { $prefix } { $uri }       (: XQuery 3.0+ :)
```

With no default element namespace, an unprefixed string name creates a
no-namespace element. For a namespace-aware dynamic name, construct a QName:

``` xquery
declare namespace r = "https://example.org/report";

element { QName("https://example.org/report", "r:album") } {
  attribute id { "a-1" }
}
```

The namespace URI—not the prefix—is the element's identity. The prefix is a
serialization choice, though supplying a useful one keeps output readable.

## Attributes must come first

In a computed element, every attribute and namespace node must be constructed
before child content:

``` xquery
element album {
  attribute id { "a-1" },
  element title { "Empire Burlesque" }
}
```

Reversing those two items is an error: once child content has begun, XQuery
cannot go back and attach an attribute. Constructing two attributes with the same
expanded name is also an error.

A clean pattern is to put all conditional attributes at the top:

``` xquery
element album {
  attribute title { $cd/title },
  if ($cd/year) then attribute year { $cd/year } else (),
  if ($cd/@genre) then attribute genre { $cd/@genre } else (),
  $cd/artist
}
```

## Inserting a node copies it

Constructors do not move an existing node under a new parent; they create a copy:

``` xquery
let $original := /catalog/cd[1]/title
let $wrapper := <result>{ $original }</result>
let $copy := $wrapper/title
return (
  $original = $copy,       (: true: same string value :)
  $original is $copy       (: false: different node identity :)
)
```

The source tree is immutable and unchanged. This is why a constructed result can
be returned independently of its input, and why identity-sensitive code must not
expect an inserted node to remain `is`-identical.

If you only want the value, make that explicit with `string($original)` or a
`text` constructor. If you want its markup and descendants, insert the node.

## Preserve markup or extract text?

These expressions look similar but create different results:

``` xquery
<result>{ $cd/title }</result>                 (: copies <title>...</title> :)
<result>{ string($cd/title) }</result>         (: inserts only its text :)
<result title="{ $cd/title }"/>                (: string in an attribute :)
```

Choose based on the result vocabulary. Accidentally copying a source element is a
common cause of unwanted wrapper elements and source namespaces in output.

## Namespaces in direct constructors

Direct constructors use ordinary XML namespace rules:

``` xquery
declare namespace r = "https://example.org/report";

<r:report>
  <r:album title="Empire Burlesque"/>
</r:report>
```

You can also declare namespaces locally:

``` xquery
<report xmlns="https://example.org/report">
  <album/>
</report>
```

The default element namespace applies to unprefixed element names, but never to
unprefixed attributes. This is ordinary XML behavior:

``` xml
<album xmlns="https://example.org/report" id="a-1"/>
```

`album` is in the report namespace; `id` is in no namespace.

For query source paths, `declare default element namespace` controls how
unprefixed element tests are interpreted:

``` xquery
declare default element namespace
  "urn:oasis:names:specification:ubl:schema:xsd:Invoice-2";
```

Use that carefully: it affects unprefixed names in both paths and constructors.
Explicit prefixes are often clearer when source and result vocabularies differ.

## Copying namespaces

When source elements are copied, their in-scope namespaces can travel with them.
The declaration:

``` xquery
declare copy-namespaces no-preserve, no-inherit;
```

controls whether copied nodes retain unused namespace bindings and whether a
constructed element inherits bindings from its new parent. The default is
`preserve, inherit`, which is safe but can leave extra declarations in results.

Reach for `no-preserve, no-inherit` when producing a tightly controlled result
vocabulary; then explicitly construct every namespace the output needs. Namespace
cleanup changes declarations and prefixes, not the expanded names of elements and
attributes.

## Boundary whitespace

Whitespace used only to lay out a direct constructor is **boundary whitespace**:

``` xquery
<album>
  <title>Empire Burlesque</title>
</album>
```

By default XQuery strips boundary whitespace, so the indentation around `title`
does not necessarily become text nodes. Make the policy explicit when it matters:

``` xquery
declare boundary-space strip;
```

or:

``` xquery
declare boundary-space preserve;
```

This affects whitespace written in the query's direct constructors, not whitespace
inside source nodes inserted by an expression. Serialization indentation is a
separate final step; it changes textual presentation, not the logical result tree.

## Computed comments and processing instructions

Comments cannot contain `--` or end in `-`, and processing-instruction content
cannot contain `?>`. The constructors validate those XML constraints:

``` xquery
comment { concat("generated ", current-dateTime()) },
processing-instruction xml-stylesheet {
  'type="text/xsl" href="report.xsl"'
}
```

If dynamic content can contain forbidden character sequences, sanitize or reject
it before constructing the node rather than trying to fix serialized XML later.

## Build a complete document

An element is not the same item as a document node. Most result serialization
accepts either, but a document constructor is useful when the result should model
a complete XML document:

``` xquery
document {
  processing-instruction xml-stylesheet {
    'type="text/xsl" href="catalog.xsl"'
  },
  <catalog-report generated="{ current-date() }">{
    for $cd in /catalog/cd
    order by $cd/title
    return
      <album title="{ $cd/title }"
             artist="{ $cd/artist }"
             price="{ xs:decimal($cd/price) }"/>
  }</catalog-report>
}
```

A document node may contain one element child plus comments and processing
instructions around it, but not stray non-whitespace text or several top-level
elements.

## A construction checklist

When output is surprising, ask:

1. Did the enclosed expression return nodes or atomic values?
2. Should a source element be copied, or only its string value?
3. Can an optional attribute be absent, rather than present as `""`?
4. Were all computed attributes constructed before children?
5. Which namespace URI does each constructed name use?
6. Is whitespace part of the XDM tree, or only serializer indentation?
7. Does later code rely on node identity after a constructor copied the node?

Those questions account for most constructor bugs.

## Where to go next

- [Functions, modules, and error handling](functions-modules-errors.md) turns
  query fragments into reusable, typed programs.
- [XQuery vs XSLT](xquery-vs-xslt.md) compares constructors and FLWOR with
  template-driven result construction.
