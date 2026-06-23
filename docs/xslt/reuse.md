# Reusing stylesheets

A real transformation is rarely one file. As it grows, the same scaffolding —
formatting rules, shared helper functions, common parameters, element templates —
gets copied between projects. The fix is the same as in any other language:
factor the shared parts into a **base** stylesheet and let each project pull it
in. XSLT offers two top-level elements for this, `xsl:include` and `xsl:import`,
and the difference between them is entirely about **precedence**.

What they pull in is **every top-level declaration** of the other stylesheet —
not just `xsl:template`. Params, variables, functions, keys, output settings,
attribute-sets: all of it merges in, and precedence decides what happens when two
declarations collide. It is easy to read the examples below as being about
*templates* (templates have a little extra machinery, so they make the tidiest
demos), but the mechanism is general. The most common real use — including the
[mapping pipelines](at-scale.md) this section builds toward — overrides
**parameters and functions** and never touches a `match` template at all.

## xsl:include — textual merge, same precedence

`xsl:include` is the simple one. It takes the declarations from another
stylesheet and merges them in as if you had pasted them at the point of
inclusion. Everything ends up at **the same import precedence**.

``` xml title="main.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:include href="base.xsl"/>          <!-- (1)! -->

<xsl:template match="/">
  <catalogue>
    <xsl:apply-templates select="catalog/cd"/>   <!-- (2)! -->
  </catalogue>
</xsl:template>

</xsl:stylesheet>
```

1.  Pulls in every template from `base.xsl` at the same precedence as the rules
    written here. It is a top-level element — a direct child of `xsl:stylesheet`.
2.  The `cd` template that handles these can live in `base.xsl`; the merge makes
    it available exactly as if it were defined locally.

!!! warning "Same precedence means conflicts are errors"
    Because an included stylesheet sits at the *same* precedence as the including
    one, two templates matching the same nodes with the same priority is a genuine
    conflict. The XSLT 1.0 spec lets a processor either signal an error or pick the
    last one — it is **processor-dependent**, so never rely on it. Use `include`
    only when the files are partitioned so no rule overlaps.

## xsl:import — lower precedence, so you can override

`xsl:import` brings in another stylesheet too, but its declarations get
**lower import precedence** than the importing stylesheet. That single rule is
what makes overriding possible: when both define the same thing — a parameter of
the same name, a function of the same signature, a template for the same nodes —
the importing stylesheet **wins**, no conflict, no error. The plainest case is a
parameter:

!!! warning "`xsl:import` must come first"
    All `xsl:import` elements must appear **before** every other top-level element
    in the stylesheet — before any template, variable, output declaration, even
    before `xsl:include`. A processor will reject an `import` that follows other
    top-level content.

``` xml title="main.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:import href="base.xsl"/>                  <!-- (1)! -->

<xsl:param name="heading" select="'Our catalogue'"/>   <!-- (2)! -->

</xsl:stylesheet>
```

1.  Must be the first top-level element. `base.xsl` is now at lower precedence.
2.  `base.xsl` also declares a global `heading` param; this one has higher import
    precedence, so it wins — no conflict, no error. Overriding is just
    "re-declare it higher up," and it works the same for a variable, a function,
    or a template.

## include vs import at a glance

| | `xsl:include` | `xsl:import` |
| --- | --- | --- |
| Precedence of pulled-in declarations | **same** as host | **lower** than host |
| Can the host override them? | no — same precedence | yes — host wins cleanly |
| Conflicting declarations | error / processor-dependent | resolved by precedence |
| Position among top-level elements | anywhere | **must be first** |
| Reach the overridden rule (templates only) | n/a | `xsl:apply-imports` |

## Precedence applies to every declaration kind

Import precedence is not a template feature. It is a property of *every* kind of
top-level declaration, and it resolves a collision differently depending on the
kind:

| Declaration | What import precedence does on a clash |
| --- | --- |
| `xsl:param` / `xsl:variable` | higher-precedence (importing) declaration **wins**; the imported one is ignored |
| `xsl:function` | a same-name/same-arity function at higher precedence **overrides** the imported one |
| `xsl:template` (match) | higher precedence wins — *and* the shadowed rule is still reachable via `xsl:apply-imports` / `xsl:next-match` |
| `xsl:key`, `xsl:attribute-set` | same-name declarations **combine** rather than override |
| `xsl:output`, `xsl:decimal-format` | merged property-by-property |

Each row is a different *kind of thing a stylesheet can declare* — and every one
of them is what `import`/`include` carries across. Here is one of each in a single
base module; the `match` template is just the last line, not the centre of gravity:

