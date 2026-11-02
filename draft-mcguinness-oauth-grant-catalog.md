---
title: "OAuth Authorization Grant Catalog"
abbrev: "OAuth Grant Catalog"
category: std

docname: draft-mcguinness-oauth-grant-catalog-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - OAuth
 - authorization grant
 - JWT authorization grant
 - cross-domain
 - metadata
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-oauth-grant-catalog"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-grant-catalog/draft-mcguinness-oauth-grant-catalog.html"

author:
 -
    fullname: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  ID-JAG: I-D.draft-ietf-oauth-identity-assertion-authz-grant-06
  JWT-DPOP: I-D.draft-parecki-oauth-jwt-dpop-grant-02
  RFC3986:
  RFC6749:
  RFC6750:
  RFC7493:
  RFC7515:
  RFC7519:
  RFC7521:
  RFC7523:
  RFC7800:
  RFC8259:
  RFC8414:
  RFC8693:
  RFC8705:
  RFC8707:
  RFC8725:
  RFC9110:
  RFC9111:
  RFC9396:
  RFC9449:
  RFC9493:
  RFC9728:
  OpenID.Enterprise:
    title: "OpenID Connect Enterprise Extensions 1.0 - draft 01"
    target: https://openid.net/specs/openid-connect-enterprise-extensions-1_0.html
    author:
      -
        ins: D. Hardt
        name: Dick Hardt
      -
        ins: K. McGuinness
        name: Karl McGuinness
    date: 2025-09

informative:
  RFC6838:
  RFC6839:
  RFC8126:
  RFC9700:
  RAR-METADATA: I-D.ietf-oauth-rar-metadata-remediation
  AUTHORITY-METADATA: I-D.zehavi-oauth-authority-metadata
  CIMD: I-D.ietf-oauth-client-id-metadata-document

--- abstract

This specification defines the OAuth Authorization Grant Catalog: an
authenticated, relationship-specific description of the authorization
grants that a Resource Authorization Server is prepared to accept from a
Grant Issuer. A Grant Issuer is an authorization server, such as an
identity provider, that issues grants for redemption at the Resource
Authorization Server. For each eligible target tenant, a catalog
describes the protected resources, scope values, authorization details
types, claims, subject identifier formats, and grant binding methods that
the Resource Authorization Server accepts. The Grant Issuer uses the
catalog as input when authoring the policy under which it issues grants;
the catalog is not an authorization decision. The catalog data model is
independent of grant encoding and of retrieval protocol. This
specification maps the data model only to the Identity Assertion JWT
Authorization Grant (ID-JAG), and defines an OAuth Retrieval Binding in
which the Grant Issuer obtains a catalog access token and retrieves its
complete catalog in one response.

--- middle

# Introduction {#introduction}

Several OAuth specifications define authorization grants that one
authorization server issues for a client to present to another
authorization server, typically in another trust domain. The Identity
Assertion JWT Authorization Grant (ID-JAG) {{ID-JAG}}, which an identity
provider issues, is one example; other profiles of the JWT authorization
grant {{RFC7523}} follow the same pattern. In each case the issuing
authorization server, called the Grant Issuer in this specification,
decides which tenants of the receiving authorization server, protected
resources, scope values, authorization details types {{RFC9396}},
claims, subject identifiers, and grant bindings to place in a grant. The
receiving authorization server, called the Resource Authorization
Server, rejects a grant that falls outside what it accepts, so the Grant
Issuer's configuration has to match the Resource Authorization Server's
configuration for that Grant Issuer.

Public metadata does not convey this information. OAuth 2.0
Authorization Server Metadata {{RFC8414}} and OAuth 2.0 Protected
Resource Metadata {{RFC9728}} are retrieved without authentication, are
generally not specific to a tenant, and describe what a server
implements rather than what it accepts from a particular Grant Issuer.
{{Section 9.7 of ID-JAG}} notes that advertising issuer-specific trust
relationships in publicly accessible metadata can disclose federation
topology, business relationships, or tenant configuration. That section
also prohibits a Resource Authorization Server from using the
`authorization_grant_profiles_supported` metadata member to disclose
issuer allow lists or other profile-specific trust relationships. In
current deployments, administrators of a Grant Issuer read a Resource
Authorization Server's documentation to learn what that server accepts,
and reproduce those values by hand.

This specification defines how a Resource Authorization Server describes
in advance, to an authenticated Grant Issuer, the grants it is prepared
to accept from that Grant Issuer, and how the Grant Issuer uses that
description, called the catalog in this specification. A catalog is
input to policy authoring, not an authorization decision: the Resource
Authorization Server applies its current policy to every grant it
processes, whether or not the grant was issued using a catalog.

Other specifications can apply the catalog to additional Grant
Profiles, define additional Retrieval Bindings, and extend the data
model ({{extensibility}}).

## Motivating Example {#motivating-example}

An enterprise uses its identity provider for single sign-on to a
software-as-a-service application, in which it has two tenants, Acme
and Acme Europe. It wants an AI assistant, a client registered at the
application, to use ID-JAG to call the application's reports API on
behalf of employees.

Today, an administrator of the identity provider reads the application's
documentation and enters the following by hand: the application's issuer
identifier and the identifiers of both tenants; the resource identifier
of the reports API; the scope values `reports.read` and
`reports.export`, and the rule that the application accepts either in a
grant but not both; the requirement for a `department` claim with
`reports.export`; and the subject identifier formats and grant bindings
that the application accepts. A mistake surfaces only when the
application rejects an ID-JAG, after the ID-JAG has disclosed its claims
to the client.

With a catalog, the identity provider retrieves, under its relationship
with the application, a Relationship View that lists both tenants with
their target tenant identifiers and contains entries that state these
values ({{example-alternatives}} shows entries for the reports API). The
administrator approves values from those entries instead of typing them,
for example `reports.read` for both tenants and `reports.export` for
Acme only. When the application later changes an entry, the identity
provider detects the new revision and asks for review instead of
continuing under the old approval.

The rest of the integration is unchanged: establishing the relationship
(the Onboarding Arrangement), registering the client at the application
and mapping its client identifier, learning any restrictions the
application applies to that particular client, deciding which employees
and clients may use the integration, and the application's runtime
authorization decisions.

## Relationship to Other Metadata {#other-metadata}

The catalog adds relationship-specific eligibility to existing metadata.
It does not replace, copy, or override that metadata, and each source
describes something different:

| Source | Describes |
| --- | --- |
| Authorization Server Metadata {{RFC8414}} | Protocol capabilities of the Resource Authorization Server |
| Protected Resource Metadata {{RFC9728}} | Capabilities of one protected resource |
| Authorization details types metadata {{RAR-METADATA}} | The structure of an authorization details type |
| Authority metadata {{AUTHORITY-METADATA}} | What a scope value or authorization details type authorizes |
| Grant Catalog | What the Resource Authorization Server accepts from one Issuer Relationship |
| Grant Issuer policy | The subset that an administrator of the Grant Issuer approves |
| Runtime authorization | The decision for one request |
{: title="Sources of Information Used to Issue a Grant"}

{{composition}} defines how a Grant Issuer combines these sources.

## Applicability {#applicability}

This specification applies to authorization grants that one
authorization server, the Grant Issuer, issues for a client to present
to another authorization server, the Resource Authorization Server,
where the Grant Issuer determines the content of each grant and the
Resource Authorization Server validates it at redemption. JWT
authorization grants {{RFC7523}} and their profiles have this form. It
does not apply to grants that an authorization server issues for
redemption at itself, such as authorization codes, device codes, and
refresh tokens, because no other party determines what such a grant
contains.

This specification defines a catalog data model independent of grant
encoding ({{data-model}}), with an Authorization Envelope ({{envelope}})
and processing rules ({{processing}}) stated in terms of the inputs a
grant conveys. Each Grant Profile Definition specifies how grants
of its Grant Profile convey those inputs ({{profile-definitions}}). This
specification provides a Grant Profile Definition only for ID-JAG
({{id-jag}}), using the conventions for JWT authorization grants
({{jwt-grants}}). A Retrieval
Binding authenticates the relationship and delivers the catalog
({{relationship-authentication}}); this specification defines an OAuth
Retrieval Binding ({{oauth-binding}}).

Both parties to a catalog are identified explicitly. The Grant Issuer
is identified by its issuer identifier and, when it is multi-tenant, an
issuer tenant identifier ({{relationship-identity}}). A Resource
Authorization Server tenant is identified by the Resource Authorization
Server's issuer identifier and, unless that identifier alone identifies
the tenant, a target tenant identifier ({{tenant-object}}).

## Out of Scope {#out-of-scope}

The following are outside the scope of this specification:

* Enumeration or registration of application clients, and establishment
  of client identifier mappings ({{clients}}). Views specific to one
  application client are an open issue ({{open-issues}}).

* Constraints on the values within an authorization details object. The
  catalog identifies permitted authorization details types only
  ({{entry-authorization-details}}).

* Paging, change notification, delta retrieval, and push delivery of
  catalogs.

* Automatic reconciliation of Grant Issuer policy with a changed entry
  ({{reconciliation}}).

* A mechanism by which a client discovers whether a grant from a
  particular Grant Issuer would be accepted ({{Section 9.7 of ID-JAG}}).

* The process by which a Resource Authorization Server and a Grant
  Issuer agree to establish an Issuer Relationship
  ({{relationship-authorization}}).

# Conventions and Definitions {#conventions}

{::boilerplate bcp14-tagged}

This specification uses the terms "access token", "authorization grant",
"authorization server", "client", "resource server", and "scope" defined
by OAuth 2.0 {{RFC6749}}; "JSON Web Token (JWT)" and "claim" defined by
{{RFC7519}}; "authorization details" and "authorization details type"
defined by {{RFC9396}}; "resource indicator" as used by {{RFC8707}};
"protected resource" as used by {{RFC9728}}; and "trust domain" as
defined in {{Section 2.2 of ID-JAG}}. In this specification, "grant"
means an authorization grant that a Grant Issuer issues for a client to
present to a Resource Authorization Server.

This specification defines the following terms:

tenant:
: An administrative boundary within an issuer, such as a customer
  organization, that scopes subjects, clients, and policy
  ({{Section 2.2 of ID-JAG}}). An issuer is multi-tenant when one issuer
  identifier hosts several tenants ({{Section 6.1 of ID-JAG}}).

tenant identifier:
: A string that, together with an issuer identifier, identifies exactly
  one tenant of that issuer. A tenant identifier is scoped by the issuer
  identifier, is not reassigned by the issuer to a different tenant, and
  is compared using simple string comparison
  ({{Section 6.2.1 of RFC3986}}). The issuer tenant identifier of a
  Grant Issuer and the target tenant identifier of a Resource
  Authorization Server tenant are tenant identifiers.

Grant Issuer:
: An authorization server that consumes a Grant Catalog and issues
  grants that a client presents to a Resource Authorization Server. In
  ID-JAG, the Grant Issuer is the IdP Authorization Server
  ({{Section 2.1 of ID-JAG}}).

Resource Authorization Server:
: An authorization server that accepts grants issued by a Grant Issuer
  and publishes a Grant Catalog. This specification uses the term as
  {{Section 2.1 of ID-JAG}} does, for any Grant Profile.

Grant Profile:
: A profile of an authorization grant, identified by a URI as used in
  the `authorization_grant_profiles_supported` metadata member
  ({{Section 7.2 of ID-JAG}}), such as
  `urn:ietf:params:oauth:grant-profile:id-jag`.

Grant Profile Definition:
: A specification that defines how catalog entries apply to a Grant
  Profile ({{profile-definitions}}).

Issuer Relationship:
: The pair of a Grant Issuer's issuer identifier and, when that issuer
  is multi-tenant, an issuer tenant identifier, that identifies the
  Grant Issuer side of a catalog relationship
  ({{relationship-identity}}).

Onboarding Arrangement:
: The agreement, established outside this protocol, under which a
  Resource Authorization Server discloses catalog content to an Issuer
  Relationship and determines the Resource Authorization Server tenants
  eligible for it ({{relationship-authorization}}).

Relationship View:
: The complete set of Tenant Objects, and the Grant Catalogs they
  reference, that a Resource Authorization Server makes available to one
  Issuer Relationship at a point in time.

Tenant Object:
: An object that binds one Resource Authorization Server tenant eligible
  for an Issuer Relationship to exactly one Grant Catalog
  ({{tenant-object}}).

Grant Catalog:
: A complete set of entries identified by a stable identifier
  ({{grant-catalog}}).

Entry:
: A member of a Grant Catalog that describes the grants of one Grant
  Profile that the Resource Authorization Server accepts for one
  protected resource ({{entry}}).

Catalog-Governed Input:
: A part of a grant whose permitted values an entry states: the
  protected resource, the scope values, the authorization details types,
  the Catalog-Governed Claims, the subject identifier, and the key
  binding.

Authorization Envelope:
: The set of grants that an entry, or an approved subset of an entry,
  permits ({{envelope}}).

Retrieval Binding:
: A protocol by which a Grant Issuer authenticates an Issuer
  Relationship to a Resource Authorization Server and retrieves its
  Relationship View ({{binding-requirements}}).

# Protocol Overview {#overview}

{{fig-overview}} shows the steps by which a Grant Issuer obtains a
Relationship View and uses it to issue grants.

