---
icon: lucide/key-round
---

# SAML — signed assertions and four namespaces

This is the XML you log in with. Every "Sign in with Okta / Entra ID / Ping"
button at work is, underneath, a **SAML** exchange: an identity provider mints a
signed XML document saying *who you are*, and an application trusts it. SAML 2.0
is an OASIS standard from 2005 that never left — enterprise single sign-on
standardized on it, and REST never displaced it there.

It belongs right after [XML Signature](xml-dsig.md) because that page *is* SAML's
trust model: an assertion is worthless unless its signature verifies, and it is
[detached from its issuer](xml-dsig.md#the-namespace-problem-concretely) and
re-parsed by a stranger — exactly the case canonicalization exists for. SAML also
splits its work across **four** namespaces, one per layer, which makes it a clean
final lesson in the [ownership pattern](soap-wsdl.md#things-to-note).

## The assertion

The heart of SAML is the `Assertion`: a statement, made by an issuer, about a
subject, valid for a window.

``` xml title="assertion.xml" linenums="1"
<saml:Assertion xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
                Version="2.0" ID="_a1b2c3d4" IssueInstant="2026-06-21T09:30:00Z">
  <saml:Issuer>https://sso.acme.example/idp</saml:Issuer>     <!-- (1)! -->
  <saml:Subject>
    <saml:NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress"
      >jordan@acme.example</saml:NameID>                      <!-- (2)! -->
    <saml:SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
      <saml:SubjectConfirmationData NotOnOrAfter="2026-06-21T09:35:00Z"
        Recipient="https://app.beancount.example/acs"/>       <!-- (3)! -->
    </saml:SubjectConfirmation>
  </saml:Subject>
  <saml:Conditions NotBefore="2026-06-21T09:29:00Z"
                   NotOnOrAfter="2026-06-21T09:35:00Z">       <!-- (4)! -->
    <saml:AudienceRestriction>
      <saml:Audience>https://app.beancount.example/sp</saml:Audience>
    </saml:AudienceRestriction>
  </saml:Conditions>
  <saml:AuthnStatement AuthnInstant="2026-06-21T09:30:00Z">   <!-- (5)! -->
    <saml:AuthnContext>
      <saml:AuthnContextClassRef
        >urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport</saml:AuthnContextClassRef>
    </saml:AuthnContext>
  </saml:AuthnStatement>
  <saml:AttributeStatement>                                   <!-- (6)! -->
    <saml:Attribute Name="email">
      <saml:AttributeValue>jordan@acme.example</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="groups">
      <saml:AttributeValue>finance</saml:AttributeValue>
      <saml:AttributeValue>admin</saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
```

1.  `Issuer` is who *minted* the statement — the identity provider (IdP). The
    receiver will trust this assertion only if it is signed by the key it has on
    file for this issuer (see [Metadata](#metadata-the-contract) below).
2.  `Subject` / `NameID` is who the assertion is *about*. The `Format` says how to
    read the value — here, an email address.
3.  `SubjectConfirmation` with method **bearer** means "whoever presents this is
    the subject" — so `Recipient` pins the one application allowed to receive it.
    Present the same assertion to a *different* app and it is invalid.
4.  `Conditions` bound validity: a few-minute window (`NotBefore` / `NotOnOrAfter`)
    and an `AudienceRestriction` naming the intended service provider (SP). Between
    them, a leaked assertion is useless at another SP or a minute later.
5.  `AuthnStatement` records *how* and *when* the user authenticated — the
    `AuthnContextClassRef` URN distinguishes a password over TLS from MFA, a
    hardware token, and so on.
6.  `AttributeStatement` carries the claims the SP actually consumes — email,
    group membership (note the **multi-valued** `groups`), department, whatever was
    agreed. This is the payload; everything above is envelope and trust.

## The response that carries it

In the everyday browser-SSO flow, the assertion travels inside a
`samlp:Response` — and here the second namespace appears.

``` xml title="response.xml" linenums="1"
<samlp:Response xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
                xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
                Version="2.0" ID="_resp9f3" IssueInstant="2026-06-21T09:30:01Z"
                Destination="https://app.beancount.example/acs"
                InResponseTo="_req42">                        <!-- (1)! -->
  <saml:Issuer>https://sso.acme.example/idp</saml:Issuer>
  <samlp:Status>                                              <!-- (2)! -->
    <samlp:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success"/>
  </samlp:Status>
  <saml:Assertion ID="_a1b2c3d4" Version="2.0" IssueInstant="2026-06-21T09:30:00Z">
    <!-- …the assertion from above, signed… -->
  </saml:Assertion>
</samlp:Response>
```

1.  The **protocol** namespace `samlp:` owns the request/response machinery;
    `saml:` owns the assertion inside it. Two namespaces, two jobs:
    `InResponseTo` ties this back to the SP's original `AuthnRequest`, and
    `Destination` must match the SP's assertion-consumer URL — both anti-forgery
    checks the SP makes before it ever looks at the claims.
2.  `samlp:Status` is the protocol-level outcome; `…:status:Success` here. A
    failed login carries a different status URN and no assertion.

!!! note "This part is *not* SOAP"
    Unlike [SOAP](soap-wsdl.md), the dominant *Web Browser SSO* profile uses no
    SOAP at all — the IdP base64-encodes this `Response` into a hidden field and
    has the browser auto-POST it to the SP. SOAP only shows up in SAML's
    back-channels: Artifact Resolution and the WS-Trust security token service.
    SAML is XML-as-document, not XML-as-RPC.

## The signature

That assertion is only trustworthy because it is **signed**. A `ds:Signature` —
the exact [XML-DSig](xml-dsig.md) vocabulary from the previous page — sits
*enveloped* inside the `Assertion` (or the `Response`, or both):

``` xml title="inside the assertion"
<saml:Assertion ID="_a1b2c3d4" …>
  <ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
    <ds:SignedInfo>
      <ds:CanonicalizationMethod Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
      <ds:Reference URI="#_a1b2c3d4">…</ds:Reference>          <!-- (1)! -->
    </ds:SignedInfo>
    <ds:SignatureValue>MC0CFFrV…==</ds:SignatureValue>
  </ds:Signature>
  <saml:Issuer>…</saml:Issuer>
  …
</saml:Assertion>
```

1.  `Reference URI="#_a1b2c3d4"` signs the assertion by its `ID` — not the whole
    document — so the signed assertion stays valid when the SP lifts it out of the
    `Response`. And the algorithm is **Exclusive C14N**, for exactly the reason the
    [XML-DSig page warned about](xml-dsig.md): the assertion is detached from the
    IdP's document and re-embedded in the SP's, so only the namespaces it *uses*
    may travel with it. Inclusive C14N here is the classic SAML interop bug.

## Metadata: the contract

How does the SP know *which* key signs the IdP's assertions, and where to send
users to log in? The two parties exchange **metadata** — a `md:EntityDescriptor`
that is, in effect, [WSDL](soap-wsdl.md#wsdl-the-contract-that-imports-a-schema)
for identity: a machine-readable description of endpoints, certificates, and
supported bindings.

``` xml title="idp-metadata.xml" linenums="1"
<md:EntityDescriptor xmlns:md="urn:oasis:names:tc:SAML:2.0:metadata"
                     xmlns:ds="http://www.w3.org/2000/09/xmldsig#"
                     entityID="https://sso.acme.example/idp">   <!-- (1)! -->
  <md:IDPSSODescriptor protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
    <md:KeyDescriptor use="signing">                            <!-- (2)! -->
      <ds:KeyInfo><ds:X509Data>
        <ds:X509Certificate>MIIBIjANBg…</ds:X509Certificate>
      </ds:X509Data></ds:KeyInfo>
    </md:KeyDescriptor>
    <md:SingleSignOnService                                     <!-- (3)! -->
      Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
      Location="https://sso.acme.example/idp/sso"/>
  </md:IDPSSODescriptor>
</md:EntityDescriptor>
```

1.  `entityID` is the stable name both sides use for this IdP — the same string
    that appears as the assertion's `Issuer`. That is the link: the SP matches
    `Issuer` against the `entityID` whose metadata it trusts.
2.  The signing **certificate**. This is the whole point — the SP verifies every
    assertion against this key. Hand-waving the metadata exchange is hand-waving
    the trust.
3.  `SingleSignOnService` publishes *where* and over *which binding* to start a
    login. An SP's metadata mirrors this with `AssertionConsumerService` endpoints.

## The four namespaces

SAML is a clean example of one document, four vocabularies, each owning a layer:

| Prefix | Namespace URI | Owns |
| --- | --- | --- |
| `saml` | `urn:oasis:names:tc:SAML:2.0:assertion` | the claims: subject, conditions, attributes |
| `samlp` | `urn:oasis:names:tc:SAML:2.0:protocol` | the request/response envelope and status |
| `ds` | `http://www.w3.org/2000/09/xmldsig#` | the signature ([XML-DSig](xml-dsig.md)) |
| `md` | `urn:oasis:names:tc:SAML:2.0:metadata` | the published contract: endpoints, certs, bindings |

Encrypted assertions add a fifth, `xenc` (`…/xmlenc#`), but the four above are
the ones you meet in every flow.

## Things to note

- The **assertion / protocol split** (`saml:` vs `samlp:`) is deliberate: the
  *same* signed assertion can ride in a browser-POST `Response`, an artifact, or a
  SOAP back-channel. Separating the claims from the message that carries them is a
  namespace decision.
- **Trust rests entirely on [XML-DSig](xml-dsig.md).** A SAML assertion is only as
  good as its signature — and because the SP lifts it out of one document and into
  another, Exclusive C14N is not optional trivia here, it is the difference between
  working and not.
- **Metadata is the contract** — identity's answer to [WSDL](soap-wsdl.md):
  `entityID`, signing certs, endpoints, and bindings, exchanged out of band so two
  organizations can trust each other's XML.
- The `Conditions` and bearer `Recipient` are what make it *safe* to hand a token
  to the user's own browser: audience, expiry, and recipient together pin a stolen
  assertion to one SP and a few minutes.

Next: [GPX and KML](geo-gpx-kml.md) — back to friendlier ground, with two
geospatial vocabularies and two different ways of allowing extensions.
