---
icon: lucide/rss
---

# Atom and feed extensions — extending a base vocabulary

The previous pages showed *big* vocabularies. **Atom** is the opposite: a small,
fixed core for syndication (`feed`, `entry`, `title`, `link`, …) that is designed
to be **extended**. This is the namespace pattern that powers podcasts, blog
metadata, and most of the structured web's "and also attach *this*" — adding
foreign elements into a document *without* changing the base vocabulary or
breaking readers that do not understand them.

## A feed with two extensions

``` xml title="feed.xml" linenums="1"
<feed xmlns="http://www.w3.org/2005/Atom"
      xmlns:dc="http://purl.org/dc/elements/1.1/"
      xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd">
  <title>XML Avant-Garde</title>
  <link href="https://example.org/" rel="alternate"/>          <!-- (1)! -->
  <link href="https://example.org/feed.xml" rel="self"/>
  <updated>2026-06-20T10:00:00Z</updated>
  <id>urn:uuid:60a76c80-d399-11d9-b93C-0003939e0af6</id>        <!-- (2)! -->
  <itunes:author>Ville</itunes:author>                         <!-- (3)! -->
  <entry>
    <title>Namespaces in practice</title>
    <id>https://example.org/posts/1</id>
    <updated>2026-06-20T10:00:00Z</updated>
    <dc:creator>Ville Vainio</dc:creator>                      <!-- (4)! -->
    <content type="html">&lt;p&gt;Hi&lt;/p&gt;</content>       <!-- (5)! -->
  </entry>
</feed>
```

1.  Atom's `link` is *typed* by `rel`: `alternate` is the human page, `self` is
    the feed's own URL. The whole feed is in the **default** Atom namespace — no
    prefix on `feed`, `title`, `entry`.
2.  Every Atom `id` must be a permanent, globally-unique IRI — here a `urn:uuid`.
    Readers de-duplicate entries by this, not by title.
3.  `itunes:author` is from Apple's **podcast** namespace. An Atom reader that has
    never heard of podcasts simply ignores it — it is in a namespace the reader
    does not process. That graceful ignoring is the entire point.
4.  `dc:creator` is **Dublin Core**, a tiny, decades-old metadata vocabulary
    (`creator`, `date`, `subject`, `rights`, …) that shows up *everywhere* —
    feeds, [Office documents](office-ooxml-odf.md), repositories. It is the
    canonical "borrowed" namespace.
5.  `content type="html"` carries escaped HTML as text. (It could instead be
    `type="xhtml"` and contain real XHTML elements — yet another namespace nested
    inside.)

The layering is the whole story: bare names (`title`, `entry`, `link`) are core
Atom; the `dc:` and `itunes:` prefixes are bolt-ons. A reader walks the tree and
skips any prefix it does not recognize.

## Why this is "extension", not "a bigger schema"

The crucial property: the Atom **schema never changed**. Apple did not get to
edit the Atom spec to add podcast tags; they minted *their own* namespace and
Atom's content model allows foreign-namespaced children. In XSD terms, the
extension point is a wildcard:

``` text title="unxml --xsd atom.xsd (the extension point)"
type atomFeed
  ...
  any ##other * (lax)            # (1)!
```

1.  `##other` means "elements from **any namespace except** the target (Atom)
    one"; `*` is zero-or-more; `lax` says "validate them if you have a schema for
    that namespace, otherwise let them pass". This single
    [wildcard](../xsd/complex-types.md) is what turns a closed vocabulary into an
    open, extensible one. Compare it with the
    [validation pipeline](../einvoicing/validation-pipeline.md): same idea, opposite
    policy — EN16931 *forbids* extensions where Atom *invites* them.

## RSS: the same job, no namespace on the core

Atom's older cousin, **RSS 2.0**, makes an instructive contrast: its *core*
elements (`channel`, `item`, `title`) are in **no namespace at all** — a design
from before namespaces were universal. Yet RSS still grew the same extension
ecosystem, purely through namespaced add-ons:

``` xml title="rss.xml (fragment)"
<rss version="2.0"
     xmlns:content="http://purl.org/rss/1.0/modules/content/"
     xmlns:podcast="https://podcastindex.org/namespace/1.0">
  <channel>
    <title>No namespace here</title>                <!-- core RSS: unqualified -->
    <item>
      <title>An episode</title>
      <content:encoded><![CDATA[<p>Full HTML body</p>]]></content:encoded>
      <podcast:transcript url="t.vtt" type="text/vtt"/>
    </item>
  </channel>
</rss>
```

So a single RSS document mixes *unqualified* core elements with *qualified*
extension elements — a living reminder that "no namespace" is itself a namespace
state, and that you can layer qualified vocabularies on top of an unqualified
base.

!!! note "Atom vs RSS in one line"
    Atom puts its **whole core in a namespace** and versions by URI (like
    [SOAP](soap-wsdl.md#two-soap-namespaces-in-the-wild)); RSS leaves its core
    **unqualified** and bolts extensions on. Both end up extensible — the route
    differs.

## Things to note

- A small, stable core plus a **wildcard extension point** beats one ever-growing
  schema — third parties extend on their own namespaces and timelines.
- Unknown namespaces are **safely ignored**, which is what makes the web's feeds
  forward-compatible.
- **Dublin Core** is the archetypal borrowed vocabulary, and a common one to meet.
- "No namespace" (RSS core) and "everything namespaced" (Atom core) are both
  workable starting points.

Next: [XSL-FO and Apache FOP](xsl-fo-fop.md) — a vocabulary you almost never type
by hand, because [XSLT](../xslt/index.md) *generates* it.