~~~ ascii-art
+---------+                                   +---------------+
|         |---(A) Retrieve public metadata -->|               |
|         |                                   |   Resource    |
|         |---(B) Authenticate relationship ->| Authorization |
|         |                                   |    Server     |
|         |<--(C) Relationship View ----------|               |
|  Grant  |                                   +---------------+
| Issuer  |                                           ^
|         |   (D) Validate, compose, and install      |
|         |   (E) Administrator approves a subset     |
|         |                                           |
|         |                  +--------+               |
|         |---(F) Grant ---->| Client |---(G) Grant -+
+---------+                  +--------+
~~~
{: #fig-overview title="Protocol Overview"}

(A)
: The Grant Issuer retrieves the Resource Authorization Server's public
  metadata, which includes the location of the catalog for a Retrieval
  Binding.

(B)
: The Grant Issuer authenticates an Issuer Relationship to the Resource
  Authorization Server using a Retrieval Binding. The Resource
  Authorization Server verifies that an Onboarding Arrangement
  authorizes that relationship ({{relationship-authentication}}).

(C)
: The Resource Authorization Server returns the Relationship View: every
  Tenant Object eligible for the Issuer Relationship, and each Grant
  Catalog those Tenant Objects reference.

(D)
: The Grant Issuer validates the Relationship View, composes each entry
  with public metadata, and installs the result as input to policy
  authoring ({{processing}}).

(E)
: An administrator of the Grant Issuer approves a subset of one or more
  entries for use by its clients and subjects ({{narrowing}}).

(F)
: When a client requests a grant, the Grant Issuer authenticates the
  client and the subject, applies its policy, and issues a grant that is
  within the approved subset of one entry ({{envelope}}).

(G)
: The client presents the grant to the Resource Authorization Server,
  which validates it and applies its current policy as specified by the
  Grant Profile.

Steps (A) through (E) occur when the Grant Issuer is configured and when
it refreshes the catalog, not for each grant.

# Catalog Data Model {#data-model}

The objects in this section describe what a Resource Authorization
Server accepts in grants from one Issuer Relationship. Every Retrieval
Binding carries them unchanged ({{binding-requirements}}), and a Grant
Profile Definition states how each member applies to grants of its
Grant Profile ({{profile-definitions}}).

## Common Conventions {#model-conventions}

Every object defined in this section is a JSON object {{RFC8259}}. A
document that carries these objects MUST conform to I-JSON {{RFC7493}}.
A Grant Issuer MUST ignore members it does not understand
({{extensibility}}).

The following members are used by several objects:

`id`:
: A stable identifier for the object. The value is a non-empty JSON
  string of at most 255 octets when encoded in UTF-8. Identifiers are
  opaque and case-sensitive, and are compared using simple string
  comparison ({{Section 6.2.1 of RFC3986}}). The length limit bounds the
  identifiers that a Grant Issuer retains with each approval
  ({{reconciliation}}), and {{validation}} specifies the effect of an
  identifier that exceeds it. The scope within which an identifier is
  unique is stated in {{identifiers}}.

`display`:
: OPTIONAL. A JSON string containing a short, human-readable name for
  the object, intended for presentation to an administrator.

`description`:
: OPTIONAL. A JSON string containing a human-readable explanation of the
  object, such as what a scope value permits, intended for an
  administrator deciding what to approve.

The values of `display` and `description` have no effect on processing.
Language negotiation for these values is outside the scope of this
specification.

Where this specification describes a member as an array of unique
values, the array MUST NOT contain the same value twice. Members that
carry protocol values, such as `issuer`, `target_tenant`, `profile`,
`resource`, `scope`, `type`, `claim`, and the names in `formats` and
`methods`, are compared using simple string comparison
({{Section 6.2.1 of RFC3986}}).

## Tenant Object {#tenant-object}

A Tenant Object binds one Resource Authorization Server tenant that is
eligible for an Issuer Relationship to the Grant Catalog that applies to
it. A Tenant Object has the following members:

`id`:
: REQUIRED. The identifier of the Tenant Object ({{model-conventions}}).
  A Tenant Object identifier identifies one logical Resource
  Authorization Server tenant. The Resource Authorization Server MUST
  NOT use the identifier of a Tenant Object for a different tenant in
  any later Relationship View for the same Issuer Relationship.

`display`:
: OPTIONAL. See {{model-conventions}}.

`description`:
: OPTIONAL. See {{model-conventions}}.

`target_tenant`:
: OPTIONAL. A JSON string containing the exact tenant identifier
  ({{conventions}}) of the Resource Authorization Server tenant. A grant
  issued for this tenant conveys this value as its target tenant, in the
  form the Grant Profile Definition specifies; a JWT authorization grant
  conveys it in the `aud_tenant` claim ({{jwt-grants}}). The
  Resource Authorization Server MUST include this member unless its
  issuer identifier alone identifies the target tenant. A Relationship
  View MUST NOT contain more than one Tenant Object without this member,
  or two Tenant Objects with the same value of this member.

`grant_catalog`:
: REQUIRED. The `id` of the Grant Catalog that applies to this tenant.

The requirement to include `target_tenant` ensures that every target
tenant an administrator selects has an explicit representation in the
grant. The Grant Issuer MUST NOT derive a target tenant identifier from
the Tenant Object `id`, from the number of Tenant Objects in a
Relationship View, or from the form of an issuer identifier. A Tenant
Object that contains `target_tenant` is usable only with entries whose
Grant Profile can convey a target tenant; the Grant Issuer does not
issue grants of other Grant Profiles for that Tenant Object
({{permitted}}).

Each Tenant Object references exactly one complete Grant Catalog, and
several Tenant Objects can reference the same Grant Catalog. Tenant
Objects for tenants that differ in any Catalog-Governed Input reference
different Grant Catalogs. The data model has no defaults, inheritance,
partial overrides, or merge rules.

A change to `target_tenant` or `grant_catalog` in a Tenant Object whose
`id` is unchanged is a change to the runtime target or to the
eligibility of that tenant, respectively, and affects Grant Issuer
policy as described in {{reconciliation}}.

The following is an example of a Tenant Object:

~~~ json
{
  "id": "tenant-123",
  "display": "Acme",
  "target_tenant": "customer-acme",
  "grant_catalog": "standard"
}
~~~

## Grant Catalog {#grant-catalog}

A Grant Catalog is a complete set of entries. A Retrieval Binding
delivers a Grant Catalog whole; a Grant Catalog is never paged or
assembled from parts. A Grant Catalog has the following members:

`issuer`:
: REQUIRED. The issuer identifier {{RFC8414}} of the Resource
  Authorization Server. Every grant issued using an entry of the Grant
  Catalog has this value as its audience.

`id`:
: REQUIRED. The identifier of the Grant Catalog ({{model-conventions}}).

`display`:
: OPTIONAL. See {{model-conventions}}.

`description`:
: OPTIONAL. See {{model-conventions}}.

`entries`:
: REQUIRED. A JSON array of entries ({{entry}}). An empty array
  indicates that the Resource Authorization Server accepts no grants for
  the tenants whose Tenant Objects reference this Grant Catalog.

A Grant Catalog can contain entries for several Grant Profiles.

## Entry {#entry}

An entry describes the grants of one Grant Profile that the Resource
Authorization Server accepts for one protected resource. An entry has
the following members:

`id`:
: REQUIRED. The identifier of the entry ({{model-conventions}}).

`revision`:
: REQUIRED. A JSON string that identifies the version of the entry's
  normative semantics ({{revision}}). The value is at most 255 octets
  when encoded in UTF-8.

`display`:
: OPTIONAL. See {{model-conventions}}.

`description`:
: OPTIONAL. See {{model-conventions}}.

`profile`:
: REQUIRED. A JSON string containing the identifier of the Grant
  Profile of the grants the entry describes.

`resource`:
: REQUIRED. A JSON string containing the resource indicator
  ({{Section 2 of RFC8707}}) of the protected resource the entry
  describes. The value is an absolute URI {{RFC3986}} and MUST NOT
  contain a fragment component.

`scope`:
: OPTIONAL. A JSON object describing the scope values the entry permits
  ({{entry-scope}}). If this member is absent, grants issued using the
  entry convey no scope values.

`authorization_details`:
: OPTIONAL. A JSON object describing the authorization details types the
  entry permits ({{entry-authorization-details}}). If this member is
  absent, grants issued using the entry convey no authorization details.

`claims`:
: OPTIONAL. A non-empty JSON array of claim objects ({{entry-claims}}).
  If this member is absent, the entry permits no Catalog-Governed
  Claims.

`sub_id`:
: OPTIONAL. A JSON object describing the subject identifier formats the
  entry permits ({{entry-sub-id}}). If this member is absent, grants
  issued using the entry convey no subject identifier of the kind this
  member describes.

`grant_binding`:
: OPTIONAL. A JSON object describing the grant binding methods the entry
  permits ({{entry-grant-binding}}). If this member is absent, grants
  issued using the entry are not bound.

The `profile` member identifies the Grant Profile, not the grant type
used to redeem the grant. The JWT bearer grant type
`urn:ietf:params:oauth:grant-type:jwt-bearer` {{RFC7523}} is shared by
several Grant Profiles, so it cannot identify the grants an entry
describes.

Each entry describes exactly one protected resource. A Grant Catalog can
contain entries for several resources, and several entries can name the
same resource when they describe different grants ({{combinations}}). A
client that needs access to several resources obtains a separate grant
for each.

### Requirement Levels {#requirement}

The `scope`, `authorization_details`, `sub_id`, and `grant_binding`
members of an entry each contain a `requirement` member, called the
presence requirement, that states whether a grant conveys the input at
all. Scope value objects, authorization details type objects, and claim
objects each contain an OPTIONAL `requirement` member, called the item
requirement, that states whether a grant conveys that item. The value of
a `requirement` member is the JSON string `required` or `optional`. An
absent item requirement is treated as `optional`.

| Requirement | Value "required" | Value "optional" |
| --- | --- | --- |
| Presence: is the input conveyed? | Every grant issued using the entry conveys the input with at least one permitted item | A grant can omit the input |
| Item: is the item conveyed? | Every grant issued using the entry conveys the item | A grant can convey the item |
{: title="Requirement Levels"}

The two levels state separate decisions. For a member with items A and
B, they permit the following selections:

| Presence | Item Requirements | Permitted Selections |
| --- | --- | --- |
| `optional` | A and B `optional` | None, A, B, or both |
| `required` | A and B `optional` | A, B, or both |
| `required` | A `required`, B `optional` | A, or both |
{: title="Selections Permitted by the Two Levels"}

A member whose presence requirement is `optional` MUST NOT contain an
item whose item requirement is `required`. The `formats` of the
`sub_id` member and the `methods` of the `grant_binding` member are
alternatives and have no item requirement: a presence requirement of
`required` means that a grant uses one of them.

An item requirement expresses conjunction only: an item that is always
present. See {{combinations}} for the rules governing combinations of
items.

### Scope {#entry-scope}

The `scope` member of an entry is a JSON object with the following
members:

`requirement`:
: REQUIRED. The presence requirement ({{requirement}}).

`values`:
: REQUIRED. A non-empty JSON array of scope value objects.

A scope value object has the following members:

`scope`:
: REQUIRED. A JSON string containing one scope token, as defined in
  {{Section 3.3 of RFC6749}}.

`requirement`:
: OPTIONAL. The item requirement ({{requirement}}).

`display`:
: OPTIONAL. See {{model-conventions}}.

`description`:
: OPTIONAL. See {{model-conventions}}.

### Authorization Details Types {#entry-authorization-details}

The `authorization_details` member of an entry is a JSON object with the
following members:

`requirement`:
: REQUIRED. The presence requirement ({{requirement}}).

`types`:
: REQUIRED. A non-empty JSON array of authorization details type
  objects.

An authorization details type object has the following members:

`type`:
: REQUIRED. A JSON string containing an authorization details type
  value ({{Section 2 of RFC9396}}).

`requirement`:
: OPTIONAL. The item requirement ({{requirement}}). When it is
  `required`, every grant issued using the entry conveys at least one
  authorization details object of this type.

`documentation_uri`:
: OPTIONAL. A JSON string containing an absolute URI, using the
  `https` scheme, of human-readable documentation for the type.

`display`:
: OPTIONAL. See {{model-conventions}}.

`description`:
: OPTIONAL. See {{model-conventions}}.

An entry constrains the `type` values of the authorization details
objects that a grant conveys, not the other members of those objects. A
grant can convey more than one authorization details object of a
permitted type. The Grant Issuer applies its policy to complete
authorization details objects, and the Resource Authorization Server
validates their contents at redemption. The catalog therefore provides
discovery of authorization details types, not a language for
constraining their values.

The catalog does not carry schemas for authorization details types.
{{rar-metadata}} describes the optional use of authorization details
types metadata {{RAR-METADATA}}.

### Claims {#entry-claims}

An entry distinguishes two kinds of claims that a grant conveys:

Reserved claims:
: Claims whose presence and values the Grant Profile or other members of
  the catalog determine, such as the issuer, audience, or target tenant
  of a grant, and claims that the Grant Profile requires in every grant.
  The Grant Profile Definition lists them.

Catalog-Governed Claims:
: All other claims, including claims that the Grant Profile defines but
  does not require, such as an authentication context claim or an email
  address, and claims such as a department or group membership. A grant
  issued using an entry conveys a Catalog-Governed Claim only if the
  entry lists that claim.

Every claim that a grant issued using an entry conveys is therefore
either a reserved claim of its Grant Profile or a claim that the entry
lists.

The `claims` member of an entry is an array of claim objects. A claim
object has the following members:

`claim`:
: REQUIRED. A JSON string naming a claim, in the namespace that the
  Grant Profile Definition specifies. For JWT authorization grants, the
  value is a JWT claim name ({{Section 4 of RFC7519}}).

`requirement`:
: OPTIONAL. The item requirement ({{requirement}}).

`definition_uri`:
: OPTIONAL. A JSON string containing an absolute URI, using the
  `https` scheme, that references the definition of the claim.

`display`:
: OPTIONAL. See {{model-conventions}}.

`description`:
: OPTIONAL. See {{model-conventions}}.

The `claim` member of a claim object MUST NOT be a reserved claim name
of the entry's Grant Profile. Listing a claim does not change the
syntax or semantics that the Grant Profile or the claim's definition
specifies for it. Claim objects carry no constraints on claim values: an
entry that lists an authentication context claim as `required` requires
the claim to be present, not a particular value. The `definition_uri`
member is descriptive and does not establish the truth or provenance of
a claim.

The Grant Issuer decides whether it releases a claim value. If an entry
requires a claim that the Grant Issuer's policy does not permit it to
release for a subject, the Grant Issuer MUST NOT issue a grant using
that entry for that subject.

### Subject Identifier {#entry-sub-id}

The `sub_id` member of an entry is a JSON object with the following
members:

`requirement`:
: REQUIRED. The presence requirement ({{requirement}}).

`formats`:
: REQUIRED. A non-empty JSON array of unique Subject Identifier Format
  names, each registered in the "Security Event Identifier Formats"
  registry ({{Section 8.1 of RFC9493}}), such as `email` or
  `saml-nameid` ({{Section 3.2.1 of ID-JAG}}).

When `requirement` is `required`, every grant issued using the entry
conveys a subject identifier {{RFC9493}} in one of the listed formats.
When `requirement` is `optional`, a grant issued using the entry either
conveys no such subject identifier or conveys one in one of the listed
formats. This member describes a subject identifier conveyed in addition
to the subject of the grant; for JWT authorization grants, it is the
`sub_id` claim ({{jwt-grants}}).

### Grant Binding {#entry-grant-binding}

The `grant_binding` member of an entry is a JSON object with the
following members:

`requirement`:
: REQUIRED. The presence requirement ({{requirement}}).

`methods`:
: REQUIRED. A non-empty JSON array of unique grant binding method names,
  each registered in the "OAuth Grant Catalog Grant Binding Methods"
  registry ({{iana-binding-methods}}).

This specification defines the following grant binding methods:

`dpop`:
: The grant is bound to a DPoP key {{RFC9449}}, and the client
  demonstrates possession of that key when presenting the grant.

`mtls`:
: The grant is bound to an X.509 certificate {{RFC8705}}, and the client
  presents that certificate when presenting the grant.

The Grant Profile Definition states how each method is represented in a
grant; {{jwt-grants}} does so for JWT authorization grants.

When `requirement` is `required`, every grant issued using the entry is
bound using one of the listed methods. When `requirement` is
`optional`, a grant issued using the entry is either unbound or bound
using one of the listed methods.

An entry does not create a binding. The Grant Issuer binds a grant only
to a key or certificate whose possession the client has demonstrated,
and selects the binding as the Grant Profile Definition specifies. Where
the Grant Profile allows a choice of method, the Grant Issuer chooses a
method that the entry permits. The Grant Issuer MUST NOT issue a grant
using an entry in either of the following cases:

* the entry requires binding and the client has not demonstrated
  possession using any listed method; or

* the Grant Profile requires the Grant Issuer to bind the grant, and the
  entry permits none of the methods that would satisfy that requirement.

Client authentication to the Resource Authorization Server and
sender-constraining of the access token that the Resource Authorization
Server issues are separate from grant binding, and the catalog does not
describe them.

## Identifier Uniqueness {#identifiers}

Identifiers are unique as follows:

* Tenant Object identifiers and Grant Catalog identifiers are each
  unique within a Relationship View.

* Entry identifiers are unique within a Grant Catalog.

The following protocol values are unique within their collections and
identify items within an entry: each `scope` value within a `values`
array, each `type` value within a `types` array, and each `claim` value
within a `claims` array. The `formats` and `methods`
arrays contain unique values.

## Entry Revision {#revision}

The `revision` member is an opaque version identifier controlled by the
Resource Authorization Server. The normative semantics of an entry
comprise:

* the `profile` and `resource` members;

* the `scope` member, including its `requirement` and the `scope` and
  `requirement` of each scope value object;

* the `authorization_details` member, including its `requirement` and
  the `type` and `requirement` of each authorization details type
  object;

* the `claims` member, including the `claim` and `requirement` of each
  claim object;

* the `sub_id` and `grant_binding` members; and

* any member registered with "Entry Semantics" set to "Yes" in the
  "OAuth Grant Catalog Members" registry ({{iana-members}}).

The Resource Authorization Server MUST change the `revision` of an entry
whenever it changes the normative semantics of the entry, and MUST NOT
use, in Relationship Views for the same Issuer Relationship, a
`revision` value it has previously used for that entry with different
normative semantics. When normative semantics are compared, the order of
elements in the `values`, `types`, `claims`, `formats`, and `methods`
arrays is not significant, an absent item requirement is equivalent to
the value `optional`, and a Grant Issuer compares only the members it
understands. A change limited to other members, such as `display`,
`description`, `documentation_uri`, or `definition_uri`, does not
require a new revision.

The combination of the Resource Authorization Server issuer identifier,
the Issuer Relationship, the Grant Catalog `id`, the entry `id`, and the
`revision` identifies one set of normative semantics. A Grant Issuer
that has currently installed that combination with different normative
semantics treats the Relationship View as invalid ({{validation}}).
Grant Catalog identifiers are unique only within a Relationship View
({{identifiers}}), so the Grant Catalogs offered to two Issuer
Relationships can share an identifier and differ in content.

The `revision` member is neither a digest nor an integrity mechanism. It
lets the Grant Issuer determine whether an approval made for an earlier
copy of an entry still applies ({{reconciliation}}). The Retrieval
Binding handles caching of representations.

# Authorization Envelope {#envelope}

An entry defines an Authorization Envelope: the set of grants that the
Resource Authorization Server is prepared to accept for one protected
resource from the Issuer Relationship. The entries of a Relationship
View describe the maximum set of grants that the Grant Issuer can
configure for the relationship; Grant Issuer policy and the runtime
request narrow it further.

## Permitted Grants {#permitted}

A grant is within the Authorization Envelope of an entry, for a Tenant
Object that references the Grant Catalog containing the entry, when all
of the following are true:

1. The grant conveys the `issuer` of the Grant Catalog as its only
   audience.

2. If the Tenant Object contains `target_tenant`, the grant conveys that
   value as its target tenant. Otherwise, the grant conveys no target
   tenant.

3. The grant conveys the entry's `resource` as its only protected
   resource.

4. If the entry has no `scope` member, the grant conveys no scope
   values. Otherwise, every scope value the grant conveys is the `scope`
   of a scope value object of the entry, the grant conveys every scope
   value whose item requirement is `required`, and the grant
   conveys at least one scope value if the presence requirement is
   `required`.

5. If the entry has no `authorization_details` member, the grant conveys
   no authorization details. Otherwise, the `type` of every
   authorization details object the grant conveys is the `type` of an
   authorization details type object of the entry, the grant conveys at
   least one object of each type whose item requirement is
   `required`, and the grant conveys at least one object if the
   presence requirement is `required`.

6. Every Catalog-Governed Claim the grant conveys is listed in the
   entry's `claims` member, and the grant conveys every claim listed
   with an item requirement of `required`.

7. If the entry has no `sub_id` member, the grant conveys no subject
   identifier of the kind that member describes. Otherwise, a subject
   identifier the grant conveys uses one of the listed formats, and the
   grant conveys one if the presence requirement is `required`.

8. If the entry has no `grant_binding` member, the grant is not bound.
   Otherwise, a binding the grant conveys uses exactly one of the listed
   methods, and the grant is bound if the presence requirement is
   `required`.

When a Grant Issuer issues a grant using an entry, the Grant Issuer MUST
ensure that the grant is within the Authorization Envelope of the subset
of that entry approved for the Tenant Object ({{narrowing}}). The Grant
Issuer issues each grant using exactly one entry, and MUST NOT combine
inputs permitted by different entries in one grant.

The catalog does not oblige the Resource Authorization Server to accept
a grant that is within an Authorization Envelope
({{sec-not-authorization}}).

## Combinations Across Members {#combinations}

The Authorization Envelope of an entry is the product of the selections
each member permits. The permitted selections for a member are all of
its `required` items together with any subset of its `optional` items,
subject to the presence requirement. Any selection permitted by one
member is permitted together with any selection permitted by each other
member.

For example, an entry whose `scope` member requires `messages.read` and
permits `messages.write` permits a grant with the scope value
`messages.read` alone or with both values, but not with
`messages.write` alone. Each of these scope selections is permitted
together with any permitted selection of authorization details types,
claims, subject identifier formats, and binding methods.

The catalog expresses no alternatives between sets of items, no
exclusions, and no conditions, either within a member or between
members. When the Resource Authorization Server accepts only some
combinations, such as one scope value or another but not both, a scope
value only together with a particular claim, or an authorization details
type that cannot be combined with a scope value, it publishes separate
entries whose Authorization Envelopes each contain only accepted
combinations. A Grant Issuer does not infer a rule that an entry does
not state. {{example-alternatives}} contains an example.

## Approval and Runtime Narrowing {#narrowing}

An administrator of the Grant Issuer approves a subset of an entry for
use with a Tenant Object that references the entry's Grant Catalog. An
approved subset is an Authorization Envelope contained in the entry's
Authorization Envelope. It retains every requirement of the entry and
can add requirements of its own: an administrator can remove `optional`
items, subject identifier formats, and binding methods, and can make an
`optional` member or item `required`. For example, for an entry whose
`grant_binding` member has the `requirement` `optional` and the methods
`dpop` and `mtls`, an administrator can approve a subset in which
binding is required and `dpop` is the only method. Using that entry, the
Grant Issuer then issues only grants bound to a DPoP key. How a Grant
Issuer records approval, and whether it approves entries for particular
clients or subjects, is a matter of Grant Issuer policy.

The rules in {{entry-claims}} and {{entry-grant-binding}} that refer to
what an entry requires or permits apply to the approved subset.

At runtime, the Grant Issuer applies subject, client, consent, and other
local policy, and the client's request, to narrow the approved subset
further. If local policy excludes an item that the approved subset
requires, the Grant Issuer MUST NOT issue a grant using that entry. If
the client's request omits such an item, the Grant Issuer either adds
it, when its policy permits issuing values that the client did not
request ({{Section 3.3 of RFC6749}}), or does not issue a grant using
that entry.

The following example shows a protected resource that publishes three
scope values, a catalog entry that permits two of them for an Issuer
Relationship, an administrator who approves one, and a client that
requests two:

~~~ ascii-art
Publicly advertised scopes:  messages.read, messages.write,
                             messages.admin
Catalog entry:               messages.read, messages.write
Approved by policy:          messages.read
Requested by the client:     messages.read, messages.write
Issued in the grant:         messages.read
~~~
{: title="Narrowing of Scope Values"}

The Grant Issuer cannot issue `messages.admin` merely because public
metadata advertises it, and cannot issue `messages.write` merely because
the catalog permits it. The same narrowing applies to authorization
details types, claims, subject identifier formats, and grant binding.

## Issuing a Grant {#issuing}

After authenticating the client and the subject and applying its policy,
the Grant Issuer issues a grant as specified by the Grant Profile, using
one approved Tenant Object and one approved entry. The Grant Issuer
selects the Tenant Object from the client's request and its own
configuration for the Issuer Relationship, and selects an entry whose
Grant Profile matches the kind of grant the client requested and whose
`resource` is the requested protected resource. The catalog does not
define a request parameter for selecting a Resource Authorization Server
tenant; the Grant Profile Definition specifies how the selection is made
({{profile-definitions}}). When more than one approved Tenant Object or
entry could satisfy a request, Grant Issuer policy selects one. Because
a grant issued using an entry conveys only that entry's resource, the
Grant Issuer issues a grant for a request that names several resources
only if its policy reduces the request to one of them, and for a request
that names none only if its policy determines one.

The Grant Issuer MUST NOT issue a grant using an entry obtained for an
Issuer Relationship unless the grant identifies its issuer by the issuer
identifier of that relationship and, when the relationship has an issuer
tenant, identifies that issuer tenant, as the Grant Profile Definition
specifies ({{profile-definitions}}).

When the Grant Issuer does not issue a grant because no approved entry
permits the request, it responds to the client as the Grant Profile
Definition specifies.

## Clients {#clients}

The catalog does not enumerate or register application clients, or
establish client identifier mappings. A Relationship View is
relationship-wide and does not establish eligibility for a particular
application client: a Resource Authorization Server can permit a grant
configuration for one application client and reject an otherwise
equivalent configuration for another. Views specific to one application
client are an open issue ({{open-issues}}). A Grant Profile whose grants
identify the client by its client identifier at the Resource
Authorization Server, as ID-JAG does, specifies how the Grant Issuer
obtains that identifier; {{id-jag}} describes this for ID-JAG.

# Catalog Processing {#processing}

This section specifies how a Grant Issuer processes a Relationship View,
whatever Retrieval Binding delivered it.

## Validating a Relationship View {#validation}

Before installing any part of a Relationship View, the Grant Issuer MUST
verify the following:

1. The `issuer` of every Grant Catalog, and any issuer identifier the
   Retrieval Binding conveys, is identical to the issuer identifier of
   the Resource Authorization Server with which the Grant Issuer
   associates the Issuer Relationship, using simple string comparison
   ({{Section 6.2.1 of RFC3986}}).

2. The Issuer Relationship that the Retrieval Binding identifies with
   the Relationship View is the one for which the Grant Issuer retrieved
   it: the issuer identifiers are identical, and the issuer tenant
   identifiers are identical or are both absent.

3. Every document conforms to I-JSON {{RFC7493}}. The structures that
   the Retrieval Binding defines, every Tenant Object, and every Grant
   Catalog other than the contents of its entries satisfy their
   definitions: every REQUIRED member is present, and every member
   defined by this specification has the specified JSON type and meets
   the constraints stated for it. Tenant Object, Grant Catalog, and
   entry identifiers satisfy {{model-conventions}} and are unique as
   specified in {{identifiers}}.

4. The `grant_catalog` of every Tenant Object identifies a Grant Catalog
   in the Relationship View, and every Grant Catalog in the Relationship
   View is referenced by at least one Tenant Object.

5. At most one Tenant Object omits `target_tenant`, and no two Tenant
   Objects have the same `target_tenant`.

6. No entry has a Resource Authorization Server issuer identifier,
   Grant Catalog `id`, entry `id`, and `revision` that the Grant Issuer
   has currently installed, for the same Issuer Relationship, with
   different normative semantics ({{revision}}).

If any of these checks fails, the retrieval is incomplete, and the
Grant Issuer MUST NOT install any part of the Relationship View obtained
in it ({{installation}}).

In addition, the Grant Issuer MUST NOT use an entry for issuance if any
of the following is true. Each of these conditions affects only that
entry:

1. The Grant Issuer does not implement a Grant Profile Definition for
   the Grant Profile identified by `profile`.

2. A member of the entry other than its `id`, or a member of an object
   within the entry, does not satisfy its definition in {{entry}} or
   {{identifiers}}: a REQUIRED member is absent, a member has a JSON
   type other than the one specified, or a constraint stated for the
   member is not met.

3. The entry contains a member for a Catalog-Governed Input that its
   Grant Profile Definition states the Grant Profile cannot convey
   ({{profile-definitions}}).

4. The entry conflicts with public metadata ({{composition}}).

An entry that requires an item the Grant Issuer cannot produce, such as
a subject identifier format or a grant binding method that the Grant
Issuer does not implement, is valid but cannot be used by that Grant
Issuer. A Grant Issuer SHOULD make available to its administrators the
reason that a Relationship View was not installed or that an entry
cannot be used for issuance.

## Composition with Public Metadata {#composition}

A Grant Issuer composes each entry with the public metadata of the
Resource Authorization Server and of the entry's protected resource.
Composition is not a JSON merge: the Grant Issuer processes each source
according to the specification that defines it, and a grant that the
Grant Issuer issues is within the public capabilities, the entry, Grant
Issuer policy, and the runtime request.

Public metadata remains authoritative for what it describes. The catalog
cannot replace an issuer identifier, token endpoint, resource
identifier, client authentication method, or cryptographic capability
published in public metadata, and an entry cannot enable a value that
public metadata excludes. Conversely, a capability advertised in public
metadata is not thereby available to an Issuer Relationship; the catalog
states what is available to the relationship.

To compose an entry with public metadata, the Grant Issuer performs the
following steps:

1. Retrieve and validate the Resource Authorization Server's
   Authorization Server Metadata ({{Section 3.3 of RFC8414}}), and
   verify that its issuer identifier is identical to the Grant Catalog
   `issuer`.

2. Request the Protected Resource Metadata for the entry's `resource`
   ({{Section 3 of RFC9728}}). If the response has HTTP status code
   200 (OK), validate the metadata as specified in
   {{Section 3.3 of RFC9728}}. Any other response leaves this step
   without effect, because publishing Protected Resource Metadata is
   optional.

3. For each member in the following table that the Authorization Server
   Metadata or the validated Protected Resource Metadata contains,
   verify that the member's values include every corresponding value of
   the entry.

4. Perform the checks that the Grant Profile Definition specifies, such
   as checks of `grant_types_supported` ({{jwt-composition}}).

| Member | Metadata | Entry Values Checked |
| --- | --- | --- |
| `authorization_grant_profiles_supported` ({{Section 7.2 of ID-JAG}}) | Authorization Server | `profile` |
| `authorization_details_types_supported` ({{Section 10 of RFC9396}}) | Authorization Server | Each authorization details `type` |
| `authorization_details_types_supported` ({{Section 2 of RFC9728}}) | Protected Resource | Each authorization details `type` |
{: title="Public Metadata Members Checked in Composition"}

A Resource Authorization Server that publishes
`authorization_grant_profiles_supported` MUST include in it the Grant
Profile of every entry it publishes. A specification that defines
another metadata member enumerating the values supported for a Grant
Profile or a Catalog-Governed Input can specify that the member is
checked in the same way. A member from which the
defining specification permits a server to omit supported values is not
checked: the scope values of an entry need not appear in
`scopes_supported`, which {{Section 2 of RFC8414}} and
{{Section 2 of RFC9728}} permit a server to leave incomplete. Similarly,
an `authorization_servers` member of Protected Resource Metadata that
omits the Resource Authorization Server's issuer identifier is not a
conflict, because {{Section 2 of RFC9728}} permits a protected resource
to omit some of its authorization servers.

If a step fails, including when Protected Resource Metadata is retrieved
and fails validation, the entry conflicts with public metadata and the
Grant Issuer MUST NOT use it for issuance. A conflict affects only that
entry. The Grant Issuer does not resolve a conflict by preferring
catalog values over public metadata or by combining values from both.

### Use of Authorization Details Types Metadata {#rar-metadata}

If the Resource Authorization Server publishes the
`authorization_details_types_metadata_endpoint` metadata member
{{RAR-METADATA}}, the Grant Issuer MAY retrieve the metadata for a
`type` that an entry permits and use its `schema` or `schema_uri` to
assist administrators or to validate authorization details objects
locally. That metadata describes valid object structure. It is not an
authorization policy for the values in an object, it cannot add a type
to an entry or broaden an entry, and its absence, or a failure to
retrieve it, does not affect the validity of an entry.

The Grant Issuer MUST NOT send a credential obtained through a
Retrieval Binding when dereferencing a `schema_uri`,
`documentation_uri`, or `definition_uri`. A Grant Issuer that uses a
schema to enforce policy retains the schema, or a cryptographic digest
of it, and requires new approval when the schema changes; otherwise its
use of the schema is advisory. Schemas that a Resource Authorization
Server publishes only to particular Issuer Relationships, and variations
of a schema by tenant, are outside the scope of this specification.

### Use of Authority Metadata {#authority-metadata}

If the Protected Resource Metadata for an entry's `resource` contains
the `authority_metadata` member ({{Section 2.1 of AUTHORITY-METADATA}}),
the Grant Issuer MAY use its authority entries to show administrators
what the scope values and authorization details types of the entry
authorize. An authority entry describes an item of the entry when its
`kind` is `scope` and its `value` is the item's `scope`, or its `kind`
is `authorization_details_type` and its `value` is the item's `type`.
The protected resource is the one the entry names, so items need no
other mapping. Authorization Server Metadata can carry the same member
for the Resource Authorization Server's own vocabulary
({{Section 2.2 of AUTHORITY-METADATA}}).

Authority metadata is advisory input to approval. It cannot add an item
to an entry or broaden an entry, it does not cause an entry to conflict
with public metadata, and the absence of an authority entry does not
affect the validity of an entry. A Grant Issuer that retains, with an
approval, the authority description that the administrator saw can
detect a later change to it and request review; such a change does not
affect the entry's revision ({{sec-scope-meaning}}). When the Grant
Issuer narrows a request at runtime ({{narrowing}}), it can use an
attenuation relationship that the authority metadata declares, but only
to select a value within the approved subset.

## Installation {#installation}

The Grant Issuer installs only a Relationship View that has passed
validation ({{validation}}), and distinguishes the following outcomes of
a retrieval:

Complete retrieval:
: The Grant Issuer obtains and validates a complete Relationship View,
  replaces the installed state for the Issuer Relationship with it, and
  reconciles its policy ({{reconciliation}}). A Tenant Object, Grant
  Catalog, or entry that the new Relationship View omits, or contains
  with a different `target_tenant`, `grant_catalog`, or `revision`, is
  withdrawn, and the Grant Issuer MUST stop issuing grants that depend
  on it in its previous form.

Incomplete retrieval:
: A transport failure occurs, a response is interrupted or fails
  validation, or the Resource Authorization Server rejects the Grant
  Issuer's credentials. The Grant Issuer MUST NOT install any state from
  that retrieval, and does not treat it as a withdrawal. State installed
  from an earlier retrieval remains in force, subject to the maximum age
  described below.

A catalog does not need to be current for the Grant Issuer to use it,
because the Resource Authorization Server applies its current policy
whenever it processes a grant. Rejection at redemption does not undo
disclosure, however: a grant that the Resource Authorization Server
rejects has still conveyed its claims to the client. The Grant Issuer
SHOULD retrieve the Relationship View periodically and after
administrative changes to the Issuer Relationship, and SHOULD stop
issuing grants that depend on installed state it has not been able to
refresh within a maximum age that it configures for the Issuer
Relationship, such as one day. The maximum age is measured from the most
recent complete retrieval; an incomplete retrieval does not extend it.
When a Resource Authorization Server ends an Issuer Relationship, the
Grant Issuer observes this as incomplete retrievals, and the maximum age
bounds how long the Grant Issuer continues to issue grants under the
state it installed before then. A Resource Authorization Server that
needs issuance to stop sooner first removes every eligible tenant from
the relationship; the Grant Issuer observes that removal at its next
refresh as a withdrawal.

## Policy Reconciliation {#reconciliation}

A Grant Issuer retains, with each approval, the Resource Authorization
Server issuer identifier, the Issuer Relationship, the Tenant Object
`id` and `target_tenant`, the Grant Catalog `id`, the entry `id` and
`revision`, the approved subset, and any client identifier mapping that
the Grant Profile requires. The items of the approved subset are
identified by their protocol values. State that is
specific to a Retrieval Binding, such as a URI or an HTTP entity tag, is
retained separately and is not part of an approval.

An approval applies while the installed Relationship View contains a
Tenant Object with the approved `id`, `target_tenant`, and
`grant_catalog`, and the Grant Catalog that the Tenant Object references
contains an entry with the approved `id` and `revision`. When the
Relationship View changes, the following rules apply:

* The appearance of a new Tenant Object or entry MUST NOT, by itself,
  cause the Grant Issuer to issue grants using it, and a change to an
  entry MUST NOT, by itself, expand an existing approval.

* When a Tenant Object, Grant Catalog, or entry is withdrawn
  ({{installation}}), the Grant Issuer suspends issuance under the
  approvals that depend on it. If a later complete retrieval contains
  the object again with the approved identifiers, `target_tenant`,
  `grant_catalog`, and `revision`, the approval applies again without
  new approval. This occurs, for example, when the Resource
  Authorization Server removes a tenant from the relationship and later
  restores it unchanged.

* When the `target_tenant` or `grant_catalog` of a Tenant Object
  changes, or the `revision` of an entry changes, the Grant Issuer MUST
  NOT issue grants under an approval made for the previous value.
  Issuance resumes only after an administrator approves the changed
  object.

These rules apply to every change of revision, including a change that
only narrows an entry, such as the removal of a scope value
({{sec-revision}}). A change limited to members outside the normative
semantics, such as `display`, does not require a new revision
({{revision}}), so it need not affect approvals. Whether to relax these
rules is an open issue ({{open-issues}}).

# Relationship Authentication {#relationship-authentication}

Before disclosing catalog content, the Resource Authorization Server
establishes, through a Retrieval Binding, which Issuer Relationship the
disclosure serves, and verifies that the relationship is authorized and
current.

## Relationship Identity {#relationship-identity}

An Issuer Relationship is identified by the Grant Issuer's issuer
identifier {{RFC8414}} and, when that issuer is multi-tenant, an issuer
tenant identifier. The issuer identifier is the value by which grants
issued by the Grant Issuer identify their issuer. The issuer tenant
identifier is a tenant identifier ({{conventions}}) of the Grant Issuer,
and is the value by which grants identify their issuer tenant. Where a
grant or a catalog access assertion conveys the issuer tenant identifier
in the `tenant` claim {{OpenID.Enterprise}}, the value is the stable,
issuer-unique form of that claim. The values `personal` and
`organization`, which Section 2.2 of {{OpenID.Enterprise}} reserves for
categories of accounts, do not identify an issuer tenant.

An issuer tenant identifier is scoped by the issuer identifier. The
Resource Authorization Server MUST treat the pair as one exact
identifier, and MUST NOT interpret, normalize, or derive an issuer
tenant identifier, or compare issuer tenant identifiers across issuers.

## Relationship Authorization {#relationship-authorization}

An Onboarding Arrangement determines which Issuer Relationships a
Resource Authorization Server discloses catalog content to, and which of
its tenants are eligible for each. An Onboarding Arrangement also
records whether the Grant Issuer uses a multi-tenant issuer, whether
client authentication or DPoP-bound catalog access assertions are
required, and the keys and algorithms the Grant Issuer uses. How an
Onboarding Arrangement is established is outside the scope of this
specification.

Authorization to retrieve a catalog is distinct from trust in the keys a
Grant Issuer uses to sign grants. A Resource Authorization Server that
accepts grants from an issuer has not thereby agreed to disclose catalog
content for every issuer tenant that issuer names. The Resource
Authorization Server MUST NOT disclose catalog content to an Issuer
Relationship that no Onboarding Arrangement authorizes, and MUST
determine the Relationship View solely from the authenticated Issuer
Relationship.

The Resource Authorization Server MUST verify that the Issuer
Relationship is authorized each time it discloses catalog content.
Ending an Issuer Relationship therefore stops further disclosure,
including disclosure that a credential issued before the relationship
ended would otherwise permit.

## Requirements for Retrieval Bindings {#binding-requirements}

A Retrieval Binding specifies how it meets the following requirements:

1. It authenticates the Grant Issuer's issuer identifier and conveys the
   issuer tenant identifier under that authentication, so that the
   Resource Authorization Server learns the Issuer Relationship from a
   statement the Grant Issuer has authenticated.

2. It authenticates the Resource Authorization Server to the Grant
   Issuer, and identifies, with the Relationship View it delivers, the
   issuer identifier of the Resource Authorization Server and the Issuer
   Relationship to which the view belongs.

3. It carries the objects defined in {{data-model}} without changing
   their members or semantics.

4. It defines how the Grant Issuer discovers the binding and obtains a
   complete Relationship View that reflects one consistent state of the
   Issuer Relationship.

5. It defines its error responses and how the Resource Authorization
   Server verifies the status of the relationship
   ({{relationship-authorization}}).

Different bindings convey the same Issuer Relationship, not the same
credential. When more than one Retrieval Binding is available for an
Issuer Relationship, the Grant Issuer MUST obtain the Relationship View
through one binding, and MUST NOT combine Tenant Objects, Grant
Catalogs, or entries retrieved through different bindings. A Grant
Issuer can compare views retrieved through different bindings for
diagnostic purposes; a difference does not expand what the Grant Issuer
installs.

# OAuth Retrieval Binding {#oauth-binding}

The OAuth Retrieval Binding requires only public metadata and an
Onboarding Arrangement. The Grant Issuer discovers the binding from the
Resource Authorization Server's metadata, obtains a short-lived catalog
access token at the token endpoint by presenting a JWT authorization
grant that it signs, and retrieves its complete Relationship View from
the catalog endpoint in one response. The binding is the same for every
Grant Profile.

## Authorization Server Metadata {#as-metadata}

This specification defines the following Authorization Server Metadata
{{RFC8414}} member for the OAuth Retrieval Binding:

`authorization_grant_catalog_endpoint`:
: URL of the catalog endpoint, from which the Grant Issuer retrieves its
  Relationship View ({{view-document}}). The URL uses the `https` scheme
  and contains no fragment component. It is also the resource indicator
  ({{Section 2 of RFC8707}}) that the Grant Issuer uses when requesting
  a catalog access token; this specification calls the protected
  resource that the URL identifies the catalog resource.

A Resource Authorization Server that supports the OAuth Retrieval
Binding MUST publish this member. A Grant Issuer MUST NOT use the
binding with a Resource Authorization Server whose metadata omits this
member.

The following is an example of Authorization Server Metadata for a
Resource Authorization Server that supports ID-JAG and the OAuth
Retrieval Binding:

~~~ json
{
  "issuer": "https://ras.example/",
  "token_endpoint": "https://ras.example/oauth2/token",
  "jwks_uri": "https://ras.example/oauth2/jwks",
  "grant_types_supported": [
    "urn:ietf:params:oauth:grant-type:jwt-bearer",
    "urn:ietf:params:oauth:grant-type:jwt-dpop"
  ],
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "token_endpoint_auth_methods_supported": ["private_key_jwt"],
  "dpop_signing_alg_values_supported": ["ES256"],
  "authorization_details_types_metadata_endpoint":
    "https://ras.example/oauth2/authorization-details-types",
  "authorization_grant_catalog_endpoint":
    "https://ras.example/oauth2/grant-catalog"
}
~~~

## Catalog Access Assertion {#assertion}

The Grant Issuer obtains a catalog access token by presenting a catalog
access assertion: a JWT authorization grant {{RFC7523}} that the Grant
Issuer signs on its own behalf, not on behalf of an End-User or a
client. The catalog access assertion is a JWS {{RFC7515}} with the
following JOSE header parameters:

`typ`:
: REQUIRED. The value `oauth-grant-catalog-access+jwt`, which explicitly
  types the assertion as {{Section 3.11 of RFC8725}} recommends.

`alg`:
: REQUIRED. An asymmetric digital signature algorithm. The value MUST
  NOT be `none` or a MAC-based algorithm.

`kid`:
: OPTIONAL. The key identifier of the Grant Issuer's signing key.

The catalog access assertion contains the following claims:

`iss`:
: REQUIRED. The issuer identifier of the Grant Issuer
  ({{relationship-identity}}).

`sub`:
: REQUIRED. The same value as `iss`. {{Section 3 of RFC7523}} requires a
  `sub` claim; for this assertion, the subject is the Grant Issuer
  itself, and the claim has no other meaning.

`aud`:
: REQUIRED. The issuer identifier of the Resource Authorization Server,
  as a JSON string. This narrows {{Section 3 of RFC7523}}, which also
  permits the token endpoint URL.

`tenant`:
: REQUIRED if the Grant Issuer uses a multi-tenant issuer; otherwise, it
  MUST be omitted. The issuer tenant identifier of the Issuer
  Relationship ({{relationship-identity}}), conveyed in the `tenant`
  claim {{OpenID.Enterprise}}.

`iat`:
: REQUIRED. The time at which the assertion was issued
  ({{Section 4.1.6 of RFC7519}}).

`exp`:
: REQUIRED. The expiration time of the assertion
  ({{Section 4.1.4 of RFC7519}}). The value MUST NOT be more than 300
  seconds after the value of `iat`.

`jti`:
: REQUIRED. A unique identifier for the assertion
  ({{Section 4.1.7 of RFC7519}}).

`cnf`:
: OPTIONAL. A confirmation claim ({{Section 3.1 of RFC7800}}) containing
  only the `jkt` confirmation member, whose value is the JWK SHA-256
  Thumbprint of a DPoP key that the Grant Issuer controls
  ({{Section 6.1 of RFC9449}}), as specified in
  {{Section 3 of JWT-DPOP}}. A catalog access assertion that contains
  this claim is DPoP-bound. The Grant Issuer MUST NOT include this claim
  unless the `grant_types_supported` member of the Resource
  Authorization Server's metadata includes
  `urn:ietf:params:oauth:grant-type:jwt-dpop`.

The catalog access assertion identifies the Issuer Relationship and,
when it is DPoP-bound, the key with which the Grant Issuer presents it,
and nothing else. It does not identify a Grant Profile, target tenant,
End-User, client, resource, scope, or authorization details, and does
not contain the `aud_tenant` claim; the Resource Authorization Server
determines the eligible tenants from the Issuer Relationship. Other
claims MAY be present and are ignored unless understood.

The following is an example of the JOSE header and claims of a catalog
access assertion:

~~~ json
{
  "typ": "oauth-grant-catalog-access+jwt",
  "alg": "ES256",
  "kid": "issuer-key-1"
}
~~~

~~~ json
{
  "iss": "https://idp.example/",
  "sub": "https://idp.example/",
  "aud": "https://ras.example/",
  "tenant": "acme-workforce",
  "iat": 1788897600,
  "exp": 1788897720,
  "jti": "catalog-request-123"
}
~~~

### Validating the Catalog Access Assertion {#assertion-validation}

The Resource Authorization Server validates the catalog access assertion
as specified in {{Section 3 of RFC7523}}, and in addition MUST ensure
the following:

1. The `typ` JOSE header parameter is `oauth-grant-catalog-access+jwt`.

2. The `iss` claim identifies a Grant Issuer for which an Onboarding
   Arrangement exists. The signature verifies using a key of that issuer
   obtained from the `jwks_uri` in the issuer's Authorization Server
   Metadata ({{Section 3 of RFC8414}}) or provisioned under the
   Onboarding Arrangement. The signature algorithm is an asymmetric
   algorithm that the Resource Authorization Server allows for that
   issuer ({{Section 3.1 of RFC8725}}).

3. The `sub` claim is identical to the `iss` claim.

4. The `aud` claim is a JSON string identical to the issuer identifier
   of the Resource Authorization Server.

5. The `iat` and `exp` claims are present, `iat` is not later than the
   current time, the assertion has not expired, and `exp` is not more
   than 300 seconds after `iat`. Each comparison with the current time
   is subject to the Resource Authorization Server's allowance for clock
   skew.

6. The `jti` claim is present and has not been used in a previously
   accepted catalog access assertion from the same issuer. An assertion
   is accepted when the Resource Authorization Server issues a catalog
   access token in response to it. The Resource Authorization Server
   retains each accepted `jti` value, across every instance of its token
   endpoint, until the assertion's `exp` time plus the allowance for
   clock skew has passed.

7. The pair of `iss` and, when present, `tenant` identifies an Issuer
   Relationship that an Onboarding Arrangement currently authorizes
   ({{relationship-authorization}}). If the issuer is multi-tenant, an
   assertion that has no `tenant` claim, or a `tenant` claim with the
   value `personal` or `organization`, identifies no Issuer
   Relationship. If the issuer is single-tenant, an assertion that has a
   `tenant` claim identifies no Issuer Relationship.

The Resource Authorization Server MUST NOT accept a JWT with a `typ`
other than `oauth-grant-catalog-access+jwt` as a catalog access
assertion, and MUST NOT accept a catalog access assertion as a grant of
any Grant Profile or as a client assertion. These rules ensure that a
catalog access assertion is accepted only as a catalog access assertion
({{Section 3.12 of RFC8725}}), whether or not a Grant Profile specifies
a `typ` value for its grants.

## Catalog Access Token Request {#token-request}

The Grant Issuer requests a catalog access token by sending a token
request ({{Section 2.1 of RFC7523}}) to the token endpoint of the
Resource Authorization Server with the following parameters:

`grant_type`:
: REQUIRED. The value `urn:ietf:params:oauth:grant-type:jwt-bearer` or,
  for a DPoP-bound catalog access assertion, the value
  `urn:ietf:params:oauth:grant-type:jwt-dpop`
  ({{Section 4.1 of JWT-DPOP}}).

`assertion`:
: REQUIRED. The catalog access assertion.

`resource`:
: REQUIRED. The value of the `authorization_grant_catalog_endpoint`
  metadata member ({{Section 2 of RFC8707}}).

`scope`:
: REQUIRED. A space-delimited list of scope values that includes
  `authorization_grant_catalog.read`, the scope value this specification
  defines for read access to the catalog.

A request that presents a DPoP-bound catalog access assertion includes
a DPoP proof ({{Section 4 of RFC9449}}) for the key that the `cnf` claim
identifies. A Resource Authorization Server that supports the OAuth
Retrieval Binding and includes
`urn:ietf:params:oauth:grant-type:jwt-dpop` in `grant_types_supported`
MUST accept DPoP-bound catalog access assertions.

Whether the token endpoint also requires client authentication is a
policy of the Resource Authorization Server, as
{{Section 3.1 of RFC7523}} permits, and is fixed for the Grant Issuer by
the Onboarding Arrangement. When client authentication is required, the
Grant Issuer is a registered client of the Resource Authorization Server
and authenticates using the method fixed by the Onboarding Arrangement,
such as a JWT client assertion signed with a key of the Grant Issuer
({{Section 2.2 of RFC7523}}). The Grant Issuer MUST NOT use a catalog
access assertion as a client assertion, and SHOULD sign client
assertions with a key that it does not use to sign catalog access
assertions or grants ({{Section 3.12 of RFC8725}}). When client
authentication is not required, the request contains no `client_id`
parameter.

The following is an example of a catalog access token request with
client authentication, in which the catalog access assertion and the
client assertion are abbreviated (line breaks for display purposes
only):

~~~ http-message
POST /oauth2/token HTTP/1.1
Host: ras.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
&assertion=eyJ0eXAiOiJvYXV0aC1ncmFudC1jYXRhbG9nLWFjY2Vzcytqd3Qi...
&resource=https%3A%2F%2Fras.example%2Foauth2%2Fgrant-catalog
&scope=authorization_grant_catalog.read
&client_id=https%3A%2F%2Fidp.example%2F
&client_assertion_type=
 urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer
&client_assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6Imlzc3Vlci1rZXktMSJ9...
~~~

The Resource Authorization Server processes the request as follows:

1. If the grant type is `urn:ietf:params:oauth:grant-type:jwt-dpop`, it
   validates the DPoP proof ({{Section 4.3 of RFC9449}}) before
   validating the assertion, so that a request rejected for a missing or
   invalid proof or nonce does not consume the assertion's `jti`
   ({{Section 5.1 of JWT-DPOP}}).

2. It validates the catalog access assertion
   ({{assertion-validation}}).

3. If the grant type is `urn:ietf:params:oauth:grant-type:jwt-dpop`, it
   verifies that the assertion is DPoP-bound, that its `cnf` claim
   contains only the `jkt` member, and that the `jkt` value is the JWK
   SHA-256 Thumbprint of the key in the DPoP proof
   ({{Section 5 of JWT-DPOP}}). If the grant type is
   `urn:ietf:params:oauth:grant-type:jwt-bearer`, it verifies that the
   assertion contains no `cnf` claim and that the Onboarding Arrangement
   does not require DPoP-bound catalog access assertions.

4. It verifies that the `resource` parameter is identical to its
   `authorization_grant_catalog_endpoint` metadata value.

5. It verifies that the `scope` parameter includes
   `authorization_grant_catalog.read`.

6. If the Onboarding Arrangement requires client authentication, it
   authenticates the client and verifies that the Onboarding
   Arrangement associates the authenticated client with the `iss` claim
   of the catalog access assertion. It does not accept as a client
   assertion the JWT presented as the catalog access assertion, or any
   JWT whose `typ` is `oauth-grant-catalog-access+jwt`.

When the Onboarding Arrangement requires client authentication, the
Resource Authorization Server MUST NOT issue a catalog access token on
the strength of the catalog access assertion alone.

Step 3 rejects a DPoP-bound catalog access assertion presented with the
`urn:ietf:params:oauth:grant-type:jwt-bearer` grant type whether or not
the Resource Authorization Server supports the
`urn:ietf:params:oauth:grant-type:jwt-dpop` grant type for other grants.
This rejection prevents a party that obtains a DPoP-bound assertion from
presenting it without the key ({{Section 7.1 of JWT-DPOP}}).

Client authentication identifies the Grant Issuer as a client, but the
catalog is disclosed to the Issuer Relationship carried in the catalog
access assertion. A Grant Issuer that uses a multi-tenant issuer
therefore needs at most one client registration at each Resource
Authorization Server. The client identifier assigned to the Grant Issuer
can equal its issuer identifier, as in the example above.

## Catalog Access Token Response {#token-response}

If the request is valid, the Resource Authorization Server responds as
specified in {{Section 5.1 of RFC6749}}. The Resource Authorization
Server MUST restrict the catalog access token to the catalog resource
and the `authorization_grant_catalog.read` scope, and MUST associate the
token with the Issuer Relationship identified by the catalog access
assertion. The Resource Authorization Server SHOULD issue catalog access
tokens with a lifetime of a few minutes, and SHOULD NOT issue a refresh
token; the Grant Issuer obtains a new catalog access token with a new
catalog access assertion. The Resource Authorization Server
MUST NOT issue an access token for the catalog resource, or with the
`authorization_grant_catalog.read` scope, in response to any token
request other than one specified in {{token-request}}; whether to allow
others is an open issue ({{open-issues}}).

The following is an example of a catalog access token response:

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "2YotnFZFEjr1zCsicMWpAA",
  "token_type": "Bearer",
  "expires_in": 300,
  "scope": "authorization_grant_catalog.read"
}
~~~

