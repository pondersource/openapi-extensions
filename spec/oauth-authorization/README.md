# OpenAPI OAuth Authorization Profiles Extension

**Spec version:** 0.1.0

---

## 1. Introduction

OAuth authorization servers can attach semantics to authorization-request
parameters that are not represented by the OAuth Flow Object. For example, one
value can request access while the user is absent, while another limits access
to the current session. These choices can also affect whether a token response
can contain a refresh token.

This proposal adds **x-oauth-authorization-profile** to an OAuth
**authorizationCode** Flow Object. It describes an API-defined authorization
profile through fixed query parameters, capabilities, and token-issuance
semantics. Separate Security Scheme Objects can expose separate profiles, so an
operation can select or offer them using the standard Security Requirement
Object.

The extension describes authorization-server behavior. It does not prescribe
which profile a client selects, how a client stores tokens, or when a client
refreshes them.

## 2. Overview

~~~yaml
components:
  securitySchemes:
    serviceOnline:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://authorization.example/authorize
          tokenUrl: https://authorization.example/token
          scopes:
            records.read: Read records
          x-oauth-authorization-profile:
            parameters:
              - parameter:
                  name: access_type
                  in: query
                  schema:
                    type: string
                    enum: [online, offline]
                value: online
            tokenIssuance:
              refreshToken:
                presence: never

    serviceOffline:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://authorization.example/authorize
          tokenUrl: https://authorization.example/token
          scopes:
            records.read: Read records
          x-oauth-authorization-profile:
            parameters:
              - parameter:
                  name: access_type
                  in: query
                  schema:
                    type: string
                    enum: [online, offline]
                value: offline
            capabilities: [offlineAccess]
            tokenIssuance:
              refreshToken:
                presence: conditional
                conditions: [firstAuthorization]
                description: >-
                  The authorization server can omit the refresh token when it
                  has previously issued one for this authorization.
~~~

