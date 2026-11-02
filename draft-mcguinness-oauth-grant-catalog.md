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
  RFC9700:
  RFC9728:
  RFC7523BIS: I-D.ietf-oauth-rfc7523bis
  OpenID.Enterprise:
    title: "OpenID Connect Enterprise Extensions 1.0 - draft 01"
    target: https://openid.net/specs/openid-connect-enterprise-extensions-1_0-01.html
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
catalog as input to the policy under which it issues grants; the catalog
is not an authorization decision. The data model is independent of grant
encoding and retrieval protocol. This specification applies it only to
the Identity Assertion JWT Authorization Grant (ID-JAG), and defines an
OAuth retrieval protocol that delivers the complete catalog in one
response.

--- middle

# Introduction {#introduction}

Several OAuth specifications define authorization grants that one
authorization server issues for a client to present to another
authorization server, typically in another trust domain. The Identity
Assertion JWT Authorization Grant (ID-JAG) {{ID-JAG}}, which an identity
provider issues, is one example. The issuing authorization server,
called the Grant Issuer in this specification, has to know which
tenants, protected resources, scope values, authorization details types
{{RFC9396}}, claims, subject identifiers, and grant bindings the
receiving authorization server, called the Resource Authorization
Server, accepts from it, because the Resource Authorization Server
rejects a grant that falls outside what it accepts.

Public metadata does not convey this information. OAuth 2.0
Authorization Server Metadata {{RFC8414}} and OAuth 2.0 Protected
Resource Metadata {{RFC9728}} are retrieved without authentication, are
generally not specific to a tenant, and describe what a server
implements rather than what it accepts from a particular Grant Issuer.
Publishing this information would advertise issuer-specific trust
relationships, which {{Section 9.7 of ID-JAG}} warns can disclose
deployment-sensitive information; that section also prohibits using the
`authorization_grant_profiles_supported` metadata member to disclose
issuer allow lists. Administrators of a Grant Issuer therefore copy
these values from the Resource Authorization Server's documentation
into local configuration by hand.

This specification makes that configuration available as authenticated
metadata, called the catalog. The Resource Authorization Server
publishes entries that describe the grants it accepts from one Grant
Issuer. An administrator of the Grant Issuer approves a subset of each
entry it uses, and the Grant Issuer issues grants within that subset. A
revision on each entry lets the Grant Issuer detect changes that need
review.

The catalog supports configuration and grant construction. It does not
establish trust between the servers, register clients, or grant access.
A catalog is input to policy authoring, not an authorization decision:
the Resource Authorization Server applies its current policy to every
grant it processes, whether or not the grant was issued using a catalog.

## Motivating Example {#motivating-example}

An enterprise uses its identity provider for single sign-on to a
software-as-a-service application. It wants an AI assistant, a client
registered at the application, to use ID-JAG to read messages through
the application's Messages API on behalf of employees.

Today, an administrator of the identity provider reads the application's
documentation and enters the following by hand: the application's issuer
identifier and the enterprise's tenant identifier at the application;
the resource
identifier of the Messages API; the scope value `messages.read`; and the
requirement that each ID-JAG be bound to a DPoP key. A mistake surfaces
only when the application rejects an ID-JAG, after the ID-JAG has
disclosed its claims to the client.

With a catalog, the identity provider retrieves, under its relationship
with the application, a Relationship View that lists the enterprise's
tenant and one entry stating these values ({{fig-model-example}}). The
administrator approves the entry instead of typing its values. When the
application later changes the entry, the identity provider detects the
new revision and asks for review instead of continuing under the old
approval. {{example-alternatives}} shows a richer case, with scope
values that cannot be combined in one grant.

The catalog replaces copied grant configuration; the rest of the
integration is unchanged: establishing the relationship (the Onboarding
Arrangement), registering the client at the application and mapping its
client identifier, mapping subjects, learning any restrictions the
application applies to that particular client, deciding which employees
and clients may use the integration, and the application's runtime
authorization decisions. Approving an entry constrains what grants
contain; what a scope value permits, and whether a request is allowed,
remain decisions of the application ({{sec-not-authorization}},
{{sec-scope-meaning}}).

## Relationship to Other Metadata {#other-metadata}

Each source of information used to issue a grant answers a different
question:

| Source | Question It Answers |
| --- | --- |
| Authorization Server Metadata {{RFC8414}} | Where are the Resource Authorization Server's endpoints, and which protocol features does it support? |
| Protected Resource Metadata {{RFC9728}} | Which authorization servers and capabilities does a protected resource advertise? |
| Authorization details types metadata {{RAR-METADATA}} | What structure does an authorization details object of a given type have? |
| Authority metadata {{AUTHORITY-METADATA}} | What does a scope value or authorization details type authorize? |
| Relationship View | Which grants does the Resource Authorization Server accept from this Issuer Relationship? |
| Grant Issuer approval | Which of those grants may the Grant Issuer issue? |
| Runtime authorization | Is this request allowed now? |
{: title="Sources of Information Used to Issue a Grant"}

{{composition}}, {{narrowing}}, and {{optional-metadata}} define how a
Grant Issuer combines these sources.

## Applicability {#applicability}

This specification applies to authorization grants that a Grant Issuer
issues for a client to present to a Resource Authorization Server,
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
of its Grant Profile convey those inputs ({{profile-definitions}}); this
specification provides one only for ID-JAG ({{id-jag}}). A retrieval
protocol authenticates the relationship and delivers the catalog
({{oauth-binding}}); this specification defines an OAuth retrieval
protocol.

The following are outside the scope of this specification:

* Enumeration or registration of application clients, and establishment
  of client identifier mappings ({{clients}}).

* Constraints on the values within an authorization details object
  ({{entry-authorization-details}}).

* Paging, change notification, delta retrieval, and push delivery of
  catalogs.

* Automatic reconciliation of Grant Issuer policy with a changed entry
  ({{installation}}).

* A mechanism by which a client discovers whether a grant from a
  particular Grant Issuer would be accepted ({{Section 9.7 of ID-JAG}}),
  which is an open issue ({{open-issues}}).

* Establishment of an Onboarding Arrangement
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
present to a Resource Authorization Server, and "client" means that
client, the application client. Where the Grant Issuer itself acts as a
client of the Resource Authorization Server ({{token-request}}), the
text says so. The issuer tenant of a grant is the tenant of the Grant
Issuer that issues it, and its target tenant is the Resource
Authorization Server tenant for which it is issued.

This specification defines the following terms:

tenant:
: An administrative boundary within an issuer, such as a customer
  organization, that scopes subjects, clients, and policy
  ({{Section 2.2 of ID-JAG}}). An issuer is multi-tenant when one issuer
  identifier hosts several tenants ({{Section 6.1 of ID-JAG}}).

tenant identifier:
: A string that, together with an issuer identifier, identifies exactly
  one tenant of that issuer. A tenant identifier is not reassigned by
  the issuer to a different tenant and is compared using simple string
  comparison ({{Section 6.2.1 of RFC3986}}).

Grant Issuer:
: An authorization server that consumes a Relationship View and issues
  grants that a client presents to a Resource Authorization Server.

Resource Authorization Server:
: An authorization server that accepts grants issued by a Grant Issuer
  and publishes Relationship Views. This specification uses the term as
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
: The relationship under which a Resource Authorization Server
  discloses catalog content to a Grant Issuer, identified by the Grant
  Issuer's issuer identifier and, when that issuer is multi-tenant, an
  issuer tenant identifier ({{relationship-identity}}).

Onboarding Arrangement:
: The agreement, established outside this protocol, under which a
  Resource Authorization Server discloses catalog content to an Issuer
  Relationship and determines the Resource Authorization Server tenants
  eligible for it ({{relationship-authorization}}).

Relationship View:
: The complete set of Tenant Objects, and the Grant Catalogs they
  reference, that a Resource Authorization Server makes available to one
  Issuer Relationship at a point in time ({{relationship-view}}).

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

Authorization Envelope:
: The set of grants that an entry, or an approved subset of an entry,
  permits ({{envelope}}).

# Protocol Overview {#overview}

{{fig-overview}} shows the protocol flow.

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
  metadata, which includes the location of the catalog for a retrieval
  protocol.

(B)
: The Grant Issuer authenticates an Issuer Relationship to the Resource
  Authorization Server using a retrieval protocol. The Resource
  Authorization Server verifies that an Onboarding Arrangement
  authorizes that relationship ({{relationship-authorization}}).

(C)
: The Resource Authorization Server returns the Relationship View for
  that Issuer Relationship.

(D)
: The Grant Issuer validates the Relationship View, composes each entry
  with public metadata, and installs the result ({{processing}}).

(E)
: An administrator of the Grant Issuer approves a subset of one or more
  entries for use by its clients and subjects ({{narrowing}}).

(F)
: When a client requests a grant, the Grant Issuer authenticates the
  client and the subject, applies its policy, and issues a grant within
  the approved subset of one entry ({{issuing}}).

(G)
: The client presents the grant to the Resource Authorization Server,
  which validates it and applies its current policy as specified by the
  Grant Profile.

Steps (A) through (E) occur when the Grant Issuer is configured and when
it refreshes the catalog, not for each grant. With the OAuth retrieval
protocol, the Grant Issuer signs a catalog access assertion, exchanges
it at the Resource Authorization Server's token endpoint for a catalog
access token, and uses that token to retrieve one complete Relationship
View ({{oauth-binding}}). This exchange is separate from the grants that
clients later request in step (F).

The design rests on the following principles, each stated
normatively in the sections cited.

* The Resource Authorization Server determines a Relationship View
  solely from the authenticated Issuer Relationship
  ({{relationship-authorization}}), and the Grant Issuer installs all of
  a Relationship View or none of it ({{installation}}).

* A grant is issued using one Tenant Object and one entry, and conveys
  an input that the entry governs only as the entry permits; every claim
  is either reserved by the Grant Profile or listed in the entry
  ({{envelope}}, {{entry-claims}}, {{issuing}}).