When the catalog access assertion is DPoP-bound, the Resource
Authorization Server MUST issue a catalog access token bound to the same
key, with the `token_type` value `DPoP` ({{Section 6 of JWT-DPOP}}).
The catalog resource is a protected resource of the Resource
Authorization Server, so the exception in that section for protected
resources that do not support DPoP does not apply.

When the catalog access assertion is not DPoP-bound, the Resource
Authorization Server MAY issue a DPoP-bound catalog access token
({{RFC9449}}) or a certificate-bound catalog access token ({{RFC8705}})
when the Grant Issuer uses those mechanisms.

## Relationship View Document {#view-document}

The Grant Issuer retrieves its Relationship View by sending an HTTP GET
request to the `authorization_grant_catalog_endpoint` URL, presenting
the catalog access token as specified in {{Section 2.1 of RFC6750}} or,
for a DPoP-bound token, {{Section 7.1 of RFC9449}}. The Grant Issuer
MUST send a catalog access token only to the
`authorization_grant_catalog_endpoint` URL from Authorization Server
Metadata it has validated ({{Section 3.3 of RFC8414}}), without further
resolution or normalization of that URL. It MUST NOT follow redirections
in response to a request that carries a catalog access token, and treats
a redirection as an incomplete retrieval ({{installation}}).