``` xml title="base.xsl — one of each top-level declaration" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="3.0"
                xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                xmlns:xs="http://www.w3.org/2001/XMLSchema"
                xmlns:f="urn:example:fns"
                exclude-result-prefixes="xs f"
                expand-text="yes">

  <xsl:output method="html" indent="yes"/>                       <!-- serialization settings -->
  <xsl:strip-space elements="*"/>                                <!-- whitespace policy -->
  <xsl:decimal-format name="eu" decimal-separator="," grouping-separator="."/>

  <xsl:param name="title" as="xs:string" select="'Catalogue'"/>  <!-- global parameter -->
  <xsl:variable name="version" select="'1.0'"/>                  <!-- global variable -->

  <xsl:key name="cd-by-id" match="cd" use="@id"/>                <!-- index for key() -->

  <xsl:attribute-set name="cell">                                <!-- reusable attributes -->
    <xsl:attribute name="class">cell</xsl:attribute>
  </xsl:attribute-set>

  <xsl:function name="f:money" as="xs:string">                   <!-- helper function -->
    <xsl:param name="n" as="xs:decimal"/>
    <xsl:sequence select="format-number($n, '#.##0,00', 'eu')"/>
  </xsl:function>

  <xsl:template name="footer"><footer>{$version}</footer></xsl:template>  <!-- named template -->

  <xsl:template match="cd"><li>{title}</li></xsl:template>       <!-- match template -->

</xsl:stylesheet>
```

Import or include that, and *all* of it comes along — the output settings, the
param, the variable, the key, the attribute-set, the function, both templates.
A higher-precedence module redeclaring any of those names follows its row in the
table above: `title` and `f:money` are replaced, `cell` and `cd-by-id` combine,
the `output` settings merge.

Two declarations of the same name at the *same* import precedence are an error
(e.g. two global params with one name) — `import` exists precisely to put one of
them lower so the clash resolves cleanly instead. The only kind with extra
machinery is the template: `apply-imports` lets an override *reach back* to the
rule it shadowed. There is no equivalent for "call the param I overrode" — for
params, variables, and functions, the higher-precedence declaration simply
replaces the lower one. That makes them the *simplest* things to override, which
is why the most common pattern uses exactly them.

## Worked example: override parameters, not templates

This is the plainest use of `xsl:import` and the one that scales to real mapping
work. A **base** stylesheet owns all the output — the templates, the types, the
guards — and declares its inputs as **bare parameters with no `select`**. It is
hand-authored once and rarely changes:

``` xml title="card-base.xsl (the scaffold)" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="3.0"
                xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                xmlns:xs="http://www.w3.org/2001/XMLSchema"
                expand-text="yes">

  <xsl:param name="name"  as="xs:string?"/>   <!-- (1)! -->
  <xsl:param name="email" as="xs:string?"/>
  <xsl:param name="org"   as="xs:string?"/>

  <xsl:template match="/">                     <!-- (2)! -->
    <card>
      <xsl:if test="exists($name)"><name>{$name}</name></xsl:if>
      <xsl:if test="exists($email)"><email>{$email}</email></xsl:if>
      <xsl:if test="exists($org)"><org>{$org}</org></xsl:if>
    </card>
  </xsl:template>

</xsl:stylesheet>
```

1.  No `select` — the base does not know *where* the value comes from. Each
    parameter is a named slot a source module fills.
2.  The one and only output template lives here. Source modules add none.

A **source module** imports the scaffold and re-declares each parameter **with a
`select`** that reaches into one particular input shape. Because the importing
module has higher import precedence, its declaration wins; the bare one in the
base is shadowed and the `select` becomes the value:

``` xml title="contact-to-card.xsl (a binding module)" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="3.0"
                xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

  <xsl:import href="card-base.xsl"/>                 <!-- (1)! -->

  <xsl:param name="name"  select="/contact/fullName"/>        <!-- (2)! -->
  <xsl:param name="email" select="/contact/emails/primary"/>
  <xsl:param name="org"   select="/contact/company/@name"/>

</xsl:stylesheet>
```