* Approval and runtime policy only narrow an entry. A new or changed
  entry, public metadata, and authority metadata never broaden an
  approval, and an entry cannot enable a value that public metadata
  excludes ({{narrowing}}, {{composition}}, {{installation}},
  {{optional-metadata}}).

* When the `revision` of an entry or the `grant_catalog` of a Tenant
  Object changes, issuance under approvals
  made for the previous value stops, and resumes only under an approval
  made for the new value ({{installation}}).

* A Grant Issuer ignores members it does not understand, and a
  specification that defines a restrictive member also defines a
  mechanism that prevents Grant Issuers that do not implement that
  member from using the entry ({{extensibility}}).

# Catalog Data Model {#data-model}

The objects in this section describe what a Resource Authorization
Server accepts in grants from one Issuer Relationship. A Relationship
View is the response for one Issuer Relationship. Its Tenant Objects
identify the target tenants eligible for that relationship, and its
Grant Catalogs contain the entries. Several Tenant Objects can reference
one Grant Catalog, so tenants that accept the same grants share entries,
while each approval remains specific to one Tenant Object
({{narrowing}}).

{{fig-model-example}} shows a complete Relationship View with one Tenant
Object and one Grant Catalog. The Tenant Object references the Grant
Catalog `read-only`, whose single entry permits ID-JAGs for the Messages
API. Grants issued using this entry contain the scope value
`messages.read` and are bound to a DPoP key. An administrator of the
Grant Issuer still approves a subset before the entry is used
({{narrowing}}).