The Resource Authorization Server determines the Relationship View from
the Issuer Relationship associated with the catalog access token,
verifies the status of that relationship
({{relationship-authorization}}), and responds with HTTP status code 200
(OK) and a Relationship View document of media type
`application/oauth-grant-catalog+json` ({{iana-media-types}}). The
Resource Authorization Server MUST construct the document from one
consistent state of the Issuer Relationship. The document is a JSON
object with the following members:

`issuer`:
: REQUIRED. The issuer identifier of the Resource Authorization Server.

`issuer_relationship`:
: REQUIRED. A JSON object that identifies the Issuer Relationship that
  the Relationship View serves. Its `issuer` member is REQUIRED and
  contains the issuer identifier of the Grant Issuer. Its `tenant`
  member contains the issuer tenant identifier, and is present if and
  only if the Issuer Relationship has one ({{relationship-identity}}).
  The Resource Authorization Server derives this object from the Issuer
  Relationship that it used to select the Relationship View.

`tenants`:
: REQUIRED. A JSON array containing every Tenant Object
  ({{tenant-object}}) eligible for the Issuer Relationship.

`grant_catalogs`:
: REQUIRED. A JSON array containing each Grant Catalog
  ({{grant-catalog}}) that a Tenant Object references, each exactly
  once.

