---
icon: lucide/webhook
---

# SOAP and WSDL — namespaces as a contract

Where [SVG](svg.md) used namespaces for convenience, **SOAP** uses them as a
*contract between machines*. A SOAP message is an envelope from one namespace
wrapping a payload from another, and a **WSDL** file is a machine-readable
description that stitches several namespaces together — including an embedded
[XSD](../xsd/index.md) for the payload types. If you have ever wondered why
enterprise XML is *so* prefix-heavy, this is where the habit comes from.

## The envelope

Every SOAP message has the same outer shape: an `Envelope` containing an optional
`Header` and a mandatory `Body`. The envelope elements live in the SOAP namespace;
everything inside the `Body` is *your* application's namespace.

``` xml title="reserve.xml" linenums="1"
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope"
               xmlns:wsa="http://www.w3.org/2005/08/addressing"
               xmlns:m="http://travel.example.org/reservation">
  <soap:Header>                                          <!-- (1)! -->
    <wsa:To>http://travel.example.org/booking</wsa:To>
    <wsa:Action>http://travel.example.org/reserve</wsa:Action>
  </soap:Header>
  <soap:Body>                                            <!-- (2)! -->
    <m:reserve>
      <m:flight>BA284</m:flight>
      <m:date>2026-07-01</m:date>
    </m:reserve>
  </soap:Body>
</soap:Envelope>
```

1.  The `Header` carries *infrastructure* metadata in its own namespaces —
    addressing (`wsa:`), security, transactions. Each WS-\* spec owns a namespace,
    and they coexist in the header without colliding. This is namespaces doing the
    job they were designed for: independent vocabularies, one document.
2.  The `Body` holds the actual message. `m:reserve` is *your* operation, in
    *your* namespace. SOAP deliberately knows nothing about it — the envelope is a
    transport, the payload is yours.

The three prefixes mark a **three-layer** stack: transport (`soap:`),
infrastructure (`wsa:`), payload (`m:`). Three specs, three namespaces, zero
ambiguity about which element belongs to which.

## WSDL: the contract that imports a schema

A SOAP service is described by a **WSDL** (Web Services Description Language)
document. WSDL is itself an XML vocabulary, and a single WSDL file routinely
juggles four namespaces at once: WSDL itself, the SOAP binding, the XSD that
defines the message *types*, and the service's own target namespace.

``` xml title="booking.wsdl" linenums="1"
<wsdl:definitions xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/"
                  xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/"
                  xmlns:xsd="http://www.w3.org/2001/XMLSchema"
                  xmlns:tns="http://travel.example.org/reservation"
                  targetNamespace="http://travel.example.org/reservation">
  <wsdl:types>                                          <!-- (1)! -->
    <xsd:schema targetNamespace="http://travel.example.org/reservation">
      <xsd:element name="reserve">
        <xsd:complexType>
          <xsd:sequence>
            <xsd:element name="flight" type="xsd:string"/>
            <xsd:element name="date" type="xsd:date"/>
          </xsd:sequence>
        </xsd:complexType>
      </xsd:element>
    </xsd:schema>
  </wsdl:types>
  <wsdl:message name="reserveRequest">                  <!-- (2)! -->
    <wsdl:part name="body" element="tns:reserve"/>
  </wsdl:message>
  <wsdl:portType name="BookingPort">                    <!-- (3)! -->
    <wsdl:operation name="Reserve">
      <wsdl:input message="tns:reserveRequest"/>
    </wsdl:operation>
  </wsdl:portType>
</wsdl:definitions>
```

1.  `wsdl:types` embeds a full **XSD** — the same schema language from the
    [XSD chapter](../xsd/index.md). It is literally an `xs:schema` element nested
    inside the WSDL. The payload's structure is defined here, once.
2.  A `message` names the data crossing the wire and points at the schema element
    via `tns:reserve`. `tns` ("this namespace") is a convention: a prefix bound to
    the document's *own* `targetNamespace`, so the WSDL can refer to its own
    definitions.