~~~ json
{
  "authorization_server": "https://ras.example/",
  "grant_issuer": {
    "issuer": "https://idp.example/",
    "tenant": "org-acme"
  },
  "tenants": [
    {
      "tenant_id": "customer-acme",
      "grant_catalog": "read-only"
    }
  ],
  "grant_catalogs": [
    {
      "id": "read-only",
      "entries": [
        {
          "id": "messages-read",
          "revision": "1",
          "profile": "urn:ietf:params:oauth:grant-profile:id-jag",
          "resource": "https://api.example/messages",
          "scope": {
            "requirement": "required",
            "values": [{"scope": "messages.read"}]
          },
          "grant_binding": {
            "requirement": "required",
            "methods": ["dpop"]
          }
        }
      ]
    }
  ]
}
~~~
{: #fig-model-example title="Example Relationship View"}

{{fig-model}} shows the object structure, without the `display` and
`description` members. {{example-catalog}} provides a fuller example of
the inputs that an entry can govern.

~~~ ascii-art
Relationship View
 |-- authorization_server
 |-- grant_issuer
 |-- tenants[]: Tenant Object
 |    |-- tenant_id
 |    `-- grant_catalog        (the id of a Grant Catalog)
 `-- grant_catalogs[]: Grant Catalog
      |-- id
      `-- entries[]: entry
           |-- id, revision
           |-- profile, resource
           `-- scope, authorization_details, claims,
               sub_id, grant_binding
~~~
{: #fig-model title="Structure of a Relationship View"}

## Common Conventions {#model-conventions}

Every object defined in this section is a JSON object {{RFC8259}}. A
document that carries these objects MUST conform to I-JSON {{RFC7493}}.
A Grant Issuer MUST ignore members it does not understand
({{extensibility}}).

Each object is defined by a table of its members, in which "string",
"object", and "array" denote JSON types; the JSON value `null` has none
of these types. The following members are
used by several objects:

`id`:
: A stable identifier for the object. The value is a non-empty JSON
  string of at most 255 octets when encoded in UTF-8. Identifiers are
  opaque and case-sensitive, and are compared using simple string
  comparison ({{Section 6.2.1 of RFC3986}}). The length limit bounds the
  identifiers that a Grant Issuer retains with each approval
  ({{installation}}). Grant Catalog identifiers are unique within a
  Relationship View, and entry identifiers are unique within a Grant
  Catalog.

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
values, the array MUST NOT contain the same value twice. Within an
entry, each `scope` value within a `values` array, each `type` value
within a `types` array, and each `claim` value within a `claims` array
is unique and identifies an item. Members that carry protocol values,
such as `authorization_server`, `tenant_id`, `profile`, `resource`,
`scope`, `type`, `claim`, and the names in `formats` and `methods`, are
compared using simple string comparison ({{Section 6.2.1 of RFC3986}}).

## Relationship View {#relationship-view}

A Relationship View is a JSON object with the following members:

| Member | Status | Description |
| --- | --- | --- |
| `authorization_server` | REQUIRED | String; the issuer identifier of the Resource Authorization Server |
| `grant_issuer` | REQUIRED | Object; identifies the Issuer Relationship that the Relationship View serves |
| `tenants` | REQUIRED | Array containing every Tenant Object ({{tenant-object}}) eligible for the Issuer Relationship |
| `grant_catalogs` | REQUIRED | Array containing each Grant Catalog ({{grant-catalog}}) that a Tenant Object references, each exactly once |
{: title="Relationship View Members"}

The `issuer` member of `grant_issuer` is REQUIRED and contains the
issuer identifier of the Grant Issuer. The `tenant` member of
`grant_issuer` contains
the issuer tenant identifier, and is present if and only if the Issuer
Relationship has one ({{relationship-identity}}). The Resource
Authorization Server derives this object from the Issuer Relationship
that it used to select the Relationship View. Comparing it with the
Issuer Relationship for which the Grant Issuer retrieved the view
({{validation}}) detects errors in routing, caching, or the association
of responses with requests. The comparison does not show that the
Resource Authorization Server selected the content of the view
correctly.

A Relationship View is complete and is not paged. If the Issuer
Relationship has no eligible tenants, both arrays are empty.

## Tenant Object {#tenant-object}

A Tenant Object binds one Resource Authorization Server tenant that is
eligible for an Issuer Relationship to the Grant Catalog that applies to
it.

| Member | Status | Description |
| --- | --- | --- |
| `tenant_id` | OPTIONAL | String; the exact tenant identifier ({{conventions}}) of the Resource Authorization Server tenant, which a grant issued for this tenant conveys as its target tenant ({{envelope}}) |
| `display`, `description` | OPTIONAL | See {{model-conventions}} |
| `grant_catalog` | REQUIRED | String; the `id` of the Grant Catalog that applies to this tenant |
{: title="Tenant Object Members"}

A Tenant Object is identified by its `tenant_id`, or, for the one
Tenant Object that may omit it, by that absence. An ID-JAG conveys
`tenant_id` in the `aud_tenant` claim ({{jwt-representation}}). The
Resource Authorization Server MUST include `tenant_id` unless its issuer
identifier alone identifies the target tenant. A Relationship View MUST
NOT contain more than one Tenant Object without `tenant_id`, or two
Tenant Objects with the same value of `tenant_id`.

A Tenant Object that contains `tenant_id` is usable only with
entries whose Grant Profile can convey a target tenant ({{envelope}},
{{issuing}}).

Several Tenant Objects can reference the same Grant Catalog. Tenant
Objects for tenants that differ in any input that an entry governs
({{envelope}}) reference different Grant Catalogs. A Tenant Object has
no default Grant Catalog, and Grant Catalogs have no inheritance,
partial overrides, or merge rules.

{{installation}} specifies how a change to `grant_catalog` affects
approvals; a Tenant Object with a different `tenant_id` is a different
Tenant Object.

The following is an example of a Tenant Object:

~~~ json
{
  "tenant_id": "customer-acme",
  "display": "Acme",
  "grant_catalog": "read-only"
}
~~~

## Grant Catalog {#grant-catalog}

A Grant Catalog is a complete set of entries and is never paged or
assembled from parts.

| Member | Status | Description |
| --- | --- | --- |
| `id` | REQUIRED | Identifier ({{model-conventions}}) |
| `display`, `description` | OPTIONAL | See {{model-conventions}} |
| `entries` | REQUIRED | Array of entries ({{entry}}) |
{: title="Grant Catalog Members"}

An empty `entries` array indicates that the Resource Authorization
Server accepts no grants for the tenants whose Tenant Objects reference
this Grant Catalog. A Grant Catalog can contain entries for several
Grant Profiles.

## Entry {#entry}

An entry describes the grants of one Grant Profile that the Resource
Authorization Server accepts for one protected resource.

| Member | Status | Description |
| --- | --- | --- |
| `id` | REQUIRED | Identifier ({{model-conventions}}) |
| `revision` | REQUIRED | String of at most 255 octets in UTF-8; the version of the entry's normative semantics ({{revision}}) |
| `display`, `description` | OPTIONAL | See {{model-conventions}} |
| `profile` | REQUIRED | String; the identifier of the Grant Profile of the grants the entry describes |
| `resource` | REQUIRED | String; the resource indicator ({{Section 2 of RFC8707}}) of the protected resource |
| `scope` | OPTIONAL | Object; permitted scope values ({{entry-scope}}) |
| `authorization_details` | OPTIONAL | Object; permitted authorization details types ({{entry-authorization-details}}) |
| `claims` | OPTIONAL | Non-empty array of claim objects ({{entry-claims}}) |
| `sub_id` | OPTIONAL | Object; permitted subject identifier formats ({{entry-sub-id}}) |
| `grant_binding` | OPTIONAL | Object; permitted grant binding methods ({{entry-grant-binding}}) |
{: title="Entry Members"}

The value of `resource` is an absolute URI {{RFC3986}} and MUST NOT
contain a fragment component.

{{envelope}} specifies the grants that the `scope`,
`authorization_details`, `claims`, `sub_id`, and `grant_binding` members
permit, including when they are absent.

The `profile` member identifies the Grant Profile, not the grant type
used to redeem the grant, because several Grant Profiles share the JWT
bearer grant type `urn:ietf:params:oauth:grant-type:jwt-bearer`
{{RFC7523}}.

A Grant Catalog can contain entries for several resources, and several
entries can name the same resource when they describe different grants
({{envelope}}). A client that needs access to several resources
obtains a separate grant for each.

### Requirement Levels {#requirement}

The `scope`, `authorization_details`, `sub_id`, and `grant_binding`
members of an entry each contain a `requirement` member, called the
presence requirement, that states whether a grant conveys the input at
all. Scope value objects, authorization details type objects, and claim
objects each contain an OPTIONAL `requirement` member, called the item
requirement, that states whether a grant conveys that item. The value of
a `requirement` member is the JSON string `required` or `optional`. An
absent item requirement is treated as `optional`.

For a member with items A and B, the two levels permit the following
selections:

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
`required` means that a grant uses one of them ({{envelope}}).

### Scope {#entry-scope}

The `scope` member of an entry is a JSON object with the following
members:

| Member | Status | Description |
| --- | --- | --- |
| `requirement` | REQUIRED | The presence requirement ({{requirement}}) |
| `values` | REQUIRED | Non-empty array of scope value objects |
{: title="Members of the scope Member"}

A scope value object has the following members:

| Member | Status | Description |
| --- | --- | --- |
| `scope` | REQUIRED | String; one scope token ({{Section 3.3 of RFC6749}}) |
| `requirement` | OPTIONAL | The item requirement ({{requirement}}) |
| `display`, `description` | OPTIONAL | See {{model-conventions}} |
{: title="Scope Value Object Members"}

### Authorization Details Types {#entry-authorization-details}

The `authorization_details` member of an entry is a JSON object with the
following members:

| Member | Status | Description |
| --- | --- | --- |
| `requirement` | REQUIRED | The presence requirement ({{requirement}}) |
| `types` | REQUIRED | Non-empty array of authorization details type objects |
{: title="Members of the authorization_details Member"}

An authorization details type object has the following members:

| Member | Status | Description |
| --- | --- | --- |
| `type` | REQUIRED | String; an authorization details type ({{Section 2 of RFC9396}}) |
| `requirement` | OPTIONAL | The item requirement ({{requirement}}) |
| `documentation_uri` | OPTIONAL | String; an absolute `https` URI of human-readable documentation for the type |
| `display`, `description` | OPTIONAL | See {{model-conventions}} |
{: title="Authorization Details Type Object Members"}

An entry constrains the `type` values of the authorization details
objects that a grant conveys, not the other members of those objects. A
grant can convey more than one authorization details object of a
permitted type. The Grant Issuer applies its policy to complete
authorization details objects, and the Resource Authorization Server
validates their contents at redemption. The catalog does not carry
schemas for authorization details types ({{optional-metadata}}).

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
lists. A claim object has the following members:

| Member | Status | Description |
| --- | --- | --- |
| `claim` | REQUIRED | String; a claim name in the namespace that the Grant Profile Definition specifies, which for JWT authorization grants is a JWT claim name ({{Section 4 of RFC7519}}) |
| `requirement` | OPTIONAL | The item requirement ({{requirement}}) |
| `definition_uri` | OPTIONAL | String; an absolute `https` URI of the definition of the claim |
| `display`, `description` | OPTIONAL | See {{model-conventions}} |
{: title="Claim Object Members"}

The `claim` member of a claim object MUST NOT be a reserved claim name
of the entry's Grant Profile. Listing a claim does not change the
syntax or semantics that the Grant Profile or the claim's definition
specifies for it. Claim objects carry no constraints on claim values: an
entry that lists an authentication context claim as `required` requires
the claim to be present, not a particular value. The `definition_uri`
member is descriptive and does not establish the truth or provenance of
a claim.

### Subject Identifier {#entry-sub-id}

The `sub_id` member of an entry is a JSON object with the following
members:

| Member | Status | Description |
| --- | --- | --- |
| `requirement` | REQUIRED | The presence requirement ({{requirement}}) |
| `formats` | REQUIRED | Non-empty array of unique format names from the "Security Event Identifier Formats" registry ({{Section 8.1 of RFC9493}}), such as `email` or `saml-nameid` ({{Section 3.2.1 of ID-JAG}}) |
{: title="Members of the sub_id Member"}

This member describes a subject identifier {{RFC9493}} conveyed in
addition to the subject of the grant; for an ID-JAG, it is the `sub_id`
claim ({{jwt-representation}}).

### Grant Binding {#entry-grant-binding}

The `grant_binding` member of an entry is a JSON object with the
following members:

| Member | Status | Description |
| --- | --- | --- |
| `requirement` | REQUIRED | The presence requirement ({{requirement}}) |
| `methods` | REQUIRED | Non-empty array of unique grant binding method names, defined below or by other specifications ({{extensibility}}) |
{: title="Members of the grant_binding Member"}

This specification defines the following grant binding methods:

`dpop`:
: The grant is bound to a DPoP key {{RFC9449}}, and the client
  demonstrates possession of that key when presenting the grant.

`mtls`:
: The grant is bound to an X.509 certificate {{RFC8705}}, and the client
  presents that certificate when presenting the grant.

The catalog does not describe client authentication to the Resource
Authorization Server or sender-constraining of the access token it
issues; both are separate from grant binding.

## Authorization Envelope {#envelope}

An entry defines an Authorization Envelope: the set of grants that the
Resource Authorization Server is prepared to accept for one protected
resource from the Issuer Relationship. The inputs that an entry governs
are the protected resource and the inputs listed in {{tab-inputs}}.
Grant Issuer policy and the runtime request narrow the Authorization
Envelope ({{narrowing}}).

A grant is within the Authorization Envelope of an entry, for a Tenant
Object that references the Grant Catalog containing the entry, when all
of the following are true:

1. The grant conveys the `authorization_server` of the Relationship View
   that contains the Grant Catalog as its only audience.

2. If the Tenant Object contains `tenant_id`, the grant conveys that
   value as its target tenant. Otherwise, the grant conveys no target
   tenant.

3. The grant conveys the entry's `resource` as its only protected
   resource.

4. For each input in {{tab-inputs}}:

   * If the entry has no member for the input, the grant conveys none of
     that input.

   * Otherwise, every item of the input that the grant conveys is listed
     in the member, and the grant conveys every listed item whose item
     requirement is `required`.

   * If the member's presence requirement is `required`, the grant
     conveys at least one item. The `claims` member has no presence
     requirement.

   For grant binding, a bound grant uses exactly one method.

| Input (Member) | Item | Conveyed When the Grant |
| --- | --- | --- |
| Scope values (`scope`, {{entry-scope}}) | The `scope` of each scope value object | Conveys that scope value |
| Authorization details (`authorization_details`, {{entry-authorization-details}}) | The `type` of each authorization details type object | Conveys at least one authorization details object of that type |
| Catalog-Governed Claims (`claims`, {{entry-claims}}) | The `claim` of each claim object | Conveys that claim |
| Subject identifier in addition to the subject (`sub_id`, {{entry-sub-id}}) | Each name in `formats` | Conveys a subject identifier in that format |
| Grant binding (`grant_binding`, {{entry-grant-binding}}) | Each name in `methods` | Is bound using that method |
{: #tab-inputs title="Inputs Governed by Members of an Entry"}

The inputs are independent: any selection of items that one member
permits is permitted together with any selection that each other member
permits.

The catalog expresses no alternatives between sets of items, no
exclusions, and no conditions, either within a member or between
members. When the Resource Authorization Server accepts only some
combinations, such as one scope value or another but not both, or a
scope value only together with a particular claim, it publishes separate
entries whose Authorization Envelopes each contain only accepted
combinations ({{example-alternatives}}). A Grant Issuer does not infer a
rule that an entry does not state.

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

* any member that the specification defining it declares part of the
  normative semantics of an entry ({{extensibility}}).

The Resource Authorization Server MUST change the `revision` of an entry
whenever it changes the normative semantics of the entry, and MUST NOT
use, in Relationship Views for the same Issuer Relationship, a
`revision` value it has previously used for that entry with different
normative semantics. A counter that increases with each change, or a
digest of the normative semantics, meets both requirements. When
normative semantics are compared, the order of elements in the `values`,
`types`, `claims`, `formats`, and `methods` arrays is not significant,
an absent item requirement is equivalent to the value `optional`, and a
Grant Issuer compares only the members it understands. A change limited
to other members, such as `display`, `description`, `documentation_uri`,
or `definition_uri`, does not require a new revision.

The combination of the Resource Authorization Server issuer identifier,
the Issuer Relationship, the Grant Catalog `id`, the entry `id`, and the
`revision` identifies one set of normative semantics ({{validation}}).
Grant Catalog identifiers are unique only within a Relationship View
({{model-conventions}}), so the Grant Catalogs offered to two Issuer
Relationships can share an identifier and differ in content.

The `revision` member is opaque to the Grant Issuer and provides no
integrity guarantee. It
lets the Grant Issuer determine whether an approval made for an earlier
copy of an entry still applies ({{installation}}). The retrieval
protocol handles caching of representations.

# Using a Catalog {#processing}

## Validating a Relationship View {#validation}

Before installing any part of a Relationship View, the Grant Issuer MUST
verify the following:

1. The `authorization_server` of the Relationship View, and any other
   issuer identifier that the retrieval protocol conveys, are identical
   to the issuer identifier of the Resource Authorization
   Server with which the Grant Issuer associates the Issuer
   Relationship, using simple string comparison
   ({{Section 6.2.1 of RFC3986}}).

2. The `grant_issuer` of the Relationship View identifies the
   Issuer Relationship for which the Grant Issuer retrieved it: the
   `issuer` of `grant_issuer` is identical to that relationship's
   issuer identifier, and the `tenant` of `grant_issuer` is
   identical to that
   relationship's issuer tenant identifier, or both are absent.

3. Every document conforms to I-JSON {{RFC7493}}. The Relationship View,
   the structures that the retrieval protocol defines, every Tenant
   Object, and every Grant Catalog other than the contents of its
   entries satisfy their definitions: every REQUIRED member is present,
   and every member defined by this specification has the specified JSON
   type and meets the constraints stated for it. Grant Catalog and
   entry identifiers satisfy {{model-conventions}}.

4. The `grant_catalog` of every Tenant Object identifies a Grant Catalog
   in the Relationship View, and every Grant Catalog in the Relationship
   View is referenced by at least one Tenant Object.

5. At most one Tenant Object omits `tenant_id`, and no two Tenant
   Objects have the same `tenant_id`.

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
   {{model-conventions}}: a REQUIRED member is absent, a member has a
   JSON type other than the one specified, or a constraint stated for
   the member is not met.

3. The entry contains a member for an input ({{envelope}}) that its
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

Public metadata remains authoritative for what it describes, and an
entry cannot enable a value that public metadata excludes. When it
installs a Relationship View, the Grant Issuer composes each entry with
public metadata as follows:

1. Retrieve and validate the Resource Authorization Server's
   Authorization Server Metadata ({{Section 3.3 of RFC8414}}), and
   verify that its issuer identifier is identical to the
   `authorization_server` of the Relationship View.

2. If the entry's `resource` is a URL that uses the `https` scheme
   ({{Section 1.2 of RFC9728}}), request its Protected Resource Metadata
   using the `oauth-protected-resource` well-known URI suffix
   ({{Section 3 of RFC9728}}); otherwise, this step has no effect. If
   the response has HTTP status code 200 (OK), validate the metadata as
   specified in {{Section 3.3 of RFC9728}}. Any other response, or no
   response, leaves this step without effect, because publishing
   Protected Resource Metadata is optional.

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
Profile or for an input that an entry governs ({{envelope}}) can
specify that the member is
checked in the same way. A member from which the
defining specification permits a server to omit supported values is not
checked: the scope values of an entry need not appear in
`scopes_supported` ({{Section 2 of RFC8414}}, {{Section 2 of RFC9728}}),
and an `authorization_servers` member of Protected Resource Metadata
that omits the Resource Authorization Server's issuer identifier is not
a conflict ({{Section 2 of RFC9728}}).

If a step fails, including when Protected Resource Metadata is retrieved
and fails validation, the entry conflicts with public metadata and the
Grant Issuer MUST NOT use it for issuance. The Grant Issuer does not
resolve a conflict by preferring catalog values over public metadata or
by combining values from both.

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
binding is required and `dpop` is the only method. How a Grant Issuer
records approval, and whether it approves entries for particular
clients or subjects, is a matter of Grant Issuer policy.

The rules in {{issuing}} that refer to what an entry requires or
permits apply to the approved subset.

At runtime, the Grant Issuer applies subject, client, consent, and other
local policy, and the client's request, to narrow the approved subset
further. If local policy excludes an item that the approved subset
requires, the Grant Issuer MUST NOT issue a grant using that entry. If
the client's request omits such an item, the Grant Issuer either adds
it, when its policy permits issuing values that the client did not
request, or does not issue a grant using
that entry.

For example:

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
the catalog permits it.

## Issuing a Grant {#issuing}

When a Grant Issuer issues a grant using an entry, the Grant Issuer MUST
ensure that the grant is within the Authorization Envelope of the subset
of that entry approved for the Tenant Object for which the grant is
issued ({{narrowing}}). The Grant
Issuer issues each grant using exactly one entry, and MUST NOT combine
inputs permitted by different entries in one grant.

After authenticating the client and the subject and applying its policy,
the Grant Issuer issues a grant as specified by the Grant Profile, using
one approved Tenant Object and one approved entry. The Grant Issuer
selects the Tenant Object from the client's request and its own
configuration for the Issuer Relationship, and selects an entry whose
Grant Profile matches the kind of grant the client requested and whose
`resource` is the requested protected resource. The catalog defines no
parameter of the client's grant request for selecting a target tenant
({{profile-definitions}}). When more than one approved Tenant Object or
entry could satisfy a request, Grant Issuer policy selects one. The
Grant Issuer issues a grant for a request that names several resources
only if its policy reduces the request to one of them, and for a request
that names none only if its policy determines one ({{envelope}}).

The Grant Issuer MUST NOT issue a grant using an entry obtained for an
Issuer Relationship unless the grant identifies its issuer by the issuer
identifier of that relationship and, when the relationship has an issuer
tenant, identifies that issuer tenant, as the Grant Profile Definition
specifies ({{profile-definitions}}).

The Grant Issuer MUST NOT derive a target tenant identifier from the
number of Tenant Objects in a Relationship View or from the form of an
issuer identifier.

The Grant Issuer decides whether it releases a claim value. If an entry
requires a claim that the Grant Issuer's policy does not permit it to
release for a subject, the Grant Issuer MUST NOT issue a grant using
that entry for that subject.

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

The Grant Issuer uses an entry with a Tenant Object only when all of the
following hold:

1. The Relationship View that contains them is installed
   ({{installation}}).

2. The entry is usable for issuance ({{validation}}) and does not
   conflict with public metadata ({{composition}}).

3. If the Tenant Object contains `tenant_id`, the entry's Grant
   Profile can convey a target tenant ({{tenant-object}}).

4. An approval for the entry and the Tenant Object applies
   ({{installation}}).

When the Grant Issuer does not issue a grant because no approved entry
permits the request, it responds to the client as the Grant Profile
Definition specifies.

### Clients {#clients}

The catalog does not enumerate or register application clients, or
establish client identifier mappings. This document does not define
views specific to one application client. A Relationship View is
relationship-wide and does not establish eligibility for a particular
application client: a Resource Authorization Server can permit a grant
configuration for one application client and reject an otherwise
equivalent configuration for another. The Grant Profile Definition of a
Grant Profile whose grants identify the client by its client identifier
at the Resource Authorization Server specifies how the Grant Issuer
obtains that identifier, as {{id-jag}} does for ID-JAG. Such an
identifier can be shared across trust domains {{CIMD}}.

## Installation and Reconciliation {#installation}

A retrieval is complete when the Grant Issuer obtains and validates a
complete Relationship View ({{validation}}), or when the retrieval
protocol confirms the installed one ({{refresh}}). It is incomplete when
a transport failure occurs, a response is interrupted or fails
validation, or the Resource Authorization Server rejects the Grant
Issuer's credentials. After a complete retrieval, the Grant Issuer
replaces the installed state for the Issuer Relationship with the new
Relationship View. A Tenant Object, Grant Catalog, or entry that the new
Relationship View omits, or contains with a different `grant_catalog` or
`revision`, is withdrawn.

A Grant Issuer retains, with each approval, the Resource Authorization
Server issuer identifier, the Issuer Relationship, the Tenant Object's
`tenant_id`, if any, the Grant Catalog `id`, the entry `id` and
`revision`, the approved subset, and any client identifier mapping that
the Grant Profile requires. The items of the approved subset are
identified by their protocol values. State that is specific to a
retrieval protocol, such as a URI or an HTTP entity tag, is retained
separately and is not part of an approval.

An approval applies while the installed Relationship View contains a
Tenant Object with the approved `tenant_id`, or without one if the
approved Tenant Object had none, and with the approved
`grant_catalog`, and the Grant Catalog that the Tenant Object references
contains an entry with the approved `id` and `revision`. The
Grant Issuer applies the rules in {{tab-transitions}} when a retrieval
completes or fails.

| Event | Rule |
| --- | --- |
| A retrieval is incomplete | The Grant Issuer MUST NOT install any state from that retrieval, and does not treat it as a withdrawal. State installed from an earlier retrieval remains in force, subject to the maximum age below. |
| A Tenant Object or entry appears | The appearance of a new Tenant Object or entry MUST NOT, by itself, cause the Grant Issuer to issue grants using it. |
| An entry changes | A change to an entry MUST NOT, by itself, expand an existing approval. |
| Only members outside the normative semantics of an entry change ({{revision}}), and every approved identifier, `grant_catalog`, and `revision` is unchanged | Approvals that depend on the changed objects continue to apply. |
| A Tenant Object, Grant Catalog, or entry is withdrawn | The Grant Issuer MUST stop issuing grants that depend on it in its previous form, and suspends issuance under the approvals that depend on it. |
| The `grant_catalog` of a Tenant Object, or the `revision` of an entry, changes | The Grant Issuer MUST NOT issue grants under an approval made for the previous value. Issuance resumes only under an approval made for the new value. |
| A later complete retrieval contains a withdrawn object again with the approved identifiers, `grant_catalog`, and `revision` | The approval applies again without new approval. |
{: #tab-transitions title="Effects of a Retrieval on Installed State and Approvals"}

A change of `grant_catalog` or `revision` also
withdraws the previous form of the object, so the withdrawal row applies
to it as well.

These rules apply to every change of revision, including a change that
only narrows an entry, such as the removal of a scope value
({{sec-revision}}). Whether to relax these rules is an open issue
({{open-issues}}).

The Grant Issuer SHOULD retrieve the Relationship View periodically and
after administrative changes to the Issuer Relationship. The Grant
Issuer SHOULD stop issuing grants that depend on installed state it has
not been able to refresh within a maximum age that it configures for the
Issuer Relationship, such as one day. The maximum age is measured from
the most recent complete retrieval; an incomplete retrieval, including
one that results from the end of the Issuer Relationship, does not
extend it ({{sec-stale}}). A Resource Authorization Server that needs
issuance to stop sooner first removes every eligible tenant from the
relationship; the Grant Issuer observes that removal at its next refresh
as a withdrawal.

## Optional Supporting Metadata {#optional-metadata}

A Grant Issuer can use other metadata to help administrators understand
entries: authorization details types metadata {{RAR-METADATA}} describes
the structure of an authorization details type, and authority metadata
{{AUTHORITY-METADATA}} describes what a scope value or authorization
details type authorizes. Such metadata is advisory. It cannot broaden an
entry or an approval, it does not cause an entry to conflict with public
metadata, and its absence, or a failure to retrieve it, does not affect
the validity of an entry. How a Grant Issuer uses schemas or authority
descriptions to enforce or narrow policy is not specified in this
document.

The Grant Issuer MUST NOT send a credential obtained through a retrieval
protocol when dereferencing a `schema_uri`, `documentation_uri`, or
`definition_uri`.

# Retrieving a Catalog {#oauth-binding}

Before disclosing catalog content to a Grant Issuer, the Resource
Authorization Server establishes, through a retrieval protocol
({{extensibility}}), which Issuer Relationship the disclosure serves,
and verifies that the relationship is authorized and current.
{{relationship-identity}} and {{relationship-authorization}} apply to
every retrieval protocol. The remaining subsections define the OAuth
retrieval protocol, which requires only public metadata and an
Onboarding Arrangement, and is the same for every Grant Profile.

## Relationship Identity {#relationship-identity}

An Issuer Relationship is identified by the Grant Issuer's issuer
identifier {{RFC8414}} and, when that issuer is multi-tenant, an issuer
tenant identifier, which is a tenant identifier ({{conventions}}) of the
Grant Issuer. Grants issued by the Grant Issuer identify their issuer
and issuer tenant by these values. Where a
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

An Onboarding Arrangement ({{conventions}}) records whether the
Grant Issuer uses a multi-tenant issuer, which Resource Authorization
Server tenants are eligible for the Issuer Relationship, the keys and
algorithms the Grant Issuer uses, and, for the OAuth retrieval protocol,
whether client authentication or DPoP-bound catalog access assertions
are required.

The Resource
Authorization Server MUST NOT disclose catalog content to an Issuer
Relationship that no Onboarding Arrangement authorizes, and MUST
determine the Relationship View solely from the authenticated Issuer
Relationship.

The Resource Authorization Server MUST verify that the Issuer
Relationship is authorized each time it discloses catalog content.
Ending an Issuer Relationship stops disclosure, even under a
credential issued before the relationship ended.

## Authorization Server Metadata {#as-metadata}

This specification defines the following Authorization Server Metadata
{{RFC8414}} member for the OAuth retrieval protocol:

`authorization_grant_catalog_endpoint`:
: URL of the catalog endpoint, from which the Grant Issuer retrieves its
  Relationship View ({{view-document}}). The URL uses the `https` scheme
  and contains no fragment component. The URL also identifies a
  protected resource, called the catalog resource, and is its resource
  indicator ({{Section 2 of RFC8707}}).

A Resource Authorization Server that supports the OAuth retrieval
protocol MUST publish this member. A Grant Issuer MUST NOT use the
OAuth retrieval protocol with a Resource Authorization Server whose
metadata omits this member.

The following is an example of Authorization Server Metadata for a
Resource Authorization Server that supports ID-JAG and the OAuth
retrieval protocol:

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

A catalog access assertion is a JWT that the Grant Issuer signs on its
own behalf, not on behalf of an end user or a client, and presents as an
authorization grant {{RFC7523}} to obtain a catalog access token. It is
not a grant in the sense of this specification ({{conventions}}). It is
a JWS {{RFC7515}} with the following JOSE header parameters:

`typ`:
: REQUIRED. The value `oauth-grant-catalog-access+jwt`
  ({{Section 3.11 of RFC8725}}).

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
  `sub` claim; for this assertion, the claim has no other meaning.

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
: OPTIONAL. A confirmation claim that binds the assertion to a DPoP key
  that the Grant Issuer controls, as specified in
  {{Section 3 of JWT-DPOP}}. A catalog access assertion that contains
  this claim is DPoP-bound. The Grant Issuer MUST NOT include this claim
  unless the `grant_types_supported` member of the Resource
  Authorization Server's metadata includes
  `urn:ietf:params:oauth:grant-type:jwt-dpop`.

The catalog access assertion identifies only the Issuer Relationship
and, when it is DPoP-bound, the Grant Issuer's DPoP key. It does not
contain the `aud_tenant` claim or identify a Grant Profile, target
tenant, End-User, client, resource, scope, or authorization details.
Other claims MAY be present and are ignored unless understood.

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

5. The `iat` claim is present and, subject to the Resource Authorization
   Server's allowance for clock skew, is not later than the current
   time, and `exp` is not more than 300 seconds after `iat`.

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
any Grant Profile or as a client assertion. These rules apply whether or
not a Grant Profile specifies a `typ` value for its grants
({{sec-cross-jwt}}).

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
: REQUIRED. The value `authorization_grant_catalog.read`, the scope
  value this specification defines for read access to the catalog.

A Resource Authorization Server that supports the OAuth
retrieval protocol and includes
`urn:ietf:params:oauth:grant-type:jwt-dpop` in `grant_types_supported`
MUST accept DPoP-bound catalog access assertions.

When the Onboarding Arrangement requires client authentication at the
token endpoint ({{Section 3.1 of RFC7523}}), the Grant Issuer is a
registered client of the Resource Authorization Server and authenticates
using the method that the Onboarding Arrangement fixes, such as a JWT
client assertion signed with a key of the Grant Issuer
({{Section 2.2 of RFC7523}}). A Grant Issuer
that authenticates with a JWT client assertion MUST use the issuer
identifier of the Resource Authorization Server as the sole value of the
assertion's `aud` claim, and the Resource Authorization Server MUST
reject a catalog access token request whose client assertion does not
contain its issuer identifier as the sole audience value
({{RFC7523BIS}}). The Grant Issuer
MUST NOT use a catalog access assertion as a client assertion, and
SHOULD sign client assertions with a key that it does not use to sign
catalog access assertions or grants ({{Section 3.12 of RFC8725}}). When
client authentication is not required, the request contains no
`client_id` parameter.

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
&client_assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6ImNsaWVudC1rZXktMSJ9...
~~~

The Resource Authorization Server processes the request as follows:

1. If the grant type is `urn:ietf:params:oauth:grant-type:jwt-dpop`, it
   validates the DPoP proof ({{Section 5 of JWT-DPOP}}) before
   validating the assertion, so that a request rejected for its proof or
   nonce does not consume the assertion's `jti`
   ({{Section 5.1 of JWT-DPOP}}).

2. It validates the catalog access assertion
   ({{assertion-validation}}).

3. If the grant type is `urn:ietf:params:oauth:grant-type:jwt-dpop`, it
   verifies that the assertion is bound to the key of the DPoP proof
   ({{Section 5 of JWT-DPOP}}) and that its `cnf` claim contains only
   the `jkt` member. If the grant type is
   `urn:ietf:params:oauth:grant-type:jwt-bearer`, it verifies that the
   assertion contains no `cnf` claim, whether or not it supports the
   `urn:ietf:params:oauth:grant-type:jwt-dpop` grant type
   ({{Section 7.1 of JWT-DPOP}}), and that the Onboarding Arrangement
   does not require DPoP-bound catalog access assertions.

4. It verifies that the `resource` parameter is identical to its
   `authorization_grant_catalog_endpoint` metadata value.

5. It verifies that the `scope` parameter is
   `authorization_grant_catalog.read`.

6. If the Onboarding Arrangement requires client authentication, it
   authenticates the client and verifies that the Onboarding
   Arrangement associates the authenticated client with the `iss` claim
   of the catalog access assertion. It does not accept as a client
      assertion any JWT whose `typ` is `oauth-grant-catalog-access+jwt`.

7. It records the assertion's `jti` value as accepted and issues the
   catalog access token. It checks that the value is still unused and
   records it in one atomic operation, so that two concurrent requests
   that carry the same assertion cannot both obtain a catalog access
   token, and a request rejected by an earlier step does not consume
   the value.

When the Onboarding Arrangement requires client authentication, the
Resource Authorization Server MUST NOT issue a catalog access token on
the strength of the catalog access assertion alone.

Because the catalog is disclosed to the Issuer Relationship identified
by the catalog access assertion, not to the authenticated client, a
Grant Issuer that uses a multi-tenant issuer needs at most one client
registration at each Resource Authorization Server, and its client
identifier can equal its issuer identifier, as in the example above.

## Catalog Access Token Response {#token-response}

If the request is valid, the Resource Authorization Server responds as
specified in {{Section 5.1 of RFC6749}}. The Resource Authorization
Server MUST restrict the catalog access token to the catalog resource
and the `authorization_grant_catalog.read` scope, and MUST associate the
token with the Issuer Relationship identified by the catalog access
assertion. The Resource Authorization Server SHOULD issue catalog access
tokens with a lifetime of a few minutes, and MUST NOT issue a refresh
token; the Grant Issuer obtains a new catalog access token with a new
catalog access assertion. The Resource Authorization Server
MUST NOT issue an access token for the catalog resource, or with the
`authorization_grant_catalog.read` scope, in response to any token
request other than one specified in {{token-request}}.

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
for a DPoP-bound token, {{Section 7.1 of RFC9449}}. The Resource
Authorization Server verifies the DPoP proof for a DPoP-bound token and
rejects a DPoP-bound token presented as a bearer token
({{Section 7.2 of RFC9449}}). The Grant Issuer
MUST send a catalog access token only to the
`authorization_grant_catalog_endpoint` URL from Authorization Server
Metadata it has validated ({{Section 3.3 of RFC8414}}), without further
resolution or normalization of that URL. It MUST NOT follow redirections
in response to a request that carries a catalog access token, and treats
a redirection as an incomplete retrieval ({{installation}}).

The Resource Authorization Server determines the Relationship View from
the Issuer Relationship associated with the catalog access token
({{relationship-authorization}}) and responds with HTTP status code 200
(OK) and a Relationship View document of media type
`application/oauth-grant-catalog+json` ({{iana-media-types}}), whose
content is the Relationship View ({{relationship-view}}). The Resource
Authorization Server MUST construct the document from one consistent
state of the Issuer Relationship. The Grant Issuer validates the
document as specified in {{validation}}, and treats a response of any
other media type as an incomplete retrieval ({{installation}}).

The following is an example of a Relationship View document request and
response for an Issuer Relationship with two eligible tenants, in which
the `entries` of each Grant Catalog are abbreviated; {{example-catalog}}
shows the Grant Catalog `standard` in full:

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
  "authorization_server": "https://ras.example/",
  "grant_issuer": {
    "issuer": "https://idp.example/",
    "tenant": "acme-workforce"
  },
  "tenants": [
    {
      "tenant_id": "customer-acme",
      "display": "Acme",
      "grant_catalog": "read-only"
    },
    {
      "tenant_id": "customer-acme-eu",
      "display": "Acme Europe",
      "grant_catalog": "standard"
    }
  ],
  "grant_catalogs": [
    {
      "id": "read-only",
      "entries": [ ... ]
    },
    {
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
({{Section 13.1.2 of RFC9110}}). When the Grant Issuer holds an entity
tag, it sends a conditional request, and sends `If-None-Match` only with
the entity tag of a representation it has installed. A 304 (Not
Modified) response ({{Section 15.4.5 of RFC9110}}) confirms the
installed Relationship View and is a complete retrieval
({{installation}}).

A Relationship View document is specific to an Issuer Relationship. The
Resource Authorization Server MUST include `Cache-Control: no-store`
({{Section 5.2.2.5 of RFC9111}}) in these responses. The Grant Issuer
retains the document and its entity tag as application state, keyed by
the Resource Authorization Server's issuer identifier and the Issuer
Relationship.

This retrieval protocol is designed for Relationship Views of up to
several thousand Tenant Objects in one response, not for unbounded
collections. The Grant Issuer SHOULD indicate support for content
codings using `Accept-Encoding` ({{Section 12.5.3 of RFC9110}}), and the
Resource Authorization Server SHOULD compress responses when the Grant
Issuer supports a content coding.

## Error Responses {#errors}

The token endpoint returns errors as specified in
{{Section 5.2 of RFC6749}} and, for the
`urn:ietf:params:oauth:grant-type:jwt-dpop` grant type,
{{Section 5 of JWT-DPOP}}. The Resource Authorization Server returns:

* `invalid_grant`, if the catalog access assertion is invalid, expired,
  or replayed, or identifies no authorized Issuer Relationship; if a
  DPoP-bound assertion is presented with the
  `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type; if an
  assertion that is not DPoP-bound is presented when the Onboarding
  Arrangement requires one; or if the client that the Grant Issuer
  authenticated as is not associated with the assertion's issuer.

* `invalid_target` ({{Section 2 of RFC8707}}), if the `resource`
  parameter is missing or is not the catalog resource.

* `invalid_scope`, if the `scope` parameter is missing or is not
  `authorization_grant_catalog.read`.

The catalog endpoint returns errors as specified in
{{Section 3 of RFC6750}}, or {{Section 7.1 of RFC9449}} for a DPoP-bound
token. The Resource Authorization Server responds with HTTP status code
401 (Unauthorized) and `invalid_token` if the Issuer Relationship
associated with the catalog access token is no longer authorized, and
with HTTP status code 403 (Forbidden) and `insufficient_scope` if the
access token does not include the `authorization_grant_catalog.read`
scope ({{Section 3.1 of RFC6750}}).

An error response MUST NOT disclose whether another issuer, issuer
tenant, Resource Authorization Server tenant, Grant Catalog, or entry
exists.

# Identity Assertion JWT Authorization Grant Profile {#id-jag}

In this Grant Profile Definition for {{ID-JAG}}, the Grant Issuer is
the IdP Authorization Server and grants are ID-JAGs. A Grant Profile
Definition for another profile of the JWT authorization grant
{{RFC7523}} can reuse the representation of inputs in
{{jwt-representation}}.

Grant Profile identifier:
: `urn:ietf:params:oauth:grant-profile:id-jag`
  ({{Section 7.2 of ID-JAG}}).

Grant binding:
: The IdP Authorization Server selects the binding as described in
  {{Section 9.11 of ID-JAG}}. When the client presents a valid DPoP
  proof, ID-JAG requires the IdP Authorization Server to bind the ID-JAG
  to the DPoP key or a client certificate
  ({{Section 9.11.1.1 of ID-JAG}}); either method
  satisfies that requirement if the approved subset permits it
  ({{issuing}}). When the approved subset permits neither, the IdP
  Authorization Server does not issue an ID-JAG using that entry for
  such a request ({{issuing}}), so a Resource Authorization Server that
  verifies DPoP-bound ID-JAGs lists `dpop` in its entries.

Resource selection:
: ID-JAG does not require the `resource` parameter of the Token Exchange
  request ({{Section 4.3 of ID-JAG}}); {{issuing}} specifies how the IdP
  Authorization Server selects an entry when that parameter is absent.

Tenant selection:
: The IdP Authorization Server selects the Tenant Object as it
  determines the `aud_tenant` claim ({{Section 6.4 of ID-JAG}}).

Error responses:
: When the IdP Authorization Server does not issue an ID-JAG because no
  approved entry permits the request, it responds to the Token Exchange
  request with the `access_denied` error code
  ({{Section 4.3.4.3 of ID-JAG}}).

Narrowed rules:
: An ID-JAG issued using an entry always contains the `resource` claim
  with exactly one resource identifier; {{Section 3.1 of ID-JAG}} makes
  the claim OPTIONAL and permits several resource identifiers. An
  ID-JAG issued using an entry contains a Catalog-Governed Claim only if
  the entry lists it ({{entry-claims}}); {{Section 3.1 of ID-JAG}}
  allows any claim valid in an ID Token and recommends the `email` and
  `aud_sub` claims for subject resolution. A Resource Authorization
  Server that uses those claims lists them in its entries. Requiring
  `tenant_id` in a Tenant Object ({{tenant-object}}) is stricter
  than {{Section 4.4.1 of ID-JAG}}, which also lets a Resource
  Authorization Server identify the target tenant from its trust
  relationship with the issuer.

Clients:
: The IdP Authorization Server obtains the value of the `client_id`
  claim, the client's identifier at the Resource Authorization Server,
  as described in {{Section 5.1 of ID-JAG}}; the catalog does not
  provide it ({{clients}}).

The following is an example of the claims of an ID-JAG issued for the
Tenant Object for `customer-acme-eu` in {{view-document}}, using the
entry in
{{example-catalog}}:

~~~ json
{
  "iss": "https://idp.example/",
  "sub": "user-123",
  "tenant": "acme-workforce",
  "aud": "https://ras.example/",
  "aud_tenant": "customer-acme-eu",
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

## Representation of Inputs {#jwt-representation}

An ID-JAG conveys its issuer, its issuer tenant ({{issuing}}), and the
values that {{envelope}} constrains as follows.

| Input | JWT Representation |
| --- | --- |
| Issuer | `iss` claim ({{Section 4.1.1 of RFC7519}}) |
| Issuer tenant | `tenant` claim {{OpenID.Enterprise}} ({{Section 6.1 of ID-JAG}}) |
| Audience | `aud` claim, as a string or an array of one string |
| Target tenant | `aud_tenant` claim ({{Section 3.1 of ID-JAG}}) |
| Protected resource | `resource` claim ({{Section 3.1 of ID-JAG}}), as a string or an array of one string |
| Scope values | `scope` claim ({{Section 4.2 of RFC8693}}) |
| Authorization details | `authorization_details` claim ({{Section 9.1 of RFC9396}}) |
| Catalog-Governed Claims | Claims named by the `claim` member of claim objects |
| Subject identifier | `sub_id` claim ({{Section 4.1 of RFC9493}}) |
| Grant binding | `cnf` claim ({{Section 3.1 of RFC7800}}) |
{: title="Representation of Inputs in an ID-JAG"}

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

The reserved claim names of this Grant Profile are `iss`, `sub`, `aud`,
`jti`, `exp`, `iat`, `nbf`, `tenant`, `aud_tenant`, `resource`, `scope`,
`authorization_details`, `sub_id`, `cnf`, `client_id`
({{Section 4.3 of RFC8693}}), and `act` ({{Section 4.1 of RFC8693}}).
The Grant Profile, not an entry, determines whether a grant conveys a
client identifier or an actor. The other claims that
{{Section 3.1 of ID-JAG}} defines (`auth_time`, `acr`, `amr`,
`aud_sub`, and `email`) are Catalog-Governed Claims. An ID-JAG has the
`typ` value `oauth-id-jag+jwt` ({{Section 3.1 of ID-JAG}},
{{sec-cross-jwt}}).

## Composition Checks {#jwt-composition}

A Resource Authorization Server that publishes entries for this Grant
Profile MUST include `grant_types_supported` in its Authorization Server
Metadata; if it lists this Grant Profile in
`authorization_grant_profiles_supported`, it also includes
`urn:ietf:params:oauth:grant-type:jwt-bearer` in `grant_types_supported`
({{Section 7.2 of ID-JAG}}). For each such entry, the Grant Issuer
performs the following checks in step 4 of {{composition}}, interpreting
`grant_types_supported` with the default value defined in
{{Section 2 of RFC8414}}:

* Unless the entry requires binding and lists `dpop` as its only method,
  `grant_types_supported` includes
  `urn:ietf:params:oauth:grant-type:jwt-bearer`.

* If the entry lists `dpop`, `grant_types_supported` includes
  `urn:ietf:params:oauth:grant-type:jwt-dpop`
  ({{Section 4.1 of JWT-DPOP}}).

# Extensibility {#extensibility}

The extension points below are designed to preserve the principles in
{{overview}}.

Grant Profiles:
: A specification can apply the catalog to another Grant Profile by
  providing a Grant Profile Definition ({{profile-definitions}}). A
  Grant Issuer that does not implement that Grant Profile Definition
  does not use that profile's entries ({{validation}}).

Members:
: A specification can define additional members of the objects defined
  in this specification. A Grant Issuer ignores members it does not
  understand ({{model-conventions}}). A specification that defines a
  member of an entry, or of an object within an entry, whose value
  affects what an administrator approves or which grants the entry
  permits declares that member part of the entry's normative semantics
  ({{revision}}).
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
: A specification can define additional grant binding methods. It
  states how the Grant Issuer binds a grant, how the Resource
  Authorization Server verifies the binding, and how a grant represents
  it.

Retrieval protocols:
: A specification can define another retrieval protocol. It defines
  discovery and error responses, authenticates the Issuer Relationship
  ({{relationship-identity}}) to the Resource Authorization Server and
  that server to the Grant Issuer, and delivers, unchanged, a complete
  Relationship View ({{relationship-view}}) that reflects one
  consistent state of the Issuer Relationship. A Grant Issuer does not
  combine objects obtained through different retrieval protocols.

## Grant Profile Definitions {#profile-definitions}

A Grant Profile Definition specifies the following:

1. The Grant Profile identifier used in the `profile` member of an
   entry.

2. How a grant of the profile conveys each input in {{envelope}}. A
   Grant Profile to which the catalog applies conveys the audience and
   the protected resource.
   Where the profile cannot convey another input, the Grant Profile
   Definition says so; an entry for that profile then does not contain
   the corresponding member.

3. How a grant of the profile identifies its issuer and, for a
   multi-tenant issuer, its issuer tenant ({{issuing}}).

4. The reserved claim names of the profile, including the claims the
   profile requires in every grant ({{entry-claims}}), and the namespace
   of the `claim` member.

5. For each grant binding method the profile supports, how the binding
   is represented in a grant, when the profile requires the Grant Issuer
   to bind a grant, and how the Grant Issuer selects among methods
   ({{issuing}}).

6. The composition checks against public metadata that apply to the
   profile, such as checks of the grant types used to redeem its grants
   ({{composition}}).

7. How the Grant Issuer selects a Tenant Object for a request, and how
   it responds when it does not issue a grant because no approved entry
   permits the request ({{issuing}}).

8. Any rule of the Grant Profile that catalog use narrows, such as
   making an OPTIONAL element required.

9. For a profile whose grants identify the client by its client
   identifier at the Resource Authorization Server, how the Grant
   Issuer obtains that identifier ({{clients}}).

A Grant Profile Definition for a JWT authorization grant SHOULD specify
a `typ` value for the grants of the profile
({{Section 3.11 of RFC8725}}), so that a Resource Authorization Server
that accepts them at the same token endpoint as catalog access
assertions can distinguish the two ({{sec-cross-jwt}}).

# Security Considerations {#security}

The security considerations of {{RFC7521}}, {{RFC7523}}, {{RFC8725}},
and {{RFC9700}}, and of the Grant Profile of each entry, such as
{{ID-JAG}}, apply to this specification.

## Catalog Is Not an Authorization Decision {#sec-not-authorization}

A catalog neither authorizes a request ({{introduction}}) nor lists
every grant that the Resource Authorization Server accepts: the Resource
Authorization Server can accept grants issued under other arrangements,
or under an entry that it has since withdrawn, as its policy determines.

The Authorization Envelope protects the Resource Authorization Server
only as far as the Grant Issuer observes it, so a Resource Authorization
Server that requires a property of every grant it accepts enforces that
property at redemption.

## Changes in Scope Meaning {#sec-scope-meaning}

An entry does not state the authority that its scope values and
authorization details types represent at the protected resource, so an
approval bounds the grants that the Grant Issuer constructs, not what
those grants authorize. For example, if the protected resource changes
`messages.write` so that it also permits deleting messages, the entry
and its revision do not change, and approvals that include
`messages.write` continue to apply. A Resource Authorization Server that
changes the authority of an existing scope value can instead publish a
new scope value, so that the change requires new approval. Authority
metadata can reveal such a change ({{optional-metadata}}) but does not
bind the protected resource.

## Relationship Authorization and Assertion Trust {#sec-relationship}

Trust in a Grant Issuer's signing keys establishes that the Grant Issuer
can issue grants, not that it is entitled to learn the Resource
Authorization Server's configuration for every issuer tenant identifier
it can name. An Onboarding Arrangement therefore authorizes each Issuer
Relationship separately ({{relationship-authorization}}). When a
Resource Authorization Server accepts one signing key for several Issuer
Relationships, as it can for a multi-tenant issuer, a compromise of that
key or of the signing service that uses it discloses the catalogs of
each of those Issuer Relationships. This specification does not require
any particular arrangement of keys across tenants. An Onboarding
Arrangement authorizes an issuer tenant identifier, not the organization
behind it: if an issuer reassigned a tenant identifier to another
tenant, contrary to its definition ({{conventions}}), the new tenant
would receive the catalog content of the old one.

## Protected Discovery and Client Authentication {#sec-protected-discovery}

A Relationship View is disclosed to an Issuer Relationship, not to a
client, so its retrieval is not the protected discovery mechanism of
{{Section 9.7 of ID-JAG}}, which requires client authentication and
is an open issue ({{open-issues}}). The Grant
Issuer's signature on the catalog
access assertion binds a response to the relationship, so client
authentication at the token endpoint is required only when the
Onboarding Arrangement requires it ({{relationship-authorization}},
{{token-request}}).

When the Onboarding Arrangement does not require client authentication,
the catalog access assertion is the only credential the Grant Issuer
presents. An assertion that is not DPoP-bound is then a bearer
credential until the Resource Authorization Server accepts it: the
replay check ({{assertion-validation}}) prevents its reuse but not its
first use by a party that obtains it, and binding only the catalog
access token does not prevent that use. A DPoP-bound catalog access
assertion cannot be used without the private key that its `cnf` claim
identifies ({{token-request}}). DPoP binding demonstrates possession of
a key that the Grant Issuer names under its own signature; it is not
client authentication. The assertion's short lifetime, its audience
restriction, and TLS further limit the exposure. A Resource
Authorization Server weighs the confidentiality of its catalog content
when deciding whether its Onboarding Arrangements require client
authentication or DPoP-bound catalog access assertions.

## Catalog Access Token Disclosure {#sec-token-disclosure}

A disclosed catalog access token permits retrieval of the Relationship
View until it expires or the relationship ends. The rules for where the
Grant Issuer sends it ({{view-document}}) limit its disclosure, and its
restriction to the catalog resource and scope ({{token-response}})
prevents its use at other protected resources of the Resource
Authorization Server.

## Response Authenticity {#sec-authenticity}

Relationship View documents are not signed. Their authenticity and
integrity rest on TLS server authentication of the catalog resource and
on validation of the Resource Authorization Server's metadata
({{Section 3.3 of RFC8414}}). An attacker able to alter catalog
responses can cause an administrator of the Grant Issuer to approve
grants that the Resource Authorization Server did not offer, or to deny
approval of grants that it did offer. The attacker cannot cause the
Resource Authorization Server to accept a grant that its current policy
rejects.

## Stale Catalogs and Relationship Termination {#sec-stale}

When the Resource Authorization Server narrows what it accepts, a Grant
Issuer that has not refreshed issues grants that the Resource
Authorization Server rejects after their claims have been disclosed to
the client. Such grants affect availability and disclosure, not
authorization.

After a relationship ends and disclosure stops
({{relationship-authorization}}), the maximum age that the Grant Issuer
configures bounds how long it continues to issue grants under state that
it can no longer refresh, but the Resource Authorization Server cannot
enforce that bound. To stop issuance at a known time, the Resource
Authorization Server first removes every eligible tenant from the
relationship ({{installation}}) and ends the relationship only after the
Grant Issuer observes the withdrawal.

## Revision Integrity and Approval {#sec-revision}

A Resource Authorization Server that changes the normative semantics of
an entry without changing its revision causes the Grant Issuer to
continue issuing grants under an approval that no longer matches the
entry. The Grant Issuer detects such a change only when it holds an
earlier copy of the entry with the same revision ({{validation}}). A
revision does not bind the Resource Authorization Server to the meaning
that the Grant Issuer approved, and a grant issued before a change can
be redeemed after it. Requiring an approval for the installed revision
({{installation}}) avoids relying on a Grant Issuer's comparison of
two revisions to decide that a change only narrows an entry; an
incorrect comparison would expand the approved subset.

## Tenant Confusion {#sec-tenant}

If a Grant Issuer derived a grant's target tenant from the number of
Tenant Objects in a Relationship View or the form of an issuer
identifier, a grant intended for one tenant could be
accepted for another ({{issuing}}). An entry obtained for one Issuer
Relationship is used only for grants whose issuer and issuer tenant
identify that relationship ({{issuing}}), so that a catalog disclosed to
one issuer tenant does not determine the grants issued for subjects of
another.

## Tenant Eligibility {#sec-eligibility}

If a Resource Authorization Server makes a tenant eligible for an Issuer
Relationship because an administrator of that tenant configured trust in
the Grant Issuer, anyone who controls a tenant at the Resource
Authorization Server can add a Tenant Object, with a `display` of their
choosing, to the Grant Issuer's Relationship View. If an administrator
of the Grant Issuer approves an entry for that Tenant Object, clients
obtain grants that disclose claims to that tenant. A Resource
Authorization Server SHOULD make a tenant eligible for an Issuer
Relationship only when the Onboarding Arrangement records agreement on
the Grant Issuer side, for example by an administrator of the issuer
tenant. An administrator of the Grant Issuer identifies a Tenant Object
by its `tenant_id`, not by its `display` ({{model-conventions}}).

## Cross-JWT Confusion {#sec-cross-jwt}

A Resource Authorization Server that accepts JWT authorization grants
and catalog access assertions at the same token endpoint relies on the
`typ` JOSE header parameter to distinguish them
({{assertion-validation}}, {{profile-definitions}}). Without that check,
a grant could be presented as a catalog access assertion, or a catalog
access assertion as a grant; their claims overlap, and the `sub` claim
of a catalog access assertion is an issuer identifier. When the client
identifier assigned to a Grant Issuer equals its issuer identifier, a
catalog access assertion also has the `iss`, `sub`, and `aud` values of
a client assertion ({{Section 3 of RFC7523}}). One intercepted JWT could
then serve as both the grant and the client authentication of a request,
and a validator of client assertions that ignores `typ` does not prevent
this. {{token-request}} forbids using a catalog access assertion as a
client assertion and recommends a separate key for client assertions.
{{Section 2.8 of RFC8725}} describes this class of attack. For the same
reason, an entry is used only for grants of its own Grant Profile
({{issuing}}), even when the grants of another profile carry similar
claims.

## External References {#sec-external}

Entries and authorization details types metadata contain URIs that refer
to other servers: the `documentation_uri`, `definition_uri`, and
`schema_uri` members, and the `resource` of each entry, whose Protected
Resource Metadata the Grant Issuer requests during composition
({{composition}}). A Resource Authorization Server, or an attacker that
can alter catalog responses, chooses these URIs
({{Section 7.7 of RFC9728}}). A Grant Issuer that dereferences them
sends no credential obtained through a retrieval protocol
({{optional-metadata}}, {{view-document}}) and applies the protections
against server-side request forgery that it applies to other URLs
obtained from external parties. The retrieval protocol does not
authenticate content retrieved from these URIs, so the Grant Issuer
treats it as untrusted input, not as catalog content. A Grant Issuer can
cache Protected Resource Metadata and limit how many such requests one
refresh makes.

A Grant Issuer presents `display`, `description`, `documentation_uri`,
and `definition_uri` to administrators as untrusted text. These members
can change without a new revision ({{revision}}), so an approval does
not cover later changes to them.

## Resource Consumption {#sec-resources}

A Relationship View can be large. A Grant Issuer SHOULD limit the size
of the responses it accepts, and treats a response that exceeds its
limits as an incomplete retrieval ({{installation}}). A Resource
Authorization Server can limit the rate at which it accepts catalog
access assertions and requests to the catalog endpoint from an Issuer
Relationship.

# Privacy Considerations {#privacy}

The privacy considerations of the Grant Profile of each entry, such as
{{ID-JAG}}, apply to this specification.

A catalog discloses business relationships: which Resource Authorization
Server tenants are associated with an issuer tenant, and the names those
tenants use. This specification limits that disclosure to an
authenticated and authorized Issuer Relationship
({{relationship-authorization}}). It also requires
responses that caches do not store ({{refresh}}) and prohibits error
responses that reveal other issuers, tenants, or catalogs ({{errors}}).

Because an entry states the claims and subject identifier formats that
the Resource Authorization Server requires or accepts ({{entry-claims}},
{{entry-sub-id}}), an administrator of the Grant Issuer decides before
issuance which claims the Grant Issuer releases and to which Resource
Authorization Server tenants. An entry that requires a claim does not
cause the Grant Issuer to release it without approval ({{issuing}}).

The Grant Issuer stores catalog content as configuration, protects it
like other relationship-specific configuration, and makes it accessible
only to administrators who manage the corresponding Issuer Relationship.

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
: {{relationship-view}} and {{view-document}} of this specification

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
: binary; a catalog access assertion is a JWT, which is a sequence of
  base64url-encoded values separated by period (".") characters.

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

## OAuth Scope Value

No IANA registry exists for OAuth scope values, and this specification
requests no registration for `authorization_grant_catalog.read`
({{token-request}}).

--- back

# Example Grant Catalog {#example-catalog}

The following Grant Catalog, from a Relationship View whose
`authorization_server` is `https://ras.example/`, has one entry for the
ID-JAG profile ({{id-jag}}). Every ID-JAG issued using the entry is
bound to a DPoP key and contains the scope value `messages.read`, the
`email` claim, and a `sub_id` claim in the `saml-nameid` format; it can
also contain the scope value `messages.write` and authorization details
objects of type `message_channel`.

~~~ json
{
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
`reports.read` or `reports.export`, but not both, and requires the
`department` claim only with `reports.export`. A single entry cannot
express either rule ({{envelope}}), so the Resource Authorization
Server `https://ras.example/` publishes two entries:

~~~ json
{
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

An administrator of the Grant Issuer approves each entry separately for
each Tenant Object; for example, it can approve `reports-read` for two
tenants and `reports-export` for only one of them.

An ID-JAG issued using `reports-read` does not contain the `department`
claim, which that entry does not list ({{entry-claims}}), and no ID-JAG
contains both scope values, because each grant is issued using exactly
one entry ({{issuing}}).

# Example Approval Lifecycle {#example-lifecycle}

An approval of the entry in {{example-catalog}} for the Tenant Object
`customer-acme-eu` changes as the Relationship View changes:

1. The Grant Issuer installs a Relationship View in which
   `customer-acme-eu` references the Grant Catalog `standard`, whose
   entry `messages-api-id-jag` has the revision `5`. An administrator
   approves the scope value `messages.read` for that entry, and the
   Grant Issuer issues ID-JAGs with that scope value.

2. A refresh fails with a transport error. The Grant Issuer installs
   nothing and continues to issue ID-JAGs under the installed state,
   within the maximum age it has configured ({{installation}}).

3. The Resource Authorization Server removes `customer-acme-eu` from the
   Issuer Relationship. The next refresh retrieves a Relationship View
   without it, and the Grant Issuer suspends issuance for
   `customer-acme-eu` ({{installation}}).

4. The Resource Authorization Server restores `customer-acme-eu` with
   the same `grant_catalog`, and the Grant Catalog still
   contains revision `5` of the entry. The approval applies again
   without new approval ({{installation}}).

5. The Resource Authorization Server adds the optional scope value
   `messages.archive` to the entry and changes its revision to `6`. The
   Grant Issuer stops issuing ID-JAGs under the approval for revision
   `5`, even though the change does not affect `messages.read`
   ({{installation}}).

6. An administrator reviews revision `6` and approves `messages.read`
   again. The Grant Issuer resumes issuing ID-JAGs with that scope
   value.

# Design Rationale {#rationale}

Data model separate from retrieval:
: Deployments differ in the administrative relationship that already
  exists between a Grant Issuer and a Resource Authorization Server.
  Objects defined independently of retrieval let any retrieval protocol
  deliver identical objects to the same Grant Issuer policy.

Complete documents and no merge rules:
: Because each Tenant Object references one Grant Catalog that is
  neither paged nor merged ({{tenant-object}}, {{grant-catalog}}), a
  Grant Issuer determines what a tenant is offered from one object and
  cannot mistake a partial retrieval for a complete one.

One response per Relationship View:
: One complete response gives the Grant Issuer one consistent state
  without a manifest of versions, and makes a withdrawal the absence of
  an object from a validated response. Compression and conditional
  requests ({{refresh}}) offset the cost of retransmitting the whole
  view until deployments show a need for incremental retrieval.

Authority in public metadata, eligibility in the catalog:
: What a scope value or authorization details type authorizes is the
  same for every Issuer Relationship, so it belongs in public metadata,
  such as authority metadata {{AUTHORITY-METADATA}}. Which scope values
  and authorization details types a relationship may use is specific to
  the relationship, so it belongs in the catalog. Entries identify items
  by the protocol values that authority metadata uses, so the two
  compose without a mapping ({{optional-metadata}}).

Envelope without alternatives:
: An Authorization Envelope that is a product of independent selections
  ({{envelope}}) can be evaluated without a rule language, and a
  Grant Issuer cannot misread a rule that does not exist.

Opaque revision identifiers:
: The format leaves the generation of revision values to the Resource
  Authorization Server, which can use a counter or a digest of the
  normative semantics, and prescribes no canonicalization. Because a
  revision is tied to the normative semantics ({{revision}}), the Grant
  Issuer retains approvals across changes to presentation text.

Optional claims listed explicitly:
: Exempting some optional claims, such as an email address, from listing
  ({{entry-claims}}) would leave their release outside the
  administrator's approval, with no principled line between them and
  related claims, such as one that states whether that email address is
  verified.

One resource per grant:
: Limiting a grant to one protected resource ({{envelope}}) avoids
  rules for how scope values and authorization details apply across
  resources, and keeps each approval about one resource.

Signed relationship assertion:
: Stating the Issuer Relationship under the signature of the keys that
  sign the Grant Issuer's grants ({{assertion}}) gives every Resource
  Authorization Server the same provenance for the relationship, reuses
  the trust it already places in the issuer, and binds catalog access to
  the identity that issues the grants the catalog describes. Authorizing
  a tenant selector against the relationships permitted to a Grant
  Issuer authenticated as a client would instead leave each Resource
  Authorization Server to define its own selector and mapping, and a
  client credential authorized for many issuer tenants would carry the
  same broad authority ({{sec-relationship}}).

No client inventory:
: Client identifiers can be shared across trust domains ({{CIMD}}) or
  established by registration, and a client inventory ({{clients}})
  would require additional disclosure policy and synchronization of
  client registration state. A view for one identified client that the
  Issuer Relationship is authorized to inspect would disclose no
  inventory, and could be defined later.

# Open Issues {#open-issues}
{:numbered="false"}

RFC Editor: please remove this appendix before publication.

* Reapproval: whether issuance can continue under an existing approval
  when an entry changes to a revision whose Authorization Envelope still
  contains the approved subset, and what a Resource Authorization Server
  would publish to make that safe ({{installation}},
  {{example-lifecycle}}). Tracked as [issue
  #12](https://github.com/mcguinness/draft-mcguinness-oauth-grant-catalog/issues/12).

* Client-facing discovery: whether to define the protected discovery
  mechanism that {{Section 9.7 of ID-JAG}} permits, through which an
  application client retrieves the entries that apply to it. A view of
  whole entries cannot express a restriction for one client that is
  narrower than an entry. Tracked as [issue
  #15](https://github.com/mcguinness/draft-mcguinness-oauth-grant-catalog/issues/15).

* Freshness and termination: whether to require a maximum age, or let
  the Resource Authorization Server publish one, so that it has a
  deadline after which a Grant Issuer stops issuing under state it
  cannot refresh ({{installation}}, {{sec-stale}}). Tracked as [issue
  #16](https://github.com/mcguinness/draft-mcguinness-oauth-grant-catalog/issues/16).

* Must-understand extensions: a mechanism, to be added before the format
  stabilizes, by which an entry marks extension members that a Grant
  Issuer must implement, so that restrictive extensions need no distinct
  Grant Profile identifier ({{extensibility}}); and whether every
  extension is to preserve the principles in {{overview}}. Tracked as
  [issue
  #19](https://github.com/mcguinness/draft-mcguinness-oauth-grant-catalog/issues/19).

# Acknowledgments
{:numbered="false"}

This specification builds on the Identity Assertion JWT Authorization
Grant {{ID-JAG}} and on the JWT authorization grant {{RFC7523}}.

# Document History
{:numbered="false"}

RFC Editor: please remove this appendix before publication.

-00

* Initial version.
