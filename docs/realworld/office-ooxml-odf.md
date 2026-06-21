---
icon: lucide/file-text
---

# Office documents — a ZIP of namespaced parts

The `.docx` on your disk and the `.odt` next to it are not single XML files. Each
is a **ZIP archive** of many XML *parts*, wired together by relationship files.
This page looks at both big families — Microsoft's **OOXML** (`.docx`, `.xlsx`,
`.pptx`) and OASIS's **OpenDocument / ODF** (`.odt`, `.ods`, `.odp`) — because
together they show what happens when a vocabulary gets *large*: dozens of
namespaces, split across files, referencing each other.

!!! info "Look inside one yourself"
    A `.docx` is just a ZIP. `unzip -l report.docx` lists the parts;
    `[Content_Types].xml`, `word/document.xml`, `word/styles.xml`, and
    `word/_rels/document.xml.rels` are the load-bearing ones. ODF is the same idea:
    `content.xml`, `styles.xml`, `meta.xml`, `META-INF/manifest.xml`.

## OOXML: the main document part

`word/document.xml` holds the text. WordprocessingML's vocabulary lives almost
entirely under one prefix, `w:`, with `r:` (relationships) reaching *out* of the
part to images and hyperlinks.

``` xml title="word/document.xml" linenums="1"
<w:document xmlns:w="http://schemas.openxmlformats.org/wordprocessingml/2006/main"
            xmlns:r="http://schemas.openxmlformats.org/officeDocument/2006/relationships">
  <w:body>
    <w:p>                                          <!-- (1)! -->
      <w:pPr><w:pStyle w:val="Heading1"/></w:pPr>
      <w:r><w:t>Quarterly report</w:t></w:r>
    </w:p>
    <w:p>
      <w:r>                                        <!-- (2)! -->
        <w:rPr><w:b/></w:rPr>
        <w:t xml:space="preserve">Bold </w:t>
      </w:r>
      <w:hyperlink r:id="rId4"><w:r><w:t>link</w:t></w:r></w:hyperlink>  <!-- (3)! -->
    </w:p>
  </w:body>
</w:document>
```

1.  `w:p` is a paragraph; `w:pPr` is its *properties* (paragraph style, spacing).
    OOXML's pattern is rigid: a content element, optionally preceded by a
    `…Pr` properties sibling. Once you see it, the whole format reads.
2.  `w:r` is a *run* — a span of text with uniform formatting — and `w:rPr` is its
    run properties (`w:b` = bold). Text always lives inside a `w:t`.
3.  `r:id="rId4"` does **not** contain the URL. It is a *relationship id* that
    points into a separate part, `word/_rels/document.xml.rels`, where `rId4` maps
    to an actual target. The `r:` namespace exists purely to express these
    cross-part links.

Note `xml:space="preserve"` on the bold run: the `xml:` prefix is the one
namespace *every* XML document gets for free (bound to
`http://www.w3.org/XML/1998/namespace`), used here so the trailing space in
`"Bold "` survives.

## The relationships part

The id from `r:id="rId4"` is resolved here. This is OOXML's answer to "how does
one XML part point at another file in the package":

``` xml title="word/_rels/document.xml.rels"
<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">
  <Relationship Id="rId4" Type="http://…/hyperlink"
                Target="https://example.org" TargetMode="External"/>
  <Relationship Id="rId5" Type="http://…/image"
                Target="media/logo.png"/>
</Relationships>
```

The indirection (`rId4` → real target) keeps `document.xml` stable when targets
move, and lets binary assets (images) live as separate package members rather
than being base64-stuffed into the markup.

## ODF: the same problem, OASIS's answer

OpenDocument splits content the same way but makes a *different* namespace choice:
instead of one big `w:`, it uses **many** small, semantic namespaces — `office:`,
`text:`, `table:`, `style:`, `draw:`, `fo:` — so each prefix names a *domain*.

``` xml title="content.xml" linenums="1"
<office:document-content
    xmlns:office="urn:oasis:names:tc:opendocument:xmlns:office:1.0"
    xmlns:text="urn:oasis:names:tc:opendocument:xmlns:text:1.0"
    xmlns:table="urn:oasis:names:tc:opendocument:xmlns:table:1.0"
    xmlns:style="urn:oasis:names:tc:opendocument:xmlns:style:1.0"
    office:version="1.3">
  <office:body>
    <office:text>
      <text:h text:style-name="Heading_20_1" text:outline-level="1">Report</text:h>
      <text:p text:style-name="Standard">Hello
        <text:span text:style-name="Bold">world</text:span>.</text:p>
    </office:text>
  </office:body>
</office:document-content>
```

!!! tip "Two philosophies, same data model"
    OOXML funnels almost everything through `w:`/`x:`/`p:` (one prefix per *app*:
    Word, Excel, PowerPoint). ODF spreads it across `text:`/`table:`/`style:` (one
    prefix per *concept*). Neither is wrong — they are two legitimate ways to carve
    up a large vocabulary with namespaces, and seeing them side by side is the
    lesson. ODF even borrows the XSL-FO `fo:` namespace
    ([next-but-one page](xsl-fo-fop.md)) for formatting properties rather than
    inventing its own.

## Querying Office XML

Both formats are heavily namespaced, so — exactly as with
[SVG](svg.md#querying-namespaced-svg-with-xpath) — `//p` finds no paragraphs.
You must bind the prefixes in your [XPath](../xpath/index.md) host and query
`//w:p` or `//text:p`. The wrinkle that surprises people: the prefix you use in
*your* query need not match the document's; only the **namespace URI** matters.
A document could declare `xmlns:foo="…wordprocessingml…"` and `//foo:p` would
still be wrong unless *you* bound `foo` to that URI too.

## Things to note

- A "file" can be a **package** of XML parts; the interesting structure is *across*
  parts, not within one.
- **Relationship indirection** (`r:id` → `.rels`) keeps markup stable and keeps
  binary assets out of the XML.
- A large vocabulary can be organized **one-prefix-per-app** (OOXML) or
  **one-prefix-per-concept** (ODF) — two namespace strategies, with different
  trade-offs.
- `xml:space` and `xml:lang` ride the always-available `xml:` namespace.

Next: [Atom and feed extensions](atom-feeds.md), where the lesson flips — instead
of one huge vocabulary, a tiny core that everyone *extends*.
