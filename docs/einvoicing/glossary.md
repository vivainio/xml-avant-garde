# Glossary

E-invoicing runs on acronyms. This page defines them in plain language, grouped by
what they *are* rather than alphabetically, so the relationships are visible. Each
term links to where it is used in depth.

## Standards and the bodies behind them

EN16931
:   The European **semantic** standard for a core electronic invoice — the data
    model and the business rules, with *no* mention of XML. "Compliant with
    EN16931" means a document carries the right information and obeys the rules; it
    says nothing about syntax. Published by CEN. The thing everything else on these
    pages specialises or implements. See the [Overview](index.md).

CEN
:   *Comité Européen de Normalisation* — the European standards body that publishes
    EN16931. (The "EN" in EN16931 = *European Norm*.)

Semantic model
:   A description of *what information* a document carries, independent of how it is
    written down. EN16931 is a semantic model; UBL and CII are two ways to *write*
    it.

Business Term (BT) / Business Group (BG)
:   The numbered fields of the EN16931 semantic model. **BT-1** is "Invoice
    number", **BT-24** is "Specification identifier", **BG-4** is "Seller" (a group
    of related BTs). A *syntax binding* maps each BT/BG to a concrete element. See
    [Anatomy of a UBL invoice](ubl-invoice.md#business-terms-vs-elements).

## Syntaxes — how the model is written as XML

Syntax / syntax binding
:   A concrete XML vocabulary that carries an EN16931 invoice, plus the mapping
    from each business term to an element. EN16931 has two official bindings.

UBL
:   *Universal Business Language* (OASIS). One of the two XML syntaxes; documents
    are rooted at `<Invoice>` and use the `cbc:`/`cac:` namespaces. The syntax used
    in most examples on this site. See [Anatomy of a UBL invoice](ubl-invoice.md).

CII
:   *Cross Industry Invoice* (UN/CEFACT). The other XML syntax, using `rsm:`/`ram:`
    namespaces and the syntax behind Factur-X/ZUGFeRD. Carries the *same* business
    terms as UBL — different element names, identical meaning. Why EN16931's rules
    are written as
    [abstract patterns bound twice](../schematron/abstract-patterns-en16931.md).
    See [Anatomy of a CII invoice](cii-invoice.md).

cbc / cac
:   The two UBL namespaces you see on nearly every element. **`cbc`**
    (*CommonBasicComponents*) = leaf fields that hold a single value;
    **`cac`** (*CommonAggregateComponents*) = containers that hold other elements.
    Mnemonic: *cbc is a value, cac is a box.*

rsm / ram / udt
:   CII's namespaces. **`rsm`** = the *message* structure (the root and its three
    top-level parts); **`ram`** (*Reusable Aggregate Business Information Entity*) =
    almost everything, both groups and leaf fields; **`udt`** (*Unqualified Data
    Type*) = low-level typed-value wrappers like `udt:DateTimeString`. Unlike UBL,
    CII does **not** split leaf from container by namespace.

