---
title: "Nameserver Access Modes with Encryption Held in Alphanumeric Configuration Keys"
abbrev: "NAMEHACK"
docname: draft-schwartz-dprive-name-signal-latest
category: exp

ipr: trust200902
area: General
workgroup: dprive
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: B. Schwartz
    name: Benjamin M. Schwartz
    organization: Google LLC
    email: bemasc@google.com
 -
    ins: W. Kumari
    name: Warren Kumari
    organization: Google LLC
    email: warren@kumari.net

normative:

informative:



--- abstract

Some recent proposals to the DPRIVE working group rely on the use of SVCB records to provide instructions about how to reach an authoritative nameserver over an encrypted transport.  These proposals will be difficult to deploy until the parent domain's delegation software has been modified to support these records.  As an interim solution for these domains, this draft proposes encoding relevant signals in the child's NS-name.

--- middle

# Conventions and Definitions

{::boilerplate bcp14}

# Background

{{!I-D.draft-schwartz-svcb-dns}} defines how to use SVCB records to describe the secure transport protocols supported by a DNS server.  {{?I-D.draft-ietf-dprive-unauth-to-authoritative}} describes the use of such records on the names of nameservers (the "NS name") to enable opportunistic encryption of recursive-to-authoritative DNS queries.  Resolvers are permitted to fetch SVCB records asynchronously and cache them, resulting in "partial opportunistic encryption": even without an active adversary forcing a downgrade, queries will sometimes be sent in cleartext.  Participating authoritative nameservers and recursive resolvers would have to be modified to make use of these records.

When the child zone is DNSSEC-signed, publishing a SVCB record of this kind is technically sufficient to enable authenticated encryption.  However, in order to support reliable authentication, recursive resolvers would have to revalidate the NS name and resolve its SVCB records on every signed delegation before issuing their intended query.  We call this behavior a "synchronous binding check".

Many validating resolvers might not be willing to enable a "synchronous binding check" behavior, as this would slow down resolution of many existing domains in order to enable a new feature (authenticated encryption) that is not yet used at all.  To enable authenticated encryption without this general performance loss, {{?I-D.draft-rescorla-dprive-adox-latest}} proposes to deliver the SVCB records from the parent, in the delegation response.  This avoids the need for a binding check, at the cost of additionally requiring modifications to the parent nameserver, which must provide these extra records in delegation responses.

Providing these additional records is sufficient to enable "full opportunistic encryption": the transport is always encrypted in the absence of an active adversary.  However, these records are not protected by DNSSEC, so the child can only achieve fully authenticated encryption if the parent also implements fully authenticated encryption or otherwise protects the delivery of these records.

Even if this approach is standardized, many parent zones may not support delivery of SVCB records in delegation responses in the near future.  To enable the broadest use of encrypted transport, we may need an interim solution that can be deployed more easily.

# Proposal

We propose to indicate a nameserver's support for encrypted transports using a signal encoded in its name.  This signal takes two forms: a "flag" and a "menu".

> QUESTION: Do we need both of these forms, or should we drop one?

We note that encoding semantics in DNS labels is a hack, but believe that the privacy benefits outweigh the ick factor.

In either form, the signal helps resolvers to acquire a SVCB RRSet for the nameserver.  Resolvers use this RRSet as specified in {{!I-D.draft-rescorla-dprive-adox-latest}}.

## Flag form

If the NS name's first label is `svcb`, this is regarded as a "flag".  The presence of a flag indicates that the nameserver operator encourages recursive resolvers to perform a synchronous binding check and establish a secure connection before issuing the query.  Resolvers that prefer opportunistic encryption (as in {{?I-D.draft-ietf-dprive-unauth-to-authoritative}}) MAY perform this check asynchronously instead.

The presence of this flag does not guarantee that the corresponding SVCB records are actually present.

Resolver implementors should take care to avoid resolution loops when performing binding checks, with special consideration for misconfigured zones.

## Menu form

If the NS name's first label starts with `svcb--`, the remainder of that label represents a "menu" of connection options, which can be decoded into a SVCB RRSet.  To produce the RRSet, the recipient first splits the remaining characters on each "-" to produce a list of menu items.  Each item is transformed into a SVCB RR with the following components:

* The owner name is the NS name plus the prefix label "_dns".
* The SvcPriority is the item's position in the list (starting at 1)
* The TargetName is the NS name
* The SvcParams are indicated in the registry entry for this menu item ({{iana}}).

For example, the name `svcb--doq-dot.ns3.example.com.` would be decoded to this RRSet:

~~~
$ORIGIN example.com.
_dns.svcb--doq-dot.ns3 IN SVCB 1 svcb--doq-dot.ns3 alpn=doq
                       IN SVCB 2 svcb--doq-dot.ns3 alpn=dot
