# Regular expressions and strings

XSLT 1.0's string toolbox — `translate`, `substring-before` / `substring-after`
— gets you a long way, but it has no notion of a *pattern*. There is no real
split, no general replace, no way to test "does this look like a date?". XSLT
2.0 (and 3.0) fix that: they add regex-driven functions plus a handful of string
helpers that make the old `translate` idioms unnecessary. Everything here needs
`version="2.0"` or `"3.0"` on the stylesheet and a 2.0/3.0 processor such as
Saxon.

For the 1.0 equivalents — and how much more work they were — see
[String functions](strings.md).

## At a glance

| Function | Does | Tiny example → result |
| --- | --- | --- |
| `tokenize(s, pattern)` | Split `s` on a regex into a sequence of strings | `tokenize('a,b,c', ',')` → `("a", "b", "c")` |
| `replace(s, pattern, repl)` | Regex replace; `$1`…`$9` reference capture groups | `replace('2024-01', '-', '/')` → `2024/01` |
| `matches(s, pattern)` | Boolean: does the regex match anywhere in `s`? | `matches('AB12', '[A-Z]+[0-9]+')` → `true` |
| `upper-case(s)` | Uppercase the whole string | `upper-case('abc')` → `ABC` |
| `lower-case(s)` | Lowercase the whole string | `lower-case('ABC')` → `abc` |
| `string-join(seq, sep)` | Glue a sequence back into one string | `string-join(('a','b'), '-')` → `a-b` |
| `normalize-space(s)` | Trim ends, collapse internal whitespace (1.0, still useful) | `normalize-space(' a  b ')` → `a b` |

These three — `tokenize`, `replace`, `matches` — all take a regular expression
in the same XPath/XML-Schema regex dialect, and all accept an optional **flags**
argument (covered at the end).

## A field to play with