CCTS / BIE
:   *Core Component Technical Specification* — the UN/CEFACT method behind CII's
    verbose names. Every element is a **Business Information Entity (BIE)** named as
    *Object · Property · Representation*, which is why
    `SpecifiedTradeSettlementHeaderMonetarySummation` reads as a sentence once
    decomposed. Kinds: **ABIE** (aggregate), **ASBIE** (association to another
    aggregate), **BBIE** (basic/leaf value). See
    [Decoding the names](cii-invoice.md#decoding-the-names).

## Profiles — narrowing the core for real networks

Peppol
:   A cross-border network for exchanging business documents (invoices, orders,
    etc.), governed by **OpenPeppol**. Originally "Pan-European Public Procurement
    OnLine". More than a format — it is a network with registered participants and
    transport rules. See [Peppol and CIUS profiles](peppol-cius.md).

Peppol BIS
:   *Business Interoperability Specification* — Peppol's published profile
    documents. **Peppol BIS Billing 3.0** is the invoicing one: a CIUS of EN16931
    that says exactly how to fill in a UBL (or CII) invoice for exchange over
    Peppol. So "Peppol BIS" ≈ "the Peppol rulebook for this document type." Despite
    the intimidating name, it is just *EN16931, narrowed, plus a handful of
    network-specific rules.*

CIUS
:   *Core Invoice Usage Specification* — a profile that **narrows** EN16931:
    it can make optional fields mandatory or restrict code lists, but **cannot add
    new business terms**. Because it only narrows, every CIUS document is still a
    valid EN16931 document. Peppol BIS Billing 3.0 is a CIUS. See
    [CIUS vs extension](peppol-cius.md#cius-vs-extension).

Extension
:   The other way to specialise EN16931: it *may* add new business terms beyond the
    core. More powerful than a CIUS, but the result is no longer fully readable by a
    plain EN16931 receiver.

Factur-X / ZUGFeRD
:   **Hybrid** invoice formats: a human-readable **PDF/A-3** with the structured
    invoice **embedded as [CII](cii-invoice.md) XML** inside it, so one file serves
    both people and machines. **ZUGFeRD** is the German standard (published by FeRD);
    **Factur-X** is the identical French–German aligned profile (FNFE-MPE), central to
    France's B2B e-invoicing mandate. Both embed an EN16931-compliant CII document —
    the main reason CII is worth knowing even outside the [Peppol](peppol-network.md)
    network. See [Factur-X & ZUGFeRD](../realworld/facturx-zugferd.md) for how the XML
    is embedded.

CustomizationID (BT-24) / ProfileID (BT-23)
:   The two fields by which an invoice **declares which rulebook and process it
    follows**. `CustomizationID` names the specification (plain EN16931, or a Peppol
    URN); `ProfileID` names the business process. See
    [how an invoice declares its profile](peppol-cius.md#how-an-invoice-declares-its-profile).

## Validation

Schematron
:   A rule-based XML validation language: assertions of the form *"in this context,
    this must be true."* EN16931's business rules ship as Schematron. It compiles to
    [XSLT](../xslt/index.md) to run. Full treatment in the
    [Schematron section](../schematron/index.md).

Business rule (BR-*)
:   A numbered EN16931 rule, e.g. **BR-01** ("an invoice shall have a specification
    identifier") or **BR-CO-10** (an arithmetic total check). Distributed as public
    Schematron. The `CO` family are *calculation* rules.

SVRL
:   *Schematron Validation Report Language* — the XML report a Schematron run
    produces, listing which assertions fired, with what severity and message. When
    an invoice is "rejected by validation", the SVRL names the exact rule. See
    [the pipeline](validation-pipeline.md#the-layers-as-a-table).

XSD
:   *XML Schema Definition* — the grammar layer that checks *structure and
    datatypes* (is this element allowed here, is this a valid date). Runs before
    Schematron; catches different errors. See the [XSD section](../xsd/index.md).

## Code lists

Code list
:   A controlled set of allowed values for a coded field — currencies, country
    codes, invoice-type codes. A coded field is only valid if its value is in the
    right list. See [Genericode code lists](genericode-codelists.md).

Genericode (.gc)
:   The OASIS file format that expresses a code list as data: a typed table with a
    key column and one row per code. EN16931 ships its lists as `.gc` files. See
    [Genericode code lists](genericode-codelists.md#what-a-gc-file-is).

UNCL
:   *United Nations Code List* — UN/EDIFACT-derived lists used by EN16931, each with
    a number: **UNCL1001** (invoice type codes), **UNCL5305** (VAT category codes),
    **UNCL4461** (payment means). The number identifies *which* list.

ISO 4217 / ISO 3166 / UN/ECE Rec 20
:   The external standards behind common lists: **ISO 4217** = currency codes
    (`EUR`), **ISO 3166-1** = country codes (`FI`), **UN/ECE Recommendation 20** =
    units of measure (`C62` = "one/piece").

## Identifiers and the network

Electronic address (EndpointID)
:   The routable identifier of a party on the network — *who to deliver to*. Peppol
    requires it (`cbc:EndpointID`) though plain EN16931 does not. See
    [what Peppol adds](peppol-cius.md#what-peppol-adds).

EAS / ICD
:   Code lists of **identifier schemes**. **EAS** (*Electronic Address Scheme*) says
    what kind of electronic address an `EndpointID` is; **ICD** (*International Code
    Designator*) does the same for organisation identifiers. The `schemeID`
    attribute on an identifier names which scheme it belongs to.

Four-corner model
:   Peppol's delivery architecture: sender (corner 1) → sender's **Access Point**
    (2) → receiver's Access Point (3) → receiver (4). The two middle corners are
    service providers; the addressing in the document is what lets them route it.

Access Point / SMP / SML
:   The Peppol transport machinery. An **Access Point** sends and receives on a
    participant's behalf; the **SMP** (*Service Metadata Publisher*) records what a
    participant can receive and where; the **SML** (*Service Metadata Locator*) is
    the network-wide directory that points to the right SMP. Transport-layer detail,
    outside the document itself.

KoSIT
:   The German public-IT coordination body that maintains the widely-used reference
    **validator** for EN16931 — the bundle of Schematron rules and `.gc` code lists
    much of the open-source tooling is built on.

## Back to the section

Return to the [Overview](index.md), or jump to
[Anatomy of a UBL invoice](ubl-invoice.md) to see these terms on a real document.