3.  The `portType` is the abstract interface — operations with inputs and outputs.
    A `binding` then says "do this over SOAP-over-HTTP" (the
    [next section](#the-binding-where-the-wsdl-meets-soap)). The layering is
    deliberate: *what* the service does is separate from *how* it is transported.

!!! note "`tns` and `targetNamespace` point at the same URI"
    Look closely: `xmlns:tns` and `targetNamespace` are the *same* string. That is
    the whole trick — the document defines things into a namespace, then refers to
    them through a prefix bound to that same namespace. It feels circular the first
    time; it is just "name your own things, then use those names".

The verbosity that makes WSDL hard to read by eye is exactly what a structural
renderer cuts through. Running `unxml --wsdl` over the file above collapses the
nested elements into the contract chain — *types → message → portType* — and
renders the embedded `xs:schema` with the same [XSD](../xsd/index.md) formatting:

``` text title="unxml --wsdl booking.wsdl"
wsdl http://travel.example.org/reservation
  ns tns = http://travel.example.org/reservation
  types
    schema http://travel.example.org/reservation
      element reserve
        flight : xsd:string
        date : xsd:date
  message reserveRequest
    part body : tns:reserve
  portType BookingPort
    op Reserve
      in : tns:reserveRequest
```

Read top to bottom, that is the whole service in eight lines: a `reserve` element
(two fields), a `message` that wraps it, and a `portType` operation that takes it
as input. The XML said the same thing in forty lines across four namespaces.

## The binding: where the WSDL meets SOAP

Everything above is *abstract* — it never says how the operation crosses the wire.
That is the `binding`'s job, and it is the actual hinge between the two specs in
this page's title. The `binding` takes the abstract `portType` and pins it to a
concrete protocol; the `service` then says at which URL it lives.

``` xml title="booking.wsdl (the concrete half)" linenums="1"
<wsdl:binding name="BookingSoap" type="tns:BookingPort">   <!-- (1)! -->
  <soap:binding style="document"
                transport="http://schemas.xmlsoap.org/soap/http"/>
  <wsdl:operation name="Reserve">
    <soap:operation soapAction="http://travel.example.org/reserve"/>  <!-- (2)! -->
    <wsdl:input>
      <soap:body use="literal"/>                           <!-- (3)! -->
    </wsdl:input>
  </wsdl:operation>
</wsdl:binding>
<wsdl:service name="BookingService">                       <!-- (4)! -->
  <wsdl:port name="BookingSoap" binding="tns:BookingSoap">
    <soap:address location="https://travel.example.org/booking"/>
  </wsdl:port>
</wsdl:service>
```

1.  `type="tns:BookingPort"` is the join: this binding *is* the abstract
    `portType` from above, now made concrete. `soap:binding` declares the style
    (**document** — see (3)) and that the transport is SOAP over HTTP.
2.  `soap:operation` gives the `Reserve` operation its **SOAPAction** — the value
    that goes in the HTTP `SOAPAction` header, which servers and intermediaries can
    route on without parsing the body.
3.  `soap:body use="literal"` is the modern default: the SOAP `Body` carries the
    `reserve` element from `wsdl:types` *verbatim*, exactly as the schema defines
    it. (The legacy alternative, `rpc/encoded`, synthesized wrapper elements and
    type attributes instead; you will see it in old services, and it is best
    avoided.) This is **why** the `m:reserve` payload in the
    [envelope at the top of this page](#the-envelope) looks the way it does — the
    binding dictated it.
4.  `service` / `port` is the concrete endpoint: `soap:address location` is the URL
    you actually POST the envelope to. So the three layers read as a sentence —
    *portType* is **what**, *binding* is **how**, *service* is **where**.

!!! note "The `soap:` binding prefix is itself version-specific"
    The `soap:` prefix here is `http://schemas.xmlsoap.org/wsdl/soap/` — the
    **SOAP 1.1** WSDL binding. A SOAP 1.2 service uses a different one
    (`…/wsdl/soap12/`), the same versioning-by-namespace idea as the envelopes
    below.

## Two SOAP namespaces in the wild

There are two SOAP envelope namespaces you will meet, and the version is encoded
*entirely* in the namespace URI:

| Version | Envelope namespace |
| --- | --- |
| SOAP 1.1 | `http://schemas.xmlsoap.org/soap/envelope/` |
| SOAP 1.2 | `http://www.w3.org/2003/05/soap-envelope` |

A SOAP 1.2 server rejects a 1.1 envelope not because the elements differ — they
are both called `Envelope`/`Header`/`Body` — but because they are in the *wrong
namespace*. The namespace **is** the version flag. This is a recurring real-world
idiom: [Atom does the same](atom-feeds.md), and so does
[XBRL](xbrl.md).

## Things to note

- Namespaces partition a document by **ownership**: transport vs. infrastructure
  vs. payload, each evolving on its own schedule.
- A schema can be **embedded** (`wsdl:types` wraps an `xs:schema`), not just
  referenced — XSD travels *inside* other vocabularies.
- The `tns` / `targetNamespace` pairing is how a document refers to its own
  definitions by name.
- **Versioning by namespace URI** is a deliberate design choice, not an accident.

Next: [Office documents](office-ooxml-odf.md), where a single file is a *ZIP* of
many XML parts, each with its own forest of namespaces.
