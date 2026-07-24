# Your first transformation

The smallest useful stylesheet matches the document root and emits some HTML.
This one turns the [CD catalog](index.md#the-running-example) into a table.

``` xml title="cdcatalog.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
<xsl:template match="/">                              <!-- (1)! -->
<html>
<body>
  <h2>My CD Collection</h2>
  <table border="1">
    <tr bgcolor="#9acd32">
      <th style="text-align:left">Title</th>
      <th style="text-align:left">Artist</th>
    </tr>
    <xsl:for-each select="catalog/cd">                <!-- (2)! -->
    <tr>
      <td><xsl:value-of select="title"/></td>         <!-- (3)! -->
      <td><xsl:value-of select="artist"/></td>
    </tr>
    </xsl:for-each>
  </table>
</body>
</html>
</xsl:template>
</xsl:stylesheet>
```

1.  `match="/"` matches the **document root** — the rule fires once, for the
    whole document. Everything inside is the template for what to output.
2.  `xsl:for-each` loops over every `<cd>` element under `<catalog>`.
3.  `xsl:value-of` pulls out the text of a child element and writes it into the
    result.

## Try it

Keep the stylesheet above as `cdcatalog.xsl` and the
[running example](index.md#the-running-example) as `catalog.xml` in the same
directory. There are two useful ways to run it.

### In a browser (while supported)

Browsers historically shipped an XSLT 1.0 processor. In a browser that still
has it enabled, add this processing instruction immediately after the XML
declaration in `catalog.xml`:

``` xml
<?xml-stylesheet type="text/xsl" href="cdcatalog.xsl"?>
```

The beginning of the source file should now look like this:

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="cdcatalog.xsl"?>
<catalog>
  ...
</catalog>
```

Open `catalog.xml` in the browser and it will apply the referenced stylesheet
before displaying the result. This route is useful for seeing the original
client-side XSLT model, but do not build a new workflow around it:
[Chromium has deprecated native XSLT and plans to remove it from stable Chrome
in late 2026](https://developer.chrome.com/docs/web-platform/deprecating-xslt);
Firefox and WebKit have also indicated removal plans.

Some browsers also restrict related files loaded directly from disk. If that
happens, serve the directory through a small local web server and open the HTTP
URL instead; the stylesheet and XML themselves do not need to change. If the
browser has disabled XSLT entirely, use Saxon below.

### With Saxon

A command-line processor is better once transformations become part of a build,
need diagnostics, or use XSLT 2.0/3.0. With a Saxon JAR, the usual command shape
is:

``` sh
java -jar Saxon-HE.jar \
  -s:catalog.xml \
  -xsl:cdcatalog.xsl \
  -o:catalog.html
```

Here `-s` names the source, `-xsl` the stylesheet, and `-o` the result file.
Open `catalog.html` to see the same table. Saxon distributions and package
managers may provide a wrapper command instead of a JAR, but the three inputs
are the same: source, stylesheet, and output.

!!! tip "Keep the first experiment literal"
    Run the example unchanged once before editing it. Then change one thing at a
    time: add the price column, alter the heading, or filter the loop. A blank
    result is much easier to diagnose when you know which single edit caused it.

## Anatomy of a stylesheet

A stylesheet is itself a well-formed XML document.

- The root element is `xsl:stylesheet` (or its synonym `xsl:transform`).
- The `xsl:` prefix is bound to the namespace
  `http://www.w3.org/1999/XSL/Transform`. The processor uses this namespace to
  tell *instructions* (things it should execute) apart from *literal result
  elements* (things it should copy to the output, like `<html>` and `<table>`).
- `version="1.0"` selects the XSLT version. It is the broadest portability
  baseline across dedicated processors. Versions 2.0 and 3.0 add more features
  and need a processor like Saxon; native browser processing historically
  stopped at 1.0 and is now being phased out.

## The result

Applied to the catalog, this produces an HTML table:

<div class="xslt-result" markdown>
**My CD Collection**

| Title | Artist |
| --- | --- |
| Empire Burlesque | Bob Dylan |
| Hide your heart | Bonnie Tyler |
| Greatest Hits | Dolly Parton |
</div>

## Next

The single big template works, but it does not scale: as documents grow you end
up with one giant rule. [Templates](templates.md) show how to break the work
into small, composable pieces.
