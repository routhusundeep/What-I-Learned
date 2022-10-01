[Registration RFC](https://www.rfc-editor.org/rfc/rfc7591) and [Management RFC](https://www.rfc-editor.org/rfc/rfc7592)

### Abstract 
This specification defines mechanisms for dynamically registering OAuth 2.0 clients with authorization servers. Registration requests send a set of desired client metadata values to the authorization server. The resulting registration responses return a client identifier to use at the authorization server and the client metadata values registered for the client.  The client can then use this registration information to communicate with the authorization server using the OAuth 2.0 protocol. This specification also defines a set of common client metadata fields and values for clients to use during registration.


#### [1.3](https://www.rfc-editor.org/rfc/rfc7591#section-1.3).  Protocol Flow
        +--------(A)- Initial Access Token (OPTIONAL)
        |
        |   +----(B)- Software Statement (OPTIONAL)
        |   |
        v   v
    +-----------+                                      +---------------+
    |           |--(C)- Client Registration Request -->|    Client     |
    | Client or |                                      | Registration  |
    | Developer |<-(D)- Client Information Response ---|   Endpoint    |
    |           |        or Client Error Response      +---------------+
    +-----------+

- (A) Optionally, the client or developer is issued an initial access token giving access to the client registration endpoint.
- (B) Optionally, the client or developer is issued a software statement for use with the client registration endpoint.
- (C) The client or developer calls the client registration endpoint with the client's desired registration metadata, optionally including the initial access token from (A) if one is required by the authorization server. 
- (D) The authorization server registers the client and returns: 
	- the client's registered metadata 
	- a client identifier that is unique at the server, and *  a set of client credentials such as a client secret, if applicable for this client.

### [2](https://www.rfc-editor.org/rfc/rfc7591#section-2).  Client Metadata
 These client metadata values are used in two ways:
 - as input values to registration requests, and
 - as output values in registration responses.


- **redirect_uris**: Array of redirection URI strings for use in redirect-based flows such as the authorization code and implicit flows.
- **token_endpoint_auth_method**: String indicator of the requested authentication method for the token endpoint.
	- "**none**": The client is a public client, and does not have a client secret.
	- "**client_secret_post**": The client uses the HTTP POST parameters.
	- "**client_secret_basic**": The client uses HTTP Basic
- **grant_types**: Array of OAuth 2.0 grant type strings that the client can use at the token endpoint.  These grant types are defined as follows:
	- authorization_code
	- implicit
	- password
	- client_credentials
	- refresh_token
	- urn:ietf:params:oauth:grant-type:jwt-bearer: The JWT Bearer Token Grant Type defined in OAuth JWT Bearer Token Profiles
	- urn:ietf:params:oauth:grant-type:saml2-bearer: The SAML 2.0 Bearer Assertion Grant defined in OAuth SAML 2 Bearer Token Profiles
- **response_types**: Array of the OAuth 2.0 response type strings that the client can use at the authorization endpoint.  These response types are defined as follows:
	- code
	- token
- **client_name**: Human-readable string name of the client to be presented to the end-user during authorization.  If omitted, the authorization server MAY display the raw "client_id" value to the end-user instead.  It is RECOMMENDED that clients always send this field. The value of this field MAY be internationalized
- **client_uri**: URL string of a web page providing information about the client. If present, the server SHOULD display this URL to the end-user in a clickable fashion.  It is RECOMMENDED that clients always send this field.  The value of this field MUST point to a valid web page.  The value of this field MAY be internationalized
- **logo_uri**: URL string that references a logo for the client.  If present, the server SHOULD display this image to the end-user during approval. The value of this field MUST point to a valid image file.  The value of this field MAY be internationalized
- **scope**: String containing a space-separated list of scope values
- **contacts**: Array of strings representing ways to contact people responsible for this client, typically email addresses.  The authorization server MAY make these contact addresses available to end-users for support requests for the client.
- **tos_uri**: URL string that points to a human-readable terms of service document for the client that describes a contractual relationship between the end-user and the client that the end-user accepts when authorizing the client.  The authorization server SHOULD display this URL to the end-user if it is provided.
- **policy_uri**: URL string that points to a human-readable privacy policy document that describes how the deployment organization collects, uses, retains, and discloses personal data.
- **jwks_uri**: URL string referencing the client's JSON Web Key (JWK) Set document, which contains the client's public keys
- **jwks**: Client's JSON Web Key Set document value, which contains the client's public keys.  The value of this field MUST be a JSON object containing a valid JWK Set.
- **software_id**: A unique identifier string (e.g., a Universally Unique Identifier (UUID)) assigned by the client developer or software publisher used by registration endpoints to identify the client software to be dynamically registered.  Unlike "client_id", which is issued by the authorization server and SHOULD vary between instances, the "software_id" SHOULD remain the same for all instances of the client software.
- **software_version**: A version identifier string for the client software identified by "software_id".  The value of the "software_version" SHOULD change on any update to the client software identified by the same "software_id".

#### [2.1](https://www.rfc-editor.org/rfc/rfc7591#section-2.1).  Relationship between Grant Types and Response Types
	   +-----------------------------------------------+-------------------+
	   | grant_types value includes:                   | response_types    |
	   |                                               | value includes:   |
	   +-----------------------------------------------+-------------------+
	   | authorization_code                            | code              |
	   | implicit                                      | token             |
	   | password                                      | (none)            |
	   | client_credentials                            | (none)            |
	   | refresh_token                                 | (none)            |
	   | urn:ietf:params:oauth:grant-type:jwt-bearer   | (none)            |
	   | urn:ietf:params:oauth:grant-type:saml2-bearer | (none)            |
	   +-----------------------------------------------+-------------------+

#### [2.2](https://www.rfc-editor.org/rfc/rfc7591#section-2.2).  Human-Readable Client Metadata
Human-readable client metadata values and client metadata values that reference human-readable values MAY be represented in multiple languages and scripts.  For example, the values of fields such as "client_name", "tos_uri", "policy_uri", "logo_uri", and "client_uri" might have multiple locale-specific values in some client registrations to facilitate use in different locations.

### [3](https://www.rfc-editor.org/rfc/rfc7591#section-3).  Client Registration Endpoint
The client registration endpoint MUST accept HTTP POST messages with request parameters encoded in the entity body using the "application/json" format. To support open registration and facilitate wider interoperability, the client registration endpoint SHOULD allow registration requests with no authorization (which is to say, with no initial access token in the request).  These requests MAY be rate-limited or otherwise limited to prevent a denial-of-service attack on the client registration endpoint.

#### [3.1](https://www.rfc-editor.org/rfc/rfc7591#section-3.1).  Client Registration Request
To register, the client or developer sends an HTTP POST to the client registration endpoint with a content type of "application/json".  The HTTP Entity Payload is a JSON [RFC7159] document consisting of a JSON object and all requested client metadata values as top-level members of that JSON object.
     POST /register HTTP/1.1
     Content-Type: application/json
     Accept: application/json
     Host: server.example.com

     {
      "redirect_uris": [
        "https://client.example.org/callback",
        "https://client.example.org/callback2"],
      "client_name": "My Example Client",
      "client_name#ja-Jpan-JP":
         "\u30AF\u30E9\u30A4\u30A2\u30F3\u30C8\u540D",
      "token_endpoint_auth_method": "client_secret_basic",
      "logo_uri": "https://client.example.org/logo.png",
      "jwks_uri": "https://client.example.org/my_public_keys.jwks",
      "example_extension_parameter": "example_value"
     }

#### [3.2.1](https://www.rfc-editor.org/rfc/rfc7591#section-3.2.1).  Client Information Response
- **client_id** REQUIRED.  OAuth 2.0 client identifier string.  It SHOULD NOT be currently valid for any other registered client, though an authorization server MAY issue the same client identifier to multiple instances of a registered client at its discretion. 
- **client_secret** OPTIONAL.  OAuth 2.0 client secret string.  If issued, this MUST be unique for each "client_id" and SHOULD be unique for multiple instances of a client using the same "client_id".  This value is used by confidential clients to authenticate to the token endpoint.
- **client_id_issued_at** OPTIONAL.  Time at which the client identifier was issued.  The time is represented as the number of seconds from 1970-01-01T00:00:00Z as measured in UTC until the date/time of issuance. 
- **client_secret_expires_at** REQUIRED if "client_secret" is issued.  Time at which the client secret will expire or 0 if it will not expire.  The time is represented as the number of seconds from 1970-01-01T00:00:00Z as measured in UTC until the date/time of expiration.

Additionally, the authorization server MUST return all registered metadata about this client, including any fields provisioned by the authorization server itself.


#### [2.2](https://www.rfc-editor.org/rfc/rfc7592#section-2.2).  Client Update Request
To update a previously registered client's registration with an authorization server, the client makes an HTTP PUT request to the client configuration endpoint with a content type of "application/ json".

#### [2.3](https://www.rfc-editor.org/rfc/rfc7592#section-2.3).  Client Delete Request
To deprovision itself on the authorization server, the client makes an HTTP DELETE request to the client configuration endpoint.

### [3](https://www.rfc-editor.org/rfc/rfc7592#section-3).  Client Information Response
This specification extends the client information response defined in "OAuth 2.0 Client Dynamic Registration", which states that the response contains the client identifier (as well as the client secret if the client is a confidential client).  When used with this specification, the client information response also contains the fully qualified URL of the client configuration endpoint (Section 2) for this specific client that the client or developer may use to manage the client's registration configuration, as well as a registration access token that is to be used by the client or developer to perform subsequent operations at the client configuration endpoint.

- **registration_client_uri** REQUIRED.  String containing the fully qualified URL of the client configuration endpoint for this client. 
- **registration_access_token** REQUIRED.  String containing the access token to be used at the client configuration endpoint to perform subsequent operations upon the client registration.