## 3. Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", and "MAY" are to be
interpreted as described in
[BCP 14](https://www.rfc-editor.org/info/bcp14) when, and only when, they appear
in all capitals.

This extension uses the OAuth terminology defined by
[RFC 6749](https://www.rfc-editor.org/rfc/rfc6749), and the OpenAPI Object and
Reference Object rules from the
[OpenAPI Specification](https://spec.openapis.org/oas/latest.html).

## 4. Object Definitions

### 4.1 OAuth Authorization Profile Object

**x-oauth-authorization-profile** MAY appear on an OAuth Flow Object whose
containing flow name is **authorizationCode**. It MUST NOT appear on another
OAuth flow.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| **parameters** | [Authorization Parameter Value Object] | no | Fixed parameters that the client adds to the authorization request. |
| **capabilities** | [string] | no | Authorization capabilities requested by this profile. Values defined here are listed below. Other values MUST be absolute URIs. |
| **tokenIssuance** | Token Issuance Object | no | Token-response issuance semantics associated with this profile. |

At least one field MUST be present. Array entries MUST be unique by parameter
name or capability value, as applicable.

The following capability is defined by this specification:

| Value | Meaning |
| --- | --- |
| **offlineAccess** | The profile requests authorization that can be used while the resource owner is absent. If granted, this normally requires a refresh token or another server-defined mechanism. |

The capability describes what the authorization request asks the authorization
server to grant. It does not state that the request will be approved.

### 4.2 Authorization Parameter Value Object

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| **parameter** | Parameter Object or Reference Object | yes | The authorization endpoint query parameter. |
| **value** | any | yes | The fixed value for this profile. |
| **description** | string | no | Additional profile-specific semantics of this value. |

**parameter** MUST be, or resolve to, an OpenAPI Parameter Object with
**in: query**. If the authorization endpoint is described by an OpenAPI
operation, this field SHOULD reference the same Parameter Object.

The value MUST conform to the parameter schema and MUST be serialized according
to the Parameter Object. A parameter name MUST NOT be **client_id**,
**redirect_uri**, **response_type**, **scope**, or **state**. Those parameters
already have protocol-defined construction rules and are not fixed profile
metadata.

### 4.3 Token Issuance Object

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| **refreshToken** | Refresh Token Issuance Object | no | Whether a successful authorization-code exchange includes a refresh token. |

### 4.4 Refresh Token Issuance Object

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| **presence** | string | yes | One of **always**, **conditional**, or **never**. |
| **conditions** | [string] | conditionally | Conditions under which the token is issued. REQUIRED when **presence** is **conditional**. Values defined here are listed below; other values MUST be absolute URIs. |
| **description** | string | conditionally | Human-readable issuance conditions. REQUIRED when **presence** is **conditional**. |

**always** means every successful token response for this profile contains a
refresh token. **conditional** means a successful response can omit it; clients
MUST tolerate its absence. **never** means a successful response does not
contain one.

These values describe issuance, not the outcome of authorization. In
particular, **conditional** MUST be used when prior consent, a previously issued
token, server policy, or another condition can cause omission. A profile MUST
NOT use **always** merely because it requests **offlineAccess**.

The following condition is defined by this specification:

| Value | Meaning |
| --- | --- |
| **firstAuthorization** | The authorization server issues the token only when it recognizes the exchange as the first authorization for the client and resource owner. |

Conditions explain when issuance occurs; they do not turn **conditional** into
a guarantee because authorization-server state and policy determine whether a
condition holds.

## 5. Applying via OpenAPI Overlays

An Overlay can add profiles without changing the source description. The
following action gives an existing authorization-code flow an online profile;
a second action can copy the Security Scheme and apply the offline values.

~~~yaml
overlay: 1.0.0
info:
  title: Online authorization profile
  version: 1.0.0
actions:
  - target: >-
      $.components.securitySchemes.serviceOnline.flows.authorizationCode
    update:
      x-oauth-authorization-profile:
        parameters:
          - parameter:
              name: access_type
              in: query
              schema:
                type: string
                enum: [online, offline]
            value: online
        tokenIssuance:
          refreshToken:
            presence: never
~~~

## 6. Example: Google Online and Offline Access

The two schemes below let an API operation offer online and offline
authorization independently. Google's **access_type=offline** requests offline
access, but a refresh token is generally returned only on the first
authorization. The offline profile therefore declares conditional issuance.

~~~yaml
components:
  parameters:
    googleAccessType:
      name: access_type
      in: query
      required: false
      schema:
        type: string
        enum: [online, offline]

  securitySchemes:
    googleOnline:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://accounts.google.com/o/oauth2/v2/auth
          tokenUrl: https://oauth2.googleapis.com/token
          scopes:
            https://www.googleapis.com/auth/drive.metadata.readonly: >-
              View metadata for files in Google Drive
          x-oauth-authorization-profile:
            parameters:
              - parameter:
                  $ref: '#/components/parameters/googleAccessType'
                value: online
            tokenIssuance:
              refreshToken:
                presence: never

    googleOffline:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://accounts.google.com/o/oauth2/v2/auth
          tokenUrl: https://oauth2.googleapis.com/token
          scopes:
            https://www.googleapis.com/auth/drive.metadata.readonly: >-
              View metadata for files in Google Drive
          x-oauth-authorization-profile:
            parameters:
              - parameter:
                  $ref: '#/components/parameters/googleAccessType'
                value: offline
            capabilities: [offlineAccess]
            tokenIssuance:
              refreshToken:
                presence: conditional
                conditions: [firstAuthorization]
                description: >-
                  Google returns a refresh token on the first authorization;
                  later authorization-code exchanges can omit it.

paths:
  /drive-files:
    get:
      security:
        - googleOnline:
            - https://www.googleapis.com/auth/drive.metadata.readonly
        - googleOffline:
            - https://www.googleapis.com/auth/drive.metadata.readonly
      responses:
        '200':
          description: File metadata
~~~

The Security Requirement Object above makes the profiles alternatives. A
client chooses between them based on its own requirements; that choice is
outside this extension.

## 7. Validation

A conforming validator MUST enforce these rules:

1. The extension appears only on an **authorizationCode** OAuth Flow Object.
2. Every parameter is an inline Parameter Object or a Reference Object that
   resolves to one, and its **in** value is **query**.
3. Parameter names are unique within a profile and do not use the reserved
   OAuth names listed in section 4.2.
4. Each fixed value validates against its parameter schema.
5. Capability values are unique and non-empty. An unrecognized capability is
   an absolute URI, preventing collisions with future values defined here.
6. Refresh-token **presence** is **always**, **conditional**, or **never**.
   **conditions** is non-empty and **description** is present and non-empty for
   conditional issuance. Conditions are unique; an unrecognized condition is
   an absolute URI.
7. A profile containing **offlineAccess** declares refresh-token issuance as
   **always** or **conditional**.
8. If the token endpoint response is described in the same OpenAPI document,
   its schema does not contradict the declared presence: **always** requires
   the refresh-token property, **conditional** permits it but does not require
   it, and **never** does not require it.

A validator SHOULD warn when two Security Scheme Objects differ only in name
but resolve to indistinguishable authorization profiles.

## 8. Security Considerations

This extension contributes fixed values only to the authorization request. It
MUST NOT be used to carry client credentials, access tokens, refresh tokens, or
arbitrary token-endpoint URLs. Tooling MUST apply the normal OAuth protections
for redirect URI validation, state correlation, PKCE, and authorization-server
metadata.

Descriptions are explanatory and MUST NOT override the normative fields. A
client MUST treat **conditional** refresh-token issuance as absence-tolerant
even when a description lists common conditions.

## References

- [OAuth 2.0, RFC 6749](https://www.rfc-editor.org/rfc/rfc6749)
- [OAuth 2.0 Authorization Server Metadata, RFC 8414](https://www.rfc-editor.org/rfc/rfc8414)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [OpenAPI Overlay Specification](https://spec.openapis.org/overlay/v1.0.0.html)
- [Google OAuth 2.0 for Web Server Applications](https://developers.google.com/identity/protocols/oauth2/web-server#offline)
