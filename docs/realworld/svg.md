---
icon: lucide/shapes
---

# SVG — a default namespace, a borrowed prefix, and mixing

**Scalable Vector Graphics** is the vocabulary your browser draws. It is the
gentlest possible introduction to real-world namespaces because you can *see* the
result, and because it shows three patterns in one small file:

1. a **default** namespace (the SVG elements carry no prefix),
2. a **borrowed** namespace (`xlink:` from another spec, for linking), and
3. **mixing** — an SVG fragment dropped straight into an HTML page.

## A small logo

``` xml title="logo.svg" linenums="1"
<svg xmlns="http://www.w3.org/2000/svg"
     xmlns:xlink="http://www.w3.org/1999/xlink"
     viewBox="0 0 120 60" width="120" height="60">
  <title>Avant logo</title>
  <defs>
    <linearGradient id="g" x1="0" y1="0" x2="1" y2="0">
      <stop offset="0%" stop-color="#4f46e5"/>
      <stop offset="100%" stop-color="#06b6d4"/>
    </linearGradient>
  </defs>
  <rect x="0" y="0" width="120" height="60" rx="8" fill="url(#g)"/>      <!-- (1)! -->
  <use xlink:href="#g"/>                                                <!-- (2)! -->
  <a xlink:href="https://example.org">                                  <!-- (3)! -->
    <text x="12" y="38" font-size="20" fill="white">XML</text>
  </a>
</svg>
```

1.  `rect`, `linearGradient`, `text` and the rest carry **no prefix** — they live
    in the *default* namespace declared by `xmlns="http://www.w3.org/2000/svg"` on
    the root. Every descendant inherits it.
2.  `xlink:href` is **not** an SVG attribute. `xlink` is a separate W3C spec
    (XLink) for expressing links; SVG 1.1 borrows it. `url(#g)` above is an
    *internal* reference written CSS-style; `xlink:href="#g"` is the XML way.
3.  `<a>` here is the **SVG** anchor element (in the SVG namespace), not the HTML
    one — same local name, different namespace. This is exactly why namespaces
    exist.

!!! note "Two ways to reference, one document"
    Notice the file uses *both* `fill="url(#g)"` (a functional IRI reference, the
    CSS heritage) and `xlink:href="#g"` (the XML heritage). SVG is a meeting point
    of the web's two cultures, and its attribute set shows the seams.

## The schema side

SVG was historically defined by a **DTD**, and later RELAX NG — not primarily
XSD — but XSD renderings exist and are useful for seeing the shape. Here is a
representative fragment for `<rect>`, rendered with `unxml --xsd`:

``` text title="unxml --xsd svg.xsd (excerpt)"
schema http://www.w3.org/2000/svg (elementFormDefault=qualified)
  xmlns = http://www.w3.org/2000/svg
  ns xlink = http://www.w3.org/1999/xlink
  import http://www.w3.org/1999/xlink from xlink.xsd          # (1)!
  type LengthType
    restriction xs:string
      pattern [-+]?[0-9]*\.?[0-9]+(px|pt|em|%)?
  attributeGroup PresentationAttrs                            # (2)!
    @fill : xs:string
    @stroke : xs:string
  element rect
    @x : LengthType
    @y : LengthType
    @width : LengthType (required)
    @height : LengthType (required)
    attributeGroup ref PresentationAttrs
    @ref xlink:href                                           # (3)!
```

1.  The schema [`import`s](../xsd/modular-schemas.md) the XLink namespace from a
    separate schema document — the XSD-level counterpart of the `xmlns:xlink`
    declaration in the instance.
2.  Presentation properties (`fill`, `stroke`, …) are bundled into an
    **attribute group** and reused across dozens of element types — the same DRY
    mechanism you saw in [Modular schemas](../xsd/modular-schemas.md).
3.  `@ref xlink:href` pulls in an attribute *declared in another namespace*. The
    schema can only do this because it imported that namespace above.

If `LengthType`'s pattern looks familiar, it is the
[regular-expression facet](../xsd/simple-types-restrictions.md) from the XSD
chapter, doing real work: constraining `width="120"` and `width="50%"` while
rejecting `width="wide"`.

## Mixing: SVG inside HTML

The payoff. Modern HTML lets you write SVG **inline**, with no `xmlns` at all:

``` xml title="page.html (fragment)"
<p>Our logo: <svg viewBox="0 0 16 16" width="16" height="16">
  <circle cx="8" cy="8" r="7" fill="indigo"/>
</svg> looks like this.</p>
```

In an HTML *document*, the parser knows that `<svg>` switches the children into
the SVG namespace automatically — the namespace is implied by the element name.
But the moment you treat the same file as **XML** (XHTML, or any XML toolchain),
that magic is gone and you must declare the namespace explicitly, because an XML
parser has no special knowledge of "svg". This split — implied in the HTML
parser, explicit in XML — is the single most common source of "it renders in the
browser but my [XPath](../xpath/index.md) finds nothing" confusion.

### Querying namespaced SVG with XPath

!!! warning "An unprefixed name means *no* namespace"
    `//circle` matches **nothing** in a namespaced SVG document — `circle` is in
    the SVG namespace, and an unprefixed name in XPath 1.0 means "no namespace".
    You must bind a prefix (say `s`) to `http://www.w3.org/2000/svg` and ask for
    `//s:circle`. This is the same rule the
    [XPath chapter](../xpath/node-tests-predicates.md) introduced, biting in a
    real document.

## Things to note

- A **default namespace** lets a whole subtree go prefix-free — convenient for
  authoring, but every name is still *in* that namespace.
- A vocabulary can **borrow** another (`xlink`) instead of reinventing linking.
- The **same local name** (`<a>`) means different things in different namespaces
  — the original motivation for the whole mechanism.
- "It works in the browser" relies on the HTML parser's namespace shortcuts; XML
  tools need the declarations spelled out.

Next: [SOAP and WSDL](soap-wsdl.md), where namespaces stop being convenient and
become a *contract* between machines.