The document contains the complete Relationship View and is not paged.
If the Issuer Relationship has no eligible tenants, both arrays are
empty.

The Grant Issuer validates the document as specified in {{validation}}:
`issuer` is the issuer identifier that the binding conveys (check 1),
and `issuer_relationship` identifies the Issuer Relationship with the
view (check 2). When a document's `issuer_relationship` differs from
the Issuer Relationship for which the Grant Issuer obtained the catalog
access token, the Grant Issuer therefore installs nothing from that
document and withdraws nothing. This check detects errors in routing,
caching, or the association of responses with requests. It relies on
the authentication that the binding provides, and does not show that
the Resource Authorization Server selected the content of the view
correctly.

The following is an example of a Relationship View document request and
response, in which the `entries` of the Grant Catalog are abbreviated;
{{example-catalog}} shows them in full:

~~~ http-message
GET /oauth2/grant-catalog HTTP/1.1
Host: ras.example
Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
If-None-Match: "view-91b6c4"
Accept-Encoding: gzip, br
~~~

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/oauth-grant-catalog+json
Cache-Control: no-store
ETag: "view-a24f91"

{
  "issuer": "https://ras.example/",
  "issuer_relationship": {
    "issuer": "https://idp.example/",
    "tenant": "acme-workforce"
  },
  "tenants": [
    {
      "id": "tenant-123",
      "display": "Acme",
      "target_tenant": "customer-acme",
      "grant_catalog": "standard"
    },
    {
      "id": "tenant-456",
      "display": "Acme Europe",
      "target_tenant": "customer-acme-eu",
      "grant_catalog": "standard"
    }
  ],
  "grant_catalogs": [
    {
      "issuer": "https://ras.example/",
      "id": "standard",
      "display": "Standard API access",
      "entries": [ ... ]
    }
  ]
}
~~~

## Caching and Refresh {#refresh}

The Resource Authorization Server SHOULD include a strong entity tag
({{Section 8.8.3 of RFC9110}}) in each Relationship View document
response, and SHOULD support conditional requests using `If-None-Match`
({{Section 13.1.2 of RFC9110}}), responding with HTTP status code 304
(Not Modified) ({{Section 15.4.5 of RFC9110}}) when the representation
is unchanged. The Grant Issuer sends `If-None-Match` only with the
entity tag of a representation it has installed. A 304 (Not Modified)
response confirms the installed Relationship View and is a complete
retrieval ({{installation}}).

A Relationship View document is specific to an Issuer Relationship. The
Resource Authorization Server MUST include `Cache-Control: no-store`
({{Section 5.2.2.5 of RFC9111}}) in these responses. The Grant Issuer
retains the document and its entity tag as application state, keyed by
the Resource Authorization Server's issuer identifier and the Issuer
Relationship.

This binding is designed for Relationship Views of up to several
thousand Tenant Objects in one response, not for unbounded collections.
Any change to a Relationship View changes the whole document. The Grant
Issuer SHOULD indicate support for content codings using
`Accept-Encoding` ({{Section 12.5.3 of RFC9110}}), and the Resource
Authorization Server SHOULD compress responses when the Grant Issuer
supports a content coding.

To refresh a Relationship View, the Grant Issuer performs the following
steps:

1. Obtain a catalog access token, unless it holds an unexpired one.

2. Retrieve the Relationship View document, using a conditional request
   when it holds an entity tag.

3. Validate the Relationship View ({{validation}}) and compose each
   entry with public metadata ({{composition}}).

4. Install the result ({{installation}}) and reconcile affected policy
   ({{reconciliation}}).

## Error Responses {#errors}

The token endpoint returns errors as specified in
{{Section 5.2 of RFC6749}}:

* If the catalog access assertion is invalid, expired, or replayed, or
  identifies no authorized Issuer Relationship; if a DPoP-bound
  assertion is presented with the
  `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type, or with a
  DPoP proof for a different key; if an assertion that is not
  DPoP-bound is presented when the Onboarding Arrangement requires one;
  or if the authenticated client is not associated with the assertion's
  issuer, the Resource
  Authorization Server responds with HTTP status code 400 (Bad Request)
  and the `invalid_grant` error code ({{Section 5.2 of RFC6749}}).

* If the `resource` parameter is missing or is not the catalog resource,
  the Resource Authorization Server responds with HTTP status code 400
  (Bad Request) and the `invalid_target` error code
  ({{Section 2 of RFC8707}}).

* If the `scope` parameter is missing or does not include
  `authorization_grant_catalog.read`, the Resource Authorization Server
  responds with HTTP status code 400 (Bad Request) and the
  `invalid_scope` error code ({{Section 5.2 of RFC6749}}).

* If the grant type is `urn:ietf:params:oauth:grant-type:jwt-dpop` and
  the DPoP proof is missing or invalid, or the Resource Authorization
  Server requires a nonce, it responds with the `invalid_dpop_proof`
  error code ({{Section 5 of RFC9449}}) or the `use_dpop_nonce` error
  code ({{Section 8 of RFC9449}}), as specified in
  {{Section 5 of JWT-DPOP}}.

* If client authentication required by the Onboarding Arrangement fails,
  the Resource Authorization Server responds with the `invalid_client`
  error code as specified in {{Section 5.2 of RFC6749}}.

The catalog endpoint returns errors as
specified in {{Section 3 of RFC6750}}, or {{Section 7.1 of RFC9449}} for
a DPoP-bound token:

* If the request contains no access token, the Resource Authorization
  Server responds with HTTP status code 401 (Unauthorized) and a
  `WWW-Authenticate` response header field that contains no error code
  ({{Section 3.1 of RFC6750}}).

* If the catalog access token is expired, revoked, or otherwise invalid,
  or the Issuer Relationship associated with it is no longer authorized,
  the Resource Authorization Server responds with HTTP status code 401
  (Unauthorized) and the `invalid_token` error code
  ({{Section 3.1 of RFC6750}}).

* If the access token does not include the
  `authorization_grant_catalog.read` scope, the Resource Authorization
  Server responds with HTTP status code 403 (Forbidden) and the
  `insufficient_scope` error code ({{Section 3.1 of RFC6750}}).

An error response MUST NOT disclose whether another issuer, issuer
tenant, Resource Authorization Server tenant, Grant Catalog, or entry
exists.

# Grant Profile Definitions {#profile-definitions}

A Grant Profile Definition applies the data model, the Authorization
Envelope, and the processing rules to one Grant Profile. A Grant Issuer
uses an entry only if it implements a Grant Profile Definition for the
entry's `profile` ({{validation}}).

A Grant Profile Definition specifies the following:

1. The Grant Profile identifier used in the `profile` member of an
   entry.

2. How a grant of the profile conveys each Catalog-Governed Input and
   each input in {{permitted}}: its audience, target tenant, protected
   resource, scope values, authorization details, claims, subject
   identifier, and grant binding. A Grant Profile to which the
   catalog applies conveys the audience and the protected resource.
   Where the profile cannot convey another input, the Grant Profile
   Definition says so; an entry for that profile then does not contain
   the corresponding member.

3. How a grant of the profile identifies its issuer and, for a
   multi-tenant issuer, its issuer tenant, so that the Grant Issuer can
   apply the rule in {{issuing}}.

4. The reserved claim names of the profile, including the claims the
   profile requires in every grant ({{entry-claims}}), and the namespace
   of the `claim` member.

5. For each grant binding method the profile supports, how the binding
   is represented in a grant, when the profile requires the Grant Issuer
   to bind a grant, and how the Grant Issuer selects among methods
   ({{entry-grant-binding}}).

6. The composition checks against public metadata that apply to the
   profile, such as checks of the grant types used to redeem its grants
   ({{composition}}).

7. How the Grant Issuer selects a Tenant Object for a request, and how
   it responds when it does not issue a grant because no approved entry
   permits the request ({{issuing}}).

8. Any rule of the Grant Profile that catalog use narrows, such as
   making an OPTIONAL element required.

A Grant Profile Definition for a profile of the JWT authorization grant
can adopt the conventions of {{jwt-grants}} for items 2, 3, 4, and 6,
and for the representation of bindings in item 5, and state only its
additions. {{id-jag}} is the only Grant Profile Definition that this
specification provides.

# JWT Authorization Grants {#jwt-grants}

This section defines conventions for representing the inputs of
{{permitted}} in JWT authorization grants {{RFC7523}}. They apply to a
Grant Profile only when its Grant Profile Definition adopts them, as
{{id-jag}} does. {{RFC7523}} does not define most of the claims below,
so a JWT authorization grant conveys an input only if its Grant Profile
defines the corresponding claim. A Grant Profile Definition that adopts
this section therefore states which inputs grants of the profile cannot
convey ({{profile-definitions}}).

## Representation of Inputs {#jwt-representation}

A JWT authorization grant conveys the inputs of {{permitted}} as
follows:

| Input | JWT Representation |
| --- | --- |
| Issuer | `iss` claim ({{Section 4.1.1 of RFC7519}}) |
| Issuer tenant | `tenant` claim {{OpenID.Enterprise}} |
| Audience | `aud` claim, as a string or an array of one string |
| Target tenant | `aud_tenant` claim ({{Section 3.1 of ID-JAG}}) |
| Protected resource | `resource` claim ({{Section 3.1 of ID-JAG}}), as a string or an array of one string |
| Scope values | `scope` claim ({{Section 4.2 of RFC8693}}) |
| Authorization details | `authorization_details` claim ({{Section 9.1 of RFC9396}}) |
| Other claims | Claims named by the `claim` member of claim objects |
| Subject identifier | `sub_id` claim ({{Section 4.1 of RFC9493}}) |
| Grant binding | `cnf` claim ({{Section 3.1 of RFC7800}}) |
{: title="Representation of Inputs in a JWT Authorization Grant"}

A `cnf` claim that represents a binding contains only the confirmation
member of one method:

`dpop`:
: The `jkt` confirmation member, containing the JWK SHA-256 Thumbprint
  of the client's DPoP key ({{Section 6.1 of RFC9449}}), as specified in
  {{Section 3 of JWT-DPOP}}.

`mtls`:
: The `x5t#S256` confirmation member, containing the SHA-256 thumbprint
  of the client's X.509 certificate ({{Section 3.1 of RFC8705}}).

## Reserved Claims {#jwt-reserved}

The reserved claim names of a Grant Profile whose Grant Profile
Definition adopts this section include `iss`, `sub`, `aud`, `jti`,
`exp`, `iat`, `nbf`, `tenant`, `aud_tenant`, `resource`, `scope`,
`authorization_details`, `sub_id`, `cnf`, `client_id`
({{Section 4.3 of RFC8693}}), and `act` ({{Section 4.1 of RFC8693}}).
The Grant Profile, not an entry, determines whether a grant conveys a
client identifier or an actor. A Grant Profile Definition can add
reserved claim names, including the claims that the profile requires in
every grant.

## Composition Checks {#jwt-composition}

A Resource Authorization Server that publishes entries whose Grant
Profile Definition adopts this section MUST include
`grant_types_supported` in its Authorization Server Metadata. For such
an entry, the Grant Issuer performs the following checks in step 4 of
{{composition}}, interpreting `grant_types_supported` with the default
value defined in {{Section 2 of RFC8414}}:

* Unless the entry requires binding and lists `dpop` as its only method,
  `grant_types_supported` includes
  `urn:ietf:params:oauth:grant-type:jwt-bearer`.

* If the entry lists `dpop`, `grant_types_supported` includes
  `urn:ietf:params:oauth:grant-type:jwt-dpop`, which
  {{Section 4.1 of JWT-DPOP}} defines for redeeming a DPoP-bound JWT
  authorization grant.

## Explicit Typing {#jwt-typing}

A Resource Authorization Server that accepts JWT authorization grants of
several Grant Profiles, and catalog access assertions, at the same token
endpoint distinguishes them by their `typ` JOSE header parameter
({{Section 3.11 of RFC8725}}). A Grant Profile Definition for a JWT
authorization grant SHOULD specify a `typ` value for grants of the
profile, as ID-JAG does with `oauth-id-jag+jwt`.

# Identity Assertion JWT Authorization Grant Profile {#id-jag}

This section is the Grant Profile Definition for the Identity Assertion
JWT Authorization Grant {{ID-JAG}}. In this profile, the Grant Issuer is
the IdP Authorization Server and grants are ID-JAGs. This Grant Profile
Definition adopts {{jwt-grants}}, with the following additions.

Grant Profile identifier:
: `urn:ietf:params:oauth:grant-profile:id-jag`
  ({{Section 7.2 of ID-JAG}}).

Inputs:
: ID-JAGs convey every input as {{jwt-representation}} specifies. The
  IdP Authorization Server includes the `tenant` claim when its issuer
  is multi-tenant ({{Section 6.1 of ID-JAG}}).

Reserved claims:
: The reserved claim names are those in {{jwt-reserved}}. The claims
  that {{Section 3.1 of ID-JAG}} defines and that are not reserved
  (`auth_time`, `acr`, `amr`, `aud_sub`, and `email`) are
  Catalog-Governed Claims.

Grant binding:
: The IdP Authorization Server selects the binding as described in
  {{Section 9.11 of ID-JAG}}. When the client presents a valid DPoP
  proof, ID-JAG requires the IdP Authorization Server to bind the
  ID-JAG, either to the DPoP key or, if the client also presents a
  certificate, to that certificate ({{Section 9.11.1.1 of ID-JAG}}).
  Any of those methods that the approved subset permits satisfies that
  requirement ({{entry-grant-binding}}). Binding to a client
  certificate is otherwise a matter of IdP Authorization Server policy
  ({{Section 9.11.2.1 of ID-JAG}}).

Composition:
: The checks in {{jwt-composition}} apply. Independently of the
  entries it publishes, a Resource Authorization Server that advertises
  this profile includes `urn:ietf:params:oauth:grant-type:jwt-bearer` in
  `grant_types_supported` ({{Section 7.2 of ID-JAG}}).

Resource selection:
: ID-JAG recommends, but does not require, the `resource` parameter of
  the Token Exchange request ({{Section 4.3 of ID-JAG}}). When that
  parameter is absent, the IdP Authorization Server selects an entry
  only if its policy determines one protected resource for the requested
  audience ({{issuing}}).

Tenant selection:
: The IdP Authorization Server selects the Tenant Object in the same
  way that it determines the `aud_tenant` claim
  ({{Section 6.4 of ID-JAG}}): from the requested audience and resource
  and the issuer tenant's configuration for that Resource Authorization
  Server.

Error responses:
: When the IdP Authorization Server does not issue an ID-JAG because no
  approved entry permits the request, it responds to the Token Exchange
  request with the `access_denied` error code, which
  {{Section 4.3.4.3 of ID-JAG}} profiles for a target that the client or
  End-User is not authorized to use.

Narrowed rules:
: An ID-JAG issued using an entry always contains the `resource` claim
  with exactly one resource identifier; {{Section 3.1 of ID-JAG}} makes
  the claim OPTIONAL and permits several resource identifiers. A client
  that needs access to several resources obtains an ID-JAG for each. An
  ID-JAG issued using an entry contains a Catalog-Governed Claim only if
  the entry lists it, which narrows {{Section 3.1 of ID-JAG}}, where any
  claim that would be valid in an ID Token can appear. This narrowing
  covers the `email` and `aud_sub` claims, which
  {{Section 3.1 of ID-JAG}} recommends for subject resolution; a
  Resource Authorization Server that uses them lists them in its
  entries. The requirement to include `target_tenant` in a Tenant Object
  ({{tenant-object}}) is stricter than {{Section 4.4.1 of ID-JAG}},
  which also allows a Resource Authorization Server to identify the
  recipient tenant from its trust relationship with the issuer.

Clients:
: An ID-JAG contains the client identifier of the client at the Resource
  Authorization Server ({{Section 3.1 of ID-JAG}}). The IdP
  Authorization Server obtains that value as described in
  {{Section 5.1 of ID-JAG}}, for example from a client identifier that
  is shared across trust domains {{CIMD}}; the catalog does not provide
  it ({{clients}}).

The following is an example of the claims of an ID-JAG issued using the
Tenant Object in {{tenant-object}} and the entry in {{example-catalog}}:

~~~ json
{
  "iss": "https://idp.example/",
  "sub": "user-123",
  "tenant": "acme-workforce",
  "aud": "https://ras.example/",
  "aud_tenant": "customer-acme",
  "client_id": "ras-client-789",
  "resource": "https://api.example/messages",
  "scope": "messages.read",
  "email": "user@acme.example",
  "sub_id": {
    "format": "saml-nameid",
    "issuer": "https://idp.example/saml",
    "nameid": "user@acme.example"
  },
  "cnf": {
    "jkt": "0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I"
  },
  "iat": 1788897700,
  "exp": 1788898000,
  "jti": "grant-456"
}
~~~

# Extensibility {#extensibility}

This specification can be extended in the following ways:

Grant Profiles:
: A specification can apply the catalog to another Grant Profile by
  providing a Grant Profile Definition ({{profile-definitions}}). A
  Grant Issuer does not use an entry whose `profile` it does not
  implement
  ({{validation}}); that entry does not affect the validity of the other
  entries.

Members:
: A specification can define additional members of the objects defined
  in this specification by registering them in the "OAuth Grant Catalog
  Members" registry ({{iana-members}}). A Grant Issuer ignores members
  it does not understand ({{model-conventions}}). A member of an entry,
  or of an object within an entry, whose value affects what an
  administrator approves or which grants the entry permits is registered
  with "Entry Semantics" set to "Yes"; a change to its value then
  requires a new revision ({{revision}}).
: An extension member is either advisory or restrictive. An advisory
  member informs approval, for example by grouping entries for
  presentation to administrators. A Grant Issuer that ignores an
  advisory member still issues only grants within the entry's
  Authorization Envelope as this specification defines it. A restrictive
  member narrows the grants that the Resource Authorization Server
  accepts below that Authorization Envelope, for example by constraining
  the values in authorization details objects. A Grant Issuer that
  ignores a restrictive member issues grants that the Resource
  Authorization Server rejects, after their claims have been disclosed
  to the client. A specification MUST NOT define a restrictive member
  unless it also defines a mechanism that prevents Grant Issuers that do
  not implement the member from using the entry, such as a distinct
  Grant Profile identifier. This specification defines no such mechanism
  ({{open-issues}}).

Grant binding methods:
: A specification can define additional grant binding methods by
  registering them in the "OAuth Grant Catalog Grant Binding Methods"
  registry ({{iana-binding-methods}}).

Retrieval Bindings:
: A specification can define another Retrieval Binding that meets the
  requirements of {{binding-requirements}}.

Additional capabilities, such as paging of large Relationship Views or
delta retrieval, can be defined as extensions that use the same objects.
Statements of what a scope value or authorization details type
authorizes belong in public metadata {{AUTHORITY-METADATA}} rather than
in the catalog ({{authority-metadata}}). Constraints on the values of
authorization details objects are restrictive and require a mechanism of
the kind described above.

# Security Considerations {#security}

The security considerations of {{RFC7521}}, {{RFC7523}}, {{RFC8725}},
and {{RFC9700}}, and of the Grant Profile of each entry, apply to this
specification. For the ID-JAG profile, the security considerations of
{{ID-JAG}} apply.

## Catalog Is Not an Authorization Decision {#sec-not-authorization}

A catalog describes what a Resource Authorization Server is prepared to
accept; it does not authorize any request ({{permitted}}). The
Resource Authorization Server can still reject a grant that a Grant
Issuer issues within an Authorization Envelope. A catalog is also not an
exhaustive list of the grants that the Resource Authorization Server
accepts: the Resource Authorization Server can accept grants issued
under other arrangements, or a grant issued under an entry that it has
since withdrawn. Whether a
grant must match a current entry is a matter of Resource Authorization
Server policy.

Conversely, the Authorization Envelope protects the Resource
Authorization Server only to the extent that the Grant Issuer observes
it. A Resource Authorization Server that requires a property of every
grant it accepts enforces that property at redemption rather than
relying on the Grant Issuer's catalog processing.

## Changes in Scope Meaning {#sec-scope-meaning}

An entry lists scope values and authorization details types; it does not
state the authority they represent at the protected resource. An
approval therefore bounds the grants that the Grant Issuer constructs,
not what those grants authorize. For example, if the protected resource
changes `messages.write` so that it also permits deleting messages, no
member of the entry changes, its revision does not change, and approvals
that include `messages.write` continue to apply. A Resource
Authorization Server that changes the authority of an existing scope
value can instead publish a new scope value, so that the change requires
new approval. Authority metadata {{AUTHORITY-METADATA}} can describe
what a scope value authorizes. A Grant Issuer that retains the
description an administrator approved can detect a change to it
({{authority-metadata}}), but the description is advisory and does not
bind the protected resource.

## Relationship Authorization and Assertion Trust {#sec-relationship}

A catalog discloses configuration that is specific to an Issuer
Relationship. Trust in a Grant Issuer's signing keys establishes that
the Grant Issuer can issue grants; it does not establish that the Grant
Issuer is entitled to learn the Resource Authorization Server's
configuration for every issuer tenant identifier it can name. For this
reason, an Onboarding Arrangement authorizes each Issuer Relationship
separately ({{relationship-authorization}}), and the Resource
Authorization Server determines the Relationship View only from the
authenticated relationship. A Grant Issuer that uses a multi-tenant
issuer signs catalog access assertions for all of its tenants with the
same keys; a compromise of those keys or of the Grant Issuer's signing
service discloses the catalogs of every Issuer Relationship that names
the issuer. An Onboarding Arrangement authorizes an issuer tenant
identifier, not the organization behind it: if an issuer reassigned a
tenant identifier to another tenant, contrary to the definition in
{{conventions}}, the new tenant would receive the catalog content of the
old one.

