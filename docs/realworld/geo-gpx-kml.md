---
icon: lucide/map-pin
---

# GPX and KML — two geo vocabularies, two extension styles

Geospatial data is one of XML's quiet success stories: your watch records a run
as **GPX**, Google Earth saves places as **KML**. Both are small enough to read
whole, both are namespaced, and — the reason they share a page — they solve
"how do I add data the base spec didn't define?" in two *different* ways. Put
next to each other, they make the extension question concrete.

## GPX: a clean schema with an `<extensions>` hatch

GPX (the GPS Exchange Format) is defined by a real, compact
[XSD](../xsd/index.md). A track is points, each with coordinates as attributes and
optional children.

``` xml title="track.gpx" linenums="1"
<gpx version="1.1" creator="AvantGPS"
     xmlns="http://www.topografix.com/GPX/1/1"
     xmlns:gpxtpx="http://www.garmin.com/xmlschemas/TrackPointExtension/v1">
  <metadata>
    <name>Morning run</name>
    <time>2026-06-20T06:00:00Z</time>
  </metadata>
  <trk>
    <name>Loop</name>
    <trkseg>
      <trkpt lat="60.1699" lon="24.9384">              <!-- (1)! -->
        <ele>12.3</ele>
        <time>2026-06-20T06:00:01Z</time>
        <extensions>                                   <!-- (2)! -->
          <gpxtpx:TrackPointExtension>
            <gpxtpx:hr>142</gpxtpx:hr>
          </gpxtpx:TrackPointExtension>
        </extensions>
      </trkpt>
    </trkseg>
  </trk>
</gpx>
```

1.  Position is in **attributes** (`lat`, `lon`), elevation and time in child
    elements — a deliberate split between identity and payload.
2.  The `<extensions>` element is GPX's named escape hatch. The base schema does
    not define heart rate, so Garmin put `hr` in *its own* namespace
    (`gpxtpx:`) and dropped it inside `<extensions>`. A reader that only knows
    base GPX skips the whole subtree.

### The GPX schema, browseably

GPX is a good schema to read because it is *small* and uses the constructs from
the XSD chapter without ceremony. `unxml --xsd` renders it as a data model:

``` text title="unxml --xsd gpx.xsd (excerpt)"
schema http://www.topografix.com/GPX/1/1 (elementFormDefault=qualified)
  xmlns = http://www.topografix.com/GPX/1/1
  element gpx : gpxType
  type gpxType
    metadata : metadataType ?            # (1)!
    wpt : wptType *
    trk : trkType *
    extensions : extensionsType ?        # (2)!
    @version : xsd:string (required)
    @creator : xsd:string (required)
  type wptType
    ele : xsd:decimal ?
    time : xsd:dateTime ?
    @lat : latitudeType (required)
    @lon : longitudeType (required)
  type latitudeType : xsd:decimal [-90.0..90.0]     # (3)!
```

1.  The occurrence suffixes are `unxml`'s shorthand: `?` optional, `*`
    zero-or-more, `+` one-or-more — the same
    [`minOccurs`/`maxOccurs`](../xsd/complex-types.md) you met in the XSD chapter,
    compressed to one character.
2.  `extensions : extensionsType ?` — the hatch is a *typed element* in the
    schema, and `extensionsType` itself is an `any` [wildcard](../xsd/complex-types.md)
    accepting foreign-namespace content. Extension is a first-class, *named* part
    of the model, not an afterthought.
3.  `latitudeType : xsd:decimal [-90.0..90.0]` is a
    [range-restricted simple type](../xsd/simple-types-restrictions.md) — the
    schema enforces that latitude is a real coordinate, so `lat="999"` fails
    validation. The bracket range is `unxml`'s rendering of
    `minInclusive`/`maxInclusive`.

## KML: extension by reserved prefix

KML (Keyhole Markup Language, now an OGC standard) describes places, styles, and
overlays. Its extension style is different: instead of a generic `<extensions>`
bag, Google reserved a **prefix**, `gx:`, for vendor extensions that live *inline*
wherever they apply.

``` xml title="place.kml" linenums="1"
<kml xmlns="http://www.opengis.net/kml/2.2"
     xmlns:gx="http://www.google.com/kml/ext/2.2">
  <Document>
    <name>Sites</name>
    <Style id="pin"><IconStyle><scale>1.2</scale></IconStyle></Style>   <!-- (1)! -->
    <Placemark>
      <name>Helsinki</name>
      <styleUrl>#pin</styleUrl>                                <!-- (2)! -->
      <Point><coordinates>24.9384,60.1699,0</coordinates></Point>      <!-- (3)! -->
    </Placemark>
  </Document>
</kml>
```

1.  A `Style` is defined once with an `id`, then referenced — the same
    define-once/reference-by-id idea as [OOXML relationships](office-ooxml-odf.md)
    and SVG's `<defs>`, recurring across vocabularies.
2.  `styleUrl>#pin` is the reference. KML reuses the URL-fragment convention
    (`#id`) for intra-document links.
3.  `coordinates` is **lon,lat,alt** — longitude *first*, the opposite of GPX's
    `lat`/`lon` attributes. A perennial bug source when converting between the two.

!!! note "Two extension philosophies, side by side"
    - **GPX**: a single, schema-defined `<extensions>` container — extensions are
      *quarantined* in one place, easy for a base reader to skip wholesale.
    - **KML**: a reserved `gx:` namespace used *inline* — extensions appear right
      where they belong (e.g. `gx:Track` beside a `Point`), at the cost of being
      sprinkled throughout the document.

    Both are valid; the choice is between *containment* and *locality*. When you
    design your own extensible format, this is the fork in the road.

## Querying coordinates

Both default-namespace their core, so the
[SVG namespace caveat](svg.md#querying-namespaced-svg-with-xpath) applies again:
`//trkpt` or `//Placemark` match nothing until you bind the GPX / KML namespace
URI to a prefix and query `//g:trkpt`. The vendor extensions (`gpxtpx:`, `gx:`)
are already prefixed, but you still must bind *their* URIs in your query host.

## What GPX/KML teach

- Two real, readable geo vocabularies that are small enough to learn whole — and
  whose **schemas** read cleanly under `unxml --xsd`.
- **Extension by container** (`<extensions>`) vs **extension by reserved prefix**
  (`gx:`) — the two dominant strategies, with opposite trade-offs.
- Simple-type [restrictions](../xsd/simple-types-restrictions.md) earn their keep:
  a latitude range catches bad data at validation time.
- Coordinate order (`lat,lon` vs `lon,lat`) is a reminder that a schema constrains
  *structure*, not *meaning* — that part is still on you.

Next: [XBRL](xbrl.md) — namespacing pushed to its limit, where you define a whole
namespace of business *concepts*.
