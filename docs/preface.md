---
icon: lucide/book-open
---

# Why this book

I came back to XML in 2026. That it was still everywhere was no surprise —
Office documents are zipped XML, SVG is XML, plenty of config is XML, and I'd
always known it. What did surprise me was **XSLT**. I'd filed it under "the
2000s," a thing you used once and moved on from. Instead I found it alive and
genuinely useful — driving e-invoicing pipelines and transforming documents at
scale. Official European standards bodies ship XSLT: the EN16931 validation
artefacts come *as* stylesheets, and you are expected to run and handle them.
And the material for someone *returning* to all this is rough going. Most of
what's online is either a tedious from-zero tutorial that spends three screens
on what a tag is, a spec page written for compiler authors, or a 2004 blog post
half of whose advice is now obsolete — verbose, dated, and aimed at someone
who's never seen any of this before.

There's a reputation to address, too. XML, the popular imagination holds, is the
domain of fifty-year-old Java developers who listen to Dire Straits and remember
when SOAP was the future. And — fine — that's largely true. But it turns out the
door is unlocked, the music is actually pretty good, and you can still join. No
one will check your age at the angle bracket.

This is the manual I wanted instead. It assumes you can program and you've seen
angle brackets before, so it skips the throat-clearing and shows you what's
actually going on. I don't expect you to follow along at a keyboard,
copy-pasting as you go. I expect you to crash on the couch with a tablet and
read it — every concept carries a small, complete example precisely so you can
see how a thing works without having to set anything up to believe it.

The book is built to be jumped into. Each section stands on its own: start with
the [XSLT tutorial](xslt/index.md) if you want a guided path, or go straight to
[real-world e-invoicing](einvoicing/index.md) if that's the problem in front of
you. XPath, XQuery, XSD, and Schematron are there when you need the underlying
machinery.

The [real-world tour](realworld/index.md) deserves a word of warning. Some of
what it covers — SOAP, XSL-FO, a few others — is more of historical interest
than state of the art, and I don't pretend otherwise; you may never write a line
of it. They earn their place because they're where the *culture* of XML shows
itself most clearly: how the people who built these things thought about
namespaces, extension, validation, and versioning. Read them less as "tools you
need today" and more as field notes on a way of thinking that still runs under a
lot of what you do.

A note on how it was made: I wrote it with **Claude Code**. That's part of why
it can be both broad and current — covering the whole processing stack and the
places XML actually shows up today — while keeping every example real rather
than illustrative. It's built with [Zensical](https://zensical.org) and
published on GitHub Pages. It's meant to be read, not consulted — written by
someone who was standing exactly where you are.

[Start the XSLT tutorial :material-arrow-right:](xslt/index.md){ .md-button .md-button--primary }