## Protected Discovery and Client Authentication {#sec-protected-discovery}

{{Section 9.7 of ID-JAG}} permits a protected mechanism by which an
authenticated client determines whether an ID-JAG from a particular
issuer would be accepted for that client, and requires that mechanism to
authenticate the client and to respond specifically to that client. The
catalog is disclosed to a Grant Issuer rather than to a client, and is
specific to an Issuer Relationship. The Grant Issuer's signature on the
catalog access assertion, not a client registration, binds a response to
that relationship, so this specification leaves client authentication at
the token endpoint to the policy of the Resource Authorization Server
({{token-request}}).

When the Onboarding Arrangement does not require client authentication,
the catalog access assertion is the only credential the Grant Issuer
presents. A catalog access assertion that is not DPoP-bound is a bearer
credential until the Resource Authorization Server accepts it. The
replay check in {{assertion-validation}} prevents reuse of the assertion
but not its first use by a party that obtains it, and binding only the
catalog access token does not prevent that use. A DPoP-bound catalog
access assertion cannot be used without the private key that its `cnf`
claim identifies, and the Resource Authorization Server rejects it when
it is presented with the `urn:ietf:params:oauth:grant-type:jwt-bearer`
grant type ({{token-request}}). DPoP binding demonstrates possession of
a key that the Grant Issuer names under its own signature. It does not
authenticate a client and is not client authentication as
{{Section 9.7 of ID-JAG}} requires for client-facing discovery. The
assertion's short lifetime, its audience restriction, and TLS further
limit the exposure. A Resource Authorization Server weighs the
confidentiality of its catalog content when deciding whether its
Onboarding Arrangements require client authentication or DPoP-bound
catalog access assertions.

## Catalog Access Token Disclosure {#sec-token-disclosure}

A catalog access token permits retrieval of a Relationship View until it
expires or the relationship ends. The Grant Issuer sends it only to the
catalog endpoint named in validated metadata, never to a URL obtained
from a response, and does not follow redirections ({{view-document}}).
The catalog access token is audience-restricted to the catalog resource
and limited to the `authorization_grant_catalog.read` scope, so a token
disclosed elsewhere cannot be used at other protected resources of the
Resource Authorization Server.

## Response Authenticity {#sec-authenticity}

Relationship View documents are not signed. Their
authenticity and integrity rest on TLS server authentication of the
Resource Authorization Server's catalog resource and on the validation
of the Resource Authorization Server's metadata
({{Section 3.3 of RFC8414}}). An attacker able to alter catalog
responses can cause an administrator of the Grant Issuer to approve
grants that the Resource Authorization Server did not offer, or to deny
approval of grants that it did offer. The attacker cannot cause the
Resource Authorization Server to accept a grant that its current policy
rejects. The `revision`
member is not an integrity mechanism ({{revision}}).

## Stale Catalogs and Relationship Termination {#sec-stale}

A Grant Issuer can hold a catalog that no longer reflects the Resource
Authorization Server's configuration. When the Resource Authorization
Server narrows what it accepts, a Grant Issuer that has not refreshed
issues grants that the Resource Authorization Server rejects after
their claims have been disclosed to the client. Such grants affect
availability and disclosure, not authorization. When a relationship
ends, the Resource Authorization Server stops disclosing catalog content
immediately ({{relationship-authorization}}). The maximum age
({{installation}}) bounds how long the Grant Issuer continues to issue
grants under state it can no longer refresh.

## Revision Integrity and Approval {#sec-revision}

A Resource Authorization Server that changes the normative semantics of
an entry without changing its revision causes the Grant Issuer to
continue issuing grants under an approval that no longer matches the
entry. The Grant Issuer detects such a change only when it holds an
earlier copy of the entry with the same revision ({{validation}}). A
revision does not bind the Resource Authorization Server to the meaning
that the Grant Issuer approved, and a grant issued before a change can
be redeemed after it. Requiring new approval for every revision change
({{reconciliation}}) avoids relying on a Grant Issuer's comparison of
two revisions to decide that a change only narrows an entry; an
incorrect comparison would expand authorization.

## Tenant Confusion {#sec-tenant}

If a Grant Issuer derived a grant's target tenant from a Tenant Object
identifier, from the number of tenants, or from the form of an issuer
identifier, a grant intended for one tenant could be accepted for
another. The Resource Authorization Server therefore supplies
`target_tenant` whenever its issuer identifier does not identify the
tenant, and the Grant Issuer uses only that value ({{tenant-object}}).
Similarly, an entry obtained for one Issuer Relationship is used only
for grants whose issuer and issuer tenant identify that relationship
({{issuing}}), so that a catalog disclosed to one issuer tenant does
not determine the grants issued for subjects of another.

## Cross-JWT Confusion {#sec-cross-jwt}

A Resource Authorization Server that accepts JWT authorization grants
and catalog access assertions at the same token endpoint relies on the
`typ` JOSE header parameter to distinguish them
({{assertion-validation}}, {{jwt-typing}}). Without that check, a grant
could be presented as a catalog access assertion, or a catalog access
assertion as a grant; their claims overlap, and the `sub` claim of a
catalog access assertion is an issuer identifier. When the client
identifier assigned to a Grant Issuer equals its issuer identifier, a
catalog access assertion also has the `iss`, `sub`, and `aud` values of
a client assertion ({{Section 3 of RFC7523}}), so one intercepted JWT
could serve as both the grant and the client authentication of a
request. The `typ` check prevents a grant and a catalog access assertion
from being substituted for each other. A validator of client assertions
that ignores `typ` does not prevent the use of a catalog access
assertion as a client assertion. The Resource Authorization Server
therefore also refuses the catalog access assertion as a client
assertion of the same request, and the Grant Issuer signs client
assertions with a separate key ({{token-request}}).
{{Section 2.8 of RFC8725}} describes this class of attack. For the same
reason, an entry is used only for grants of its own Grant Profile
({{issuing}}), even when the grants of another profile carry similar
claims.

## External References {#sec-external}

Entries and authorization details types metadata contain URIs that refer
to other servers: the `documentation_uri`, `definition_uri`, and
`schema_uri` members, and the `resource` of each entry, whose Protected
Resource Metadata the Grant Issuer requests during composition
({{composition}}).
A Resource Authorization Server, or an attacker that can alter catalog
responses, chooses these URIs ({{Section 7.7 of RFC9728}}). A Grant
Issuer that dereferences them does not send a credential obtained
through a Retrieval Binding ({{rar-metadata}}), applies the protections
against server-side request forgery that it applies to other URLs
obtained from external parties, and treats the content as untrusted
input. Retrieval of such a URI is
not authenticated by the Retrieval Binding, so content obtained from it
is not catalog content. A Grant Issuer can cache Protected Resource
Metadata and limit how many such requests one refresh makes.

A Grant Issuer presents `display`, `description`, `documentation_uri`,
and `definition_uri` to administrators as untrusted text. Because these
members are outside the normative semantics of an entry, they can change
without a new revision ({{revision}}), and an approval does not cover
later changes to them.

## Resource Consumption {#sec-resources}

A Relationship View can be large. A Grant Issuer SHOULD limit the size
of the responses it accepts, and treats a response that exceeds its
limits as an incomplete retrieval ({{installation}}). A Resource
Authorization Server can limit the rate at which it accepts catalog
access assertions and requests to the catalog endpoint from an Issuer
Relationship.

# Privacy Considerations {#privacy}

The privacy considerations of the Grant Profile of each entry apply to
this specification. For the ID-JAG profile, the privacy considerations
of {{ID-JAG}} apply.

A catalog discloses business relationships: which Resource Authorization
Server tenants are associated with an issuer tenant, and the names those
tenants use. {{Section 9.7 of ID-JAG}} identifies such information as
deployment-sensitive when it is published in metadata. This
specification limits disclosure of that information to an authenticated
and authorized Issuer Relationship ({{relationship-authentication}}),
requires responses that caches do not store ({{refresh}}), and prohibits
error responses that reveal other issuers, tenants, or catalogs
({{errors}}).

An entry states which claims the Resource Authorization Server
requires, which Catalog-Governed Claims it accepts, and which subject
identifier formats it accepts. Every other claim a grant conveys is a
reserved claim of its Grant Profile ({{entry-claims}}).
This lets an administrator of the Grant Issuer decide before issuance
which claims the Grant Issuer releases and to which Resource
Authorization Server tenants. A Grant Issuer that cannot release a
required claim for a subject does not issue a grant using that entry
({{entry-claims}}), rather than releasing the claim without approval.

The Grant Issuer stores catalog content as configuration, protects that
content as it protects other relationship-specific configuration, and
makes the content accessible only to administrators who manage the
corresponding Issuer Relationship.

Dereferencing a URI contained in a catalog, such as a
`documentation_uri` or `definition_uri`, can reveal to the server
hosting it that the Grant Issuer has a relationship with the Resource
Authorization Server. A Grant Issuer can avoid such retrieval or perform
it through an intermediary.

# IANA Considerations {#iana}

## OAuth Authorization Server Metadata Registry {#iana-as-metadata}

This specification requests registration of the following value in the
"OAuth Authorization Server Metadata" registry established by
{{RFC8414}}.

Metadata Name:
: `authorization_grant_catalog_endpoint`

Metadata Description:
: URL of the OAuth Authorization Grant Catalog endpoint, which is also
  the resource indicator for requesting a catalog access token

Change Controller:
: IETF

Specification Document(s):
: {{as-metadata}} of this specification

## Media Types Registry {#iana-media-types}

This specification requests registration of the following media types
in the "Media Types" registry, in the manner described in {{RFC6838}}.

### application/oauth-grant-catalog+json

Type name:
: application

Subtype name:
: oauth-grant-catalog+json

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: binary; a document of this type is a JSON object {{RFC8259}} encoded
  in UTF-8.

Security considerations:
: See {{security}} of this specification.

Interoperability considerations:
: N/A

Published specification:
: {{oauth-binding}} of this specification

Applications that use this media type:
: Authorization servers that publish an OAuth Authorization Grant
  Catalog, and authorization servers that retrieve one to issue grants.

Fragment identifier considerations:
: The syntax and semantics of fragment identifiers specified for
  `+json` in {{Section 3.1 of RFC6839}} apply.

Additional information:
: Deprecated alias names for this type: N/A
: Magic number(s): N/A
: File extension(s): N/A
: Macintosh file type code(s): N/A

Person & email address to contact for further information:
: Karl McGuinness, public@karlmcguinness.com

Intended usage:
: COMMON

Restrictions on usage:
: none

Author:
: Karl McGuinness, public@karlmcguinness.com

Change controller:
: IETF

### application/oauth-grant-catalog-access+jwt

Type name:
: application

Subtype name:
: oauth-grant-catalog-access+jwt

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: binary; a catalog access assertion is a JWT, and a JWT is a sequence
  of base64url-encoded values separated by period (".") characters.

Security considerations:
: See {{security}} of this specification.

Interoperability considerations:
: N/A

Published specification:
: {{assertion}} of this specification

Applications that use this media type:
: Authorization servers that request a catalog access token for an
  OAuth Authorization Grant Catalog, and authorization servers that
  issue one.

Fragment identifier considerations:
: N/A

Additional information:
: Deprecated alias names for this type: N/A
: Magic number(s): N/A
: File extension(s): N/A
: Macintosh file type code(s): N/A

Person & email address to contact for further information:
: Karl McGuinness, public@karlmcguinness.com

Intended usage:
: COMMON

Restrictions on usage:
: none

Author:
: Karl McGuinness, public@karlmcguinness.com

Change controller:
: IETF

The registration of `application/oauth-grant-catalog-access+jwt`
establishes the `typ` JOSE header parameter value
`oauth-grant-catalog-access+jwt` used by catalog access assertions
({{assertion}}).

## OAuth Grant Catalog Members Registry {#iana-members}

This specification establishes the "OAuth Grant Catalog Members"
registry. The registry records the members of the objects used by the
OAuth Authorization Grant Catalog. Values are registered on a
Specification Required basis after a three-week review period on the
oauth-ext-review@ietf.org mailing list, on the advice of one or more
Designated Experts {{RFC8126}}.

Designated Experts verify that a proposed member is defined clearly
enough for interoperable implementation and does not duplicate an
existing member of the same object. For a member of an entry or of an
object within an entry, they also verify that the "Entry Semantics"
value is correct and that either the member is advisory or its
specification defines a mechanism that prevents Grant Issuers that do
not implement the member from using the entry ({{extensibility}}).

### Registration Template {#iana-members-template}

Member Name:
: The name of the member, which is case-sensitive.

Object:
: The object in which the member appears: "Tenant Object", "Grant
  Catalog", "Entry", "Scope", "Scope Value", "Authorization Details",
  "Authorization Details Type", "Claim", "Subject Identifier", or
  "Grant Binding"; or, for the OAuth Retrieval Binding, "Relationship
  View Document" or "Issuer Relationship". A member name is unique
  within an object.

Description:
: A brief description of the member.

Entry Semantics:
: "Yes" if the member is part of the normative semantics of an entry
  ({{revision}}), "No" if it is not, and "N/A" for a member of the
  Relationship View Document, Issuer Relationship, Tenant Object, or
  Grant Catalog.

Change Controller:
: For IETF Stream RFCs, "IETF". For others, the name of the responsible
  party.

Specification Document(s):
: Reference to the document or documents that specify the member.

### Initial Registry Contents {#iana-members-initial}

The initial contents of the registry are listed in the following
tables. For every row, the Change Controller is "IETF" and the
Specification Document is the section of this specification listed in
the table.

| Member Name | Object | Entry Semantics | Section |
| --- | --- | --- | --- |
| `id` | Tenant Object | N/A | {{tenant-object}} |
| `display` | Tenant Object | N/A | {{tenant-object}} |
| `description` | Tenant Object | N/A | {{tenant-object}} |
| `target_tenant` | Tenant Object | N/A | {{tenant-object}} |
| `grant_catalog` | Tenant Object | N/A | {{tenant-object}} |
| `issuer` | Grant Catalog | N/A | {{grant-catalog}} |
| `id` | Grant Catalog | N/A | {{grant-catalog}} |
| `display` | Grant Catalog | N/A | {{grant-catalog}} |
| `description` | Grant Catalog | N/A | {{grant-catalog}} |
| `entries` | Grant Catalog | N/A | {{grant-catalog}} |
{: title="Initial Tenant Object and Grant Catalog Members"}

| Member Name | Object | Entry Semantics | Section |
| --- | --- | --- | --- |
| `id` | Entry | No | {{entry}} |
| `revision` | Entry | No | {{entry}} |
| `display` | Entry | No | {{entry}} |
| `description` | Entry | No | {{entry}} |
| `profile` | Entry | Yes | {{entry}} |
| `resource` | Entry | Yes | {{entry}} |
| `scope` | Entry | Yes | {{entry}} |
| `authorization_details` | Entry | Yes | {{entry}} |
| `claims` | Entry | Yes | {{entry}} |
| `sub_id` | Entry | Yes | {{entry}} |
| `grant_binding` | Entry | Yes | {{entry}} |
{: title="Initial Entry Members"}