1.  First, as always. Everything in `card-base.xsl` is now lower precedence.
2.  Higher precedence than the base's `name` param, so *this* one is used and its
    `select` (evaluated against the source document's root) supplies the value.
    No template here at all — the base's `match="/"` fires and reads these params.

Point Saxon at a `<contact>` document with `contact-to-card.xsl` and you get the
`<card>`. A second input vocabulary needs a second module that binds the **same
three slots** from a different shape — `<xsl:param name="name" select="vcard/fn"/>`
and so on — with the scaffold untouched.

!!! tip "This is the codegen seam"
    A binding module is nothing but a list of `<xsl:param … select="…"/>` lines —
    a flat *slot → XPath* table. That is exactly the kind of file you can
    **generate** from a mapping spreadsheet or a script, while the scaffold (the
    output structure, the typed signatures, the `exists()` guards) stays
    **hand-authored**. Generated and hand-written XSLT meet only at the parameter
    names. The same trick works for `xsl:function`: put a default implementation
    in the base and override it in a module that needs different behaviour.
    [A whole case study](generated-and-handwritten.md) builds out this
    generated-plus-hand-written split — and shows why the two binding files want
    `include`, not `import`.

## Templates add one more thing: reaching the shadowed rule

!!! note "Advanced — skip on a first read"
    Everything above is the whole composition mechanism. This last section is one
    extra trick that applies *only* to `match` templates; params, variables, and
    functions neither have it nor need it.

Overriding a `match` template works the same way — higher precedence wins — but
templates have the one extra capability the table above noted: an override can
re-run the rule it replaced, against the current node, with `xsl:apply-imports`.
Here `base.xsl` provides a generic `cd` template the whole organisation shares:

``` xml title="base.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:template match="catalog">
  <list>
    <xsl:apply-templates select="cd"/>
  </list>
</xsl:template>

<xsl:template match="cd">                 <!-- (1)! -->
  <item>
    <xsl:value-of select="title"/> — <xsl:value-of select="price"/>
  </item>
</xsl:template>

</xsl:stylesheet>
```

1.  The generic rendering of a `<cd>`. Most projects are happy with it as-is.

One project wants the same overall structure but a richer `cd` line — with the
artist, and the price tagged. It imports the base and overrides **only** the
`cd` template; the `catalog` template is inherited untouched.

``` xml title="main.xsl" linenums="1"
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

<xsl:import href="base.xsl"/>             <!-- (1)! -->

<xsl:template match="cd">                 <!-- (2)! -->
  <item>
    <by><xsl:value-of select="artist"/></by>
    <xsl:apply-imports/>                  <!-- (3)! -->
  </item>
</xsl:template>

</xsl:stylesheet>
```

1.  First, as required. `base.xsl` is now lower-precedence.
2.  Overrides the base `cd` template. The base `catalog` template still fires and
    still calls `apply-templates select="cd"` — but now *this* rule answers.
3.  `xsl:apply-imports` runs the template this one overrode — the lower-precedence
    `cd` from `base.xsl` — against the **same** current node. So we add the artist,
    then delegate the title/price line back to the base instead of duplicating it.

Run against the [catalog](index.md#the-running-example):

<div class="xslt-result" markdown>
```
<list><item><by>Bob Dylan</by><item>Empire Burlesque — 10.90</item></item>...</list>
```
</div>

!!! tip "`apply-imports` is `super` for templates"
    Think of it as calling the base class's method. It does not take a `select` —
    it re-applies the imported templates for the **current** node only, letting an
    override decorate the inherited output rather than replace it wholesale.

!!! note "Where `href` is resolved"
    For both `include` and `import`, `href` is resolved **relative to the
    stylesheet that contains it** — not relative to the source document or the
    working directory. If `main.xsl` and `base.xsl` sit side by side,
    `href="base.xsl"` is correct; a base in a subfolder would be
    `href="common/base.xsl"`.

## What goes in a module

Once a stylesheet is more than one file, a handful of module shapes recur. The
most common is a **function or helper library** — a file of `xsl:function`s (date
and money formatting, string helpers, code-list lookups) with no `match`
templates, `include`d wherever they are needed. Close behind is a **parameter
module**, a file of global `xsl:param` defaults that downstream layers `import`
and override — the configuration surface of [large stylesheets](at-scale.md) and
the scaffold half of the [generated-plus-hand-written](generated-and-handwritten.md)
pattern. **Lookup-data modules** hold `xsl:variable` maps or `xsl:key`
declarations over embedded code lists; **output-settings modules** centralise
`xsl:output`, `xsl:strip-space`, and `xsl:decimal-format`. At larger scale the
split turns structural rather than by-kind: a **base layer** others override,
**per-output-format layers** (HTML vs print vs EPUB), a **module per element
family**, and **modes** wiring them together — all of which the
[DocBook case study](at-scale.md) walks through in real code.

Two things this page does *not* cover. Pulling in runtime *data* — a second XML
file, JSON, plain text — is a different mechanism, not `import`/`include`; that is
[external documents](external-documents.md), below. And for a genuine library with
an *enforced* public surface rather than the textual merge `import` performs, XSLT
3.0's [packages](packages.md) are the stronger tool.

## Next

Pulling in other stylesheets has a sibling: pulling in other *data*.
[External documents](external-documents.md) covers `document()`, which lets a
transformation read a second XML file at run time.