~~~

Menu items are case-insensitive.  Parsers MUST ignore any menu items that they do not recognize.

## Implementation requirements

Resolvers that implement support for "menu" mode MUST also support the "flag" mode.  Resolvers that support either mode MUST also support {{!I-D.draft-rescorla-dprive-adox-latest}}, and ignore the in-name signal if any SVCB records are included in a delegation response.

When possible, zones SHOULD use SVCB records in the delegation response and omit any in-name signal.

# Security Considerations

NS names received during delegation are not protected by DNSSEC.  Therefore, just like in {{?I-D.draft-rescorla-dprive-adox-latest}}, this scheme only enables authenticated encryption if the parent domain can provide authentication without DNSSEC validation, e.g. using a secure transport or Zone Digest {{?RFC8976}}.

> QUESTION: Do we expect to have parent zones that can provide authenticated NS names but cannot provide authenticated SVCB records in delegation responses?  (Maybe the root, with ZONEMD?)  If not, does this proposal provide enough value?

# Operational Considerations

It is possible that an existing NS name already matches the "flag" pattern.  Such a "false positive flag" will result in a small performance loss due to the unnecessary synchronous binding check, but will not otherwise impair functionality.

If a pre-existing NS name contains the menu pattern, that nameserver will become unreachable by resolvers implementing this specification.  The authors believe that no such nameservers are currently deployed, and such servers are unlikely to be deployed by accident.

# IANA Considerations {#iana}

IANA is requested to create a new registry entitled "Authoritative Server Transport Mnemonics", with the following fields:

* Mnemonic: a sequence of lower-case letters and digits
* SvcParams: a valid SVCB SvcParams set in presentation format

The registry policy is First-Come-First-Served.  Registry entries MUST begin with a letter.

The initial contents (**DO NOT USE**, subject to change) are as follows:

| Mnemonic | SvcParams                        |
| -------- | ---------                        |
| dot      | alpn=dot                         |
| doh      | alpn=h2 dohpath=/dns-query{?dns} |
| doh3     | alpn=h3 dohpath=/dns-query{?dns} |
| doq      | alpn=doq                         |

--- back

# Comparison with related designs

Several other designs have been proposed to encode a transport upgrade signal in an existing record type.

## Indicating DoT support with a name prefix

Section 3.6 of {{?I-D.draft-levine-dprive-signal-02}} discusses using the "xs--" name prefix to indicate support for DNS over TLS.  This is equivalent to a "svcb--dot" label in this formulation.  This draft may be seen as an expansion of that proposal, harmonized with the SVCB-based discovery drafts.

## Encoding the SPKI pin in the leaf label

{{?I-D.draft-bretelle-dprive-dot-spki-in-ns-name}} also proposes to encode a signal in the leaf label.  The signal includes an SPKI pin, for authentication of the TLS connection.

Including an SPKI pin allows authentication of the nameserver without relying on DANE or PKI validation.  However, like this draft, it does not achieve authenticated encryption unless the NS name can be delivered securely during delegation.  It may also create operational challenges when rotating TLS keys, due to the need to update the parent zone.

## Encoding the signal in an additional NS record

It would be possible to encode the signal by adding a special NS record to the RRSet.  This would avoid the need to rename any existing nameservers.  However, this arrangement has different semantics: it is scoped to the entire child zone, rather than a specific nameserver.  It also relies heavily on existing resolvers having robust and performant fallback behavior, which may not be a safe assumption.

(Credit: Paul Hoffman)

## Extending the DS record

{{?I-D.draft-vandijk-dprive-ds-dot-signal-and-pin}} encodes a signal and pin in a DS record by allocating a new fake "signature algorithm" and encoding the TLS SPKI in a DNSKEY record.  This enables fully authenticated encryption (only requiring that the parent zone is signed).  However, it has very limited flexibility for representing different transport configurations, and creates challenges during TLS key rotation.

## Enabling authentication of delegation data

{{?I-D.draft-fujiwara-dnsop-delegation-information-signer}} adds a DS record over the delegation information.  When combined with this draft, this would enable fully authenticated encrypted transport.  However, this approach requires very tight coherence between the child and parent (e.g. when removing a nameserver) that may not be achievable in practice.

{{?I-D.draft-vandijk-dnsop-ds-digest-verbatim}} allows children to push arbitrary authenticated delegation data into the parent.  This could be used to convey SVCB RRSets for the delegation securely.  However, it requires parents to accept a new digest type, and bends the usual DS semantics even further.

# Acknowledgments
{:numbered="false"}

**TODO**