The running [catalog](index.md#the-running-example) has `cd/title/artist/price`.
For these examples imagine each `cd` also carries a hyphenated catalog code and a
comma-separated tag list:

``` xml title="catalog-codes.xml" linenums="1"
<catalog>
  <cd code="ROCK-1985-017" tags="rock, folk, classic">
    <title>Empire Burlesque</title>
    <artist>Bob Dylan</artist>
    <price>10.90</price>
  </cd>
</catalog>
```

## Splitting: tokenize

`tokenize($s, $pattern)` cuts a string wherever the regex matches and returns the
pieces as a **sequence** of strings. Splitting a delimited list is now a
one-liner:

``` xml title="split-tags.xsl" linenums="1"
<xsl:for-each select="tokenize(@tags, ',\s*')">   <!-- (1)! -->
  <tag><xsl:value-of select="."/></tag>
</xsl:for-each>
```

1.  Split on a comma followed by any run of whitespace, so the pieces come back
    already trimmed.

<div class="xslt-result" markdown>
```
<tag>rock</tag>
<tag>folk</tag>
<tag>classic</tag>
```
</div>

!!! note "Contrast with XSLT 1.0"
    In 1.0 there is no split at all — you write a recursive named template that
    peels off one `substring-before(...)` at a time and calls itself with the
    `substring-after(...)` remainder. `tokenize` replaces that whole pattern with
    a single function call. See the recursive idiom in
    [String functions](strings.md).

Because the result is a real sequence, you can count or index it:

``` xml
<xsl:value-of select="count(tokenize(@tags, ',\s*'))"/>   <!-- → 3 -->
```

## Replacing: replace

`replace($s, $pattern, $replacement)` performs a regex search-and-replace over
the whole string. The replacement may refer to **capture groups** from the
pattern with `$1`, `$2`, … :

``` xml title="reformat-code.xsl" linenums="1"
<xsl:value-of select="replace(@code, '^([A-Z]+)-(\d{4})-(\d+)$',
                               '$2/$1 (#$3)')"/>   <!-- (1)! -->
```

1.  Three capture groups — genre, year, serial — reordered in the replacement.

<div class="xslt-result" markdown>
1985/ROCK (#017)
</div>

!!! warning "`$` and `\` are special in the replacement"
    Inside the replacement string `$` introduces a group reference and `\` is an
    escape. To output a literal `$` or `\`, double the backslash: `\$` and `\\`.

A plain replace with no groups works too — `replace('a.b.c', '\.', '-')` gives
`a-b-c` (note `\.` to match a literal dot, since bare `.` matches any character).

## Testing: matches

`matches($s, $pattern)` returns a boolean — true if the regex matches anywhere in
the string. It is the natural fit for `xsl:if` and predicates:

``` xml
<xsl:if test="matches(@code, '^[A-Z]+-\d{4}-\d+$')">
  <p>Well-formed catalog code.</p>
</xsl:if>
```

Anchor with `^` and `$` when you mean "the *whole* string matches"; without them
`matches` is satisfied by any substring.

## Match by match: xsl:analyze-string

`tokenize` / `replace` / `matches` treat the string as a whole. When you need to
walk a string and handle the matched and unmatched pieces **differently**, use
`xsl:analyze-string`. It runs the regex over `select`, then routes each piece to
one of two children:

- `xsl:matching-substring` — fires once per match; inside it `regex-group(n)`
  gives the captured groups.
- `xsl:non-matching-substring` — fires for the text *between* matches.

Here we parse the `code` attribute into its three parts:

``` xml title="parse-code.xsl" linenums="1"
<xsl:analyze-string select="@code" regex="([A-Z]+)-(\d{{4}})-(\d+)">  <!-- (1)! -->
  <xsl:matching-substring>
    <parsed>
      <genre><xsl:value-of select="regex-group(1)"/></genre>   <!-- (2)! -->
      <year><xsl:value-of select="regex-group(2)"/></year>
      <serial><xsl:value-of select="regex-group(3)"/></serial>
    </parsed>
  </xsl:matching-substring>
  <xsl:non-matching-substring>
    <unparsed><xsl:value-of select="."/></unparsed>            <!-- (3)! -->
  </xsl:non-matching-substring>
</xsl:analyze-string>
```

1.  In the `regex` **attribute** the braces of a quantifier must be doubled
    (`\d{{4}}`) because XSLT first does attribute-value-template expansion on
    `{ }`. In `replace`/`tokenize` *string arguments* you write `\d{4}` normally.
2.  `regex-group(1)` is the first parenthesised group of the current match.
3.  Anything that did not match the regex arrives here as the context item `.`.

<div class="xslt-result" markdown>
```
<parsed>
  <genre>ROCK</genre>
  <year>1985</year>
  <serial>017</serial>
</parsed>
```
</div>

A common second use is **wrapping** matched tokens while leaving the rest of the
text alone — e.g. turning every `#tag` in a string into an element, with the
plain text passed through verbatim:

``` xml title="wrap-hashtags.xsl" linenums="1"
<xsl:analyze-string select="'see #rock and #folk here'" regex="#(\w+)">
  <xsl:matching-substring>
    <tag><xsl:value-of select="regex-group(1)"/></tag>
  </xsl:matching-substring>
  <xsl:non-matching-substring>
    <xsl:value-of select="."/>
  </xsl:non-matching-substring>
</xsl:analyze-string>
```

<div class="xslt-result" markdown>
```
see <tag>rock</tag> and <tag>folk</tag> here
```
</div>

## The new string helpers

A few small additions retire the clumsiest 1.0 idioms:

- **`upper-case($s)` / `lower-case($s)`** — case conversion in one call, instead
  of the 1.0 `translate($s, $lower, $upper)` dance: `upper-case('Bob Dylan')` →
  `BOB DYLAN`.
- **`string-join($seq, $sep)`** — the inverse of `tokenize`; joins a sequence of
  strings with a separator: `string-join(('rock','folk'), ' / ')` →
  `rock / folk`.
- **`normalize-space($s)`** — unchanged from 1.0, but as relevant as ever: trims
  the ends and collapses internal whitespace runs to single spaces. Reach for it
  before comparing or splitting messy source text.

A round trip — split, transform, rejoin — reads cleanly now:

``` xml
<xsl:value-of select="string-join(
    for $t in tokenize(@tags, ',\s*') return upper-case($t),
    ' | ')"/>
```

<div class="xslt-result" markdown>
ROCK | FOLK | CLASSIC
</div>

## Regex flags

The regex functions take an optional final **flags** string. The most useful is
`i` for case-insensitive matching:

``` xml
<xsl:value-of select="matches('Rock', 'rock', 'i')"/>   <!-- → true -->
```

Other flags include `s` (dot matches newlines), `m` (multi-line `^`/`$`) and `x`
(ignore whitespace in the pattern). `xsl:analyze-string` accepts the same set via
its own `flags` attribute.

!!! tip "One dialect everywhere"
    `tokenize`, `replace`, `matches` and `xsl:analyze-string` all use the same
    XPath regex syntax and the same flags, so a pattern you build for one works
    unchanged in the others.

## Next

[Modern identity and text](modern-identity.md) — with strings under control, the
next step is the 2.0/3.0 way to copy a document and tweak just the parts you care
about.