| Member Name | Object | Entry Semantics | Section |
| --- | --- | --- | --- |
| `requirement` | Scope | Yes | {{entry-scope}} |
| `values` | Scope | Yes | {{entry-scope}} |
| `scope` | Scope Value | Yes | {{entry-scope}} |
| `requirement` | Scope Value | Yes | {{entry-scope}} |
| `display` | Scope Value | No | {{entry-scope}} |
| `description` | Scope Value | No | {{entry-scope}} |
| `requirement` | Authorization Details | Yes | {{entry-authorization-details}} |
| `types` | Authorization Details | Yes | {{entry-authorization-details}} |
| `type` | Authorization Details Type | Yes | {{entry-authorization-details}} |
| `requirement` | Authorization Details Type | Yes | {{entry-authorization-details}} |
| `documentation_uri` | Authorization Details Type | No | {{entry-authorization-details}} |
| `display` | Authorization Details Type | No | {{entry-authorization-details}} |
| `description` | Authorization Details Type | No | {{entry-authorization-details}} |
| `claim` | Claim | Yes | {{entry-claims}} |
| `requirement` | Claim | Yes | {{entry-claims}} |
| `definition_uri` | Claim | No | {{entry-claims}} |
| `display` | Claim | No | {{entry-claims}} |
| `description` | Claim | No | {{entry-claims}} |
| `requirement` | Subject Identifier | Yes | {{entry-sub-id}} |
| `formats` | Subject Identifier | Yes | {{entry-sub-id}} |
| `requirement` | Grant Binding | Yes | {{entry-grant-binding}} |
| `methods` | Grant Binding | Yes | {{entry-grant-binding}} |
{: title="Initial Members of Objects Within an Entry"}

The Relationship View document and its Issuer Relationship object are
defined by the OAuth Retrieval Binding ({{oauth-binding}}) rather than
by the data model. Their initial members are:

| Member Name | Object | Entry Semantics | Section |
| --- | --- | --- | --- |
| `issuer` | Relationship View Document | N/A | {{view-document}} |
| `issuer_relationship` | Relationship View Document | N/A | {{view-document}} |
| `tenants` | Relationship View Document | N/A | {{view-document}} |
| `grant_catalogs` | Relationship View Document | N/A | {{view-document}} |
| `issuer` | Issuer Relationship | N/A | {{view-document}} |
| `tenant` | Issuer Relationship | N/A | {{view-document}} |
{: title="Initial Relationship View Document Members"}

## OAuth Grant Catalog Grant Binding Methods Registry {#iana-binding-methods}

This specification establishes the "OAuth Grant Catalog Grant Binding
Methods" registry. The registry records the names of methods by which a
grant is bound to a key or certificate, for use in the `methods` member
of the `grant_binding` member of an entry ({{entry-grant-binding}}).
Values are registered on a Specification Required basis after a
three-week review period on the oauth-ext-review@ietf.org mailing list,
on the advice of one or more Designated Experts {{RFC8126}}. Designated
Experts verify that the specification defines how the Grant Issuer
binds a grant, how the Resource Authorization Server verifies the
binding, and how the binding is represented in grants, including, for
JWT authorization grants, the confirmation member of the `cnf` claim.

### Registration Template {#iana-binding-methods-template}

Method Name:
: The name of the method, which is case-sensitive.

Description:
: A brief description of the method, including the confirmation member
  of the `cnf` claim that represents it in a JWT authorization grant.

Change Controller:
: For IETF Stream RFCs, "IETF". For others, the name of the responsible
  party.

Specification Document(s):
: Reference to the document or documents that specify the method.

### Initial Registry Contents {#iana-binding-methods-initial}

Method Name:
: `dpop`

Description:
: Binding to a DPoP key, represented in a JWT authorization grant by the
  `jkt` member of the `cnf` claim

Change Controller:
: IETF

Specification Document(s):
: {{entry-grant-binding}} and {{jwt-representation}} of this
  specification, {{RFC9449}}, {{JWT-DPOP}}

Method Name:
: `mtls`

Description:
: Binding to an X.509 certificate, represented in a JWT authorization
  grant by the `x5t#S256` member of the `cnf` claim

Change Controller:
: IETF

Specification Document(s):
: {{entry-grant-binding}} and {{jwt-representation}} of this
  specification, {{RFC8705}}

## OAuth Scope Value

This specification defines the scope value
`authorization_grant_catalog.read` ({{token-request}}). There is no IANA
registry for OAuth scope values, and this specification requests no
registration for it.

--- back

# Example Grant Catalog {#example-catalog}

The following is an example of a Grant Catalog with one entry for the
ID-JAG profile ({{id-jag}}). The entry describes ID-JAGs for the
protected resource `https://api.example/messages`. Every ID-JAG issued
using the entry is bound to a DPoP key and contains the scope value
`messages.read`, the `email` claim, and a `sub_id` claim in the
`saml-nameid` format. An ID-JAG issued using the entry can also contain
the scope value `messages.write` and authorization details objects of
type `message_channel`.

~~~ json
{
  "issuer": "https://ras.example/",
  "id": "standard",
  "display": "Standard API access",
  "entries": [
    {
      "id": "messages-api-id-jag",
      "revision": "5",
      "display": "Messages API using ID-JAG",
      "profile": "urn:ietf:params:oauth:grant-profile:id-jag",
      "resource": "https://api.example/messages",
      "scope": {
        "requirement": "required",
        "values": [
          {
            "scope": "messages.read",
            "requirement": "required",
            "display": "Read messages",
            "description": "Read messages available to the subject."
          },
          {
            "scope": "messages.write",
            "display": "Write messages",
            "description": "Create messages as the subject."
          }
        ]
      },
      "authorization_details": {
        "requirement": "optional",
        "types": [
          {
            "type": "message_channel",
            "display": "Message channel",
            "documentation_uri":
              "https://docs.example/rar/message-channel"
          }
        ]
      },
      "claims": [
        {
          "claim": "email",
          "requirement": "required",
          "display": "Email address"
        }
      ],
      "sub_id": {
        "requirement": "required",
        "formats": ["saml-nameid"]
      },
      "grant_binding": {
        "requirement": "required",
        "methods": ["dpop"]
      }
    }
  ]
}
~~~

# Example of Alternative Entries {#example-alternatives}

A Resource Authorization Server accepts ID-JAGs for the protected
resource `https://api.example/reports` that carry either the scope value
`reports.read` or the scope value `reports.export`, but not both. The
Resource Authorization Server requires the `department` claim only with
`reports.export`. A single
entry cannot express either rule ({{combinations}}), so the Resource
Authorization Server publishes two entries:

~~~ json
{
  "issuer": "https://ras.example/",
  "id": "reports",
  "entries": [
    {
      "id": "reports-read",
      "revision": "1",
      "profile": "urn:ietf:params:oauth:grant-profile:id-jag",
      "resource": "https://api.example/reports",
      "scope": {
        "requirement": "required",
        "values": [
          {
            "scope": "reports.read",
            "requirement": "required"
          }
        ]
      }
    },
    {
      "id": "reports-export",
      "revision": "1",
      "profile": "urn:ietf:params:oauth:grant-profile:id-jag",
      "resource": "https://api.example/reports",
      "scope": {
        "requirement": "required",
        "values": [
          {
            "scope": "reports.export",
            "requirement": "required"
          }
        ]
      },
      "claims": [
        {
          "claim": "department",
          "requirement": "required"
        }
      ]
    }
  ]
}
~~~

An ID-JAG issued using the entry `reports-read` contains the scope value
`reports.read` and does not contain the `department` claim, which is a
Catalog-Governed Claim that the entry does not list. An ID-JAG issued
using the entry `reports-export` contains the scope value
`reports.export` and the `department` claim. No ID-JAG contains both
scope values, because each grant is issued using exactly one entry
({{permitted}}).

# Example Approval Lifecycle {#example-lifecycle}

The following sequence shows how an approval of the entry in
{{example-catalog}}, for the Tenant Object `tenant-123`, changes as the
Relationship View changes:

1. The Grant Issuer installs a Relationship View in which `tenant-123`
   references the Grant Catalog `standard`, whose entry
   `messages-api-id-jag` has the revision `5`. An administrator approves
   the scope value `messages.read` for that entry, and the Grant Issuer
   issues ID-JAGs with that scope value.

2. A refresh fails with a transport error. The Grant Issuer installs
   nothing and continues to issue ID-JAGs under the installed state,
   within the maximum age it has configured ({{installation}}).

3. The Resource Authorization Server removes `tenant-123` from the
   Issuer Relationship. The next refresh retrieves a Relationship View
   without it, and the Grant Issuer suspends issuance for `tenant-123`
   ({{installation}}).

4. The Resource Authorization Server restores `tenant-123` with the same
   `target_tenant` and `grant_catalog`, and the Grant Catalog still
   contains revision `5` of the entry. The approval applies again
   without new approval ({{reconciliation}}).

5. The Resource Authorization Server adds the optional scope value
   `messages.archive` to the entry and changes its revision to `6`. The
   Grant Issuer stops issuing ID-JAGs under the approval for revision
   `5`, even though the change does not affect `messages.read`. This
   interruption is the operational cost of the rule in
   {{reconciliation}}; whether to relax that rule is an open issue
   ({{open-issues}}).

6. An administrator reviews revision `6` and approves `messages.read`
   again. The Grant Issuer resumes issuing ID-JAGs with that scope
   value.

# Design Rationale {#rationale}

This appendix explains choices that are not evident from the
requirements alone.

Data model separate from retrieval:
: Deployments differ in the administrative relationship that already
  exists between a Grant Issuer and a Resource Authorization Server.
  Defining the objects independently of retrieval lets the OAuth
  Retrieval Binding and other Retrieval Bindings deliver identical
  objects that feed the same Grant Issuer policy.

Complete documents and no merge rules:
: Each Tenant Object references one complete Grant Catalog, and Grant
  Catalogs are never paged or merged. A Grant Issuer can therefore
  determine what a tenant is offered from one object, and a partial
  retrieval can never be mistaken for a complete one.

One response per Relationship View:
: Returning the complete Relationship View in one response gives the
  Grant Issuer one consistent state without a manifest of versions, and
  makes a withdrawal the absence of an object from a validated response.
  The cost is retransmitting the whole view when anything changes;
  compression and conditional requests address that until deployments
  show a need for incremental retrieval.

Authority in public metadata, eligibility in the catalog:
: What a scope value or authorization details type authorizes is the
  same for every Issuer Relationship, so it belongs in public metadata,
  such as authority metadata {{AUTHORITY-METADATA}}. Which scope values
  and authorization details types a relationship may use is specific to
  the relationship, so it belongs in the catalog. Entries identify items
  by the protocol values that authority metadata uses, so the two
  compose without a mapping ({{authority-metadata}}). The same division
  settles several related questions. The catalog provides no client
  inventory ({{clients}}). A Grant Issuer synchronizes a complete
  Relationship View with conditional requests ({{refresh}}) rather than
  retrieving catalog content for each grant. An entry can list a scope
  value that public metadata omits, because `scopes_supported` can be
  incomplete, but not an authorization details type that a complete
  public enumeration excludes ({{composition}}). An entry bounds which
  scope values and authorization details types a grant conveys, not the
  values within an authorization details object; such constraints would
  be restrictive members ({{extensibility}}).

Envelope without alternatives:
: An Authorization Envelope that is a product of independent selections
  can be evaluated without a rule language, and a Grant Issuer cannot
  misread a rule that does not exist. Separate entries express the rules
  that a product cannot ({{combinations}}).

Revision rather than digest:
: A digest changes when presentation text changes and requires
  canonicalization. An opaque revision that changes whenever the
  normative semantics change, and need not change otherwise, lets the
  Grant Issuer retain approvals across presentation changes, while
  entity tags handle caching of representations.

Optional claims listed explicitly:
: Every claim a grant conveys, other than the reserved claims of its
  Grant Profile, is listed in the entry ({{entry-claims}}).
  Exempting some optional claims, such as an email address, would leave
  their release outside the administrator's approval, with no
  principled line between them and related claims, such as one that
  states whether that email address is verified.

One resource per grant:
: An entry, and so a grant issued using it, names one protected
  resource. This avoids rules for how scope values and authorization
  details apply across resources, and keeps each approval about one
  resource. A client that needs several resources obtains several
  grants.

Signed relationship assertion:
: The catalog access assertion states the Issuer Relationship (an issuer
  identifier and, when the issuer is multi-tenant, an issuer tenant
  identifier) under the signature of the keys that sign the Grant
  Issuer's grants. The Resource Authorization Server can obtain those
  keys from the issuer's metadata. Every Resource Authorization Server
  thereby receives the same standardized provenance for the
  relationship, reuses the trust it already places in the issuer, and
  binds catalog access to the identity that will issue the grants the
  catalog describes. A Resource Authorization Server could instead
  authorize a tenant selector against the relationships permitted to an
  authenticated client, but each Resource Authorization Server would
  then define its own selector and mapping. Client authentication
  remains available and composes with the assertion ({{token-request}}),
  and a Grant Issuer that needs proof of possession without a client
  registration binds the assertion to a DPoP key ({{assertion}}). A
  credential that is authorized for many issuer tenants carries broad
  authority whichever form it takes ({{sec-relationship}}).

No client inventory:
: The catalog does not provide a client inventory. Client identifiers
  can be shared across trust domains ({{CIMD}}) or established by
  registration, and defining an inventory would require additional
  disclosure policy and synchronization of client registration state.
  Not listing clients is a
  different decision from not describing what is available to one
  client: a view for one identified client that the Issuer Relationship
  is authorized to inspect discloses no inventory, and is an open issue
  ({{open-issues}}).

# Open Issues {#open-issues}
{:numbered="false"}

The following issues are open for discussion:

* Reapproval: every change of an entry's revision suspends issuance
  until an administrator approves the changed entry
  ({{reconciliation}}), even when the change only adds an item that no
  approval includes ({{example-lifecycle}}). The essential invariant is
  that no change expands an approval. Whether issuance can continue
  while the approved subset remains within the changed entry's
  Authorization Envelope, and what a Resource Authorization Server would
  publish to make that safe, are open.

* Token path: a catalog access token is issued only in response to a
  catalog access assertion ({{token-response}}). Whether a catalog
  access token can also be issued in response to other token requests
  that establish the same Issuer Relationship with the same guarantees,
  so that existing authorization infrastructure can be used, is open.

* Client-specific views: a Relationship View describes what is
  available to the Issuer Relationship, not to a particular application
  client ({{clients}}). Whether Resource Authorization Servers restrict
  grant configurations by client often enough to need an optional view
  for one identified client is open. Such a view would keep two parties
  distinct: the application client it describes, identified by its
  client identifier at the Resource Authorization Server, and the Issuer
  Relationship that retrieves it, which needs authorization to inspect
  that client's configuration. {{Section 9.7 of ID-JAG}} requires client
  authentication for client-facing discovery. The client context would
  include the target tenant where needed to identify the client
  registration ({{Section 6.2 of ID-JAG}}). The response would identify
  that context for comparison with the request. Installed state,
  revision comparisons, and approvals would be scoped to that context,
  and those approvals would apply only to grants identifying that
  client. The view would only narrow what the relationship permits.

* Must-understand extensions: a mechanism by which an entry marks the
  extension members that a Grant Issuer must implement to use it, so
  that restrictive extensions can be defined without a distinct Grant
  Profile identifier ({{extensibility}}). It is to be added before the
  format stabilizes, or with the first restrictive extension.

* Implementation experience: evidence that a Resource Authorization
  Server can generate entries and revisions from the configuration it
  enforces, and that independent Grant Issuers configure the same
  integration from one catalog with less manual work.

# Acknowledgments
{:numbered="false"}

This specification builds on the Identity Assertion JWT Authorization
Grant {{ID-JAG}} and on the JWT authorization grant {{RFC7523}}.

# Document History
{:numbered="false"}

-00

* Initial version.
