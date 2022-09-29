[RFC](https://www.rfc-editor.org/rfc/rfc7662)

### Abstract 
This specification defines a method for a protected resource to query an OAuth 2.0 authorization server to determine the active state of an OAuth 2.0 token and to determine meta-information about this token. OAuth 2.0 deployments can use this method to convey information about the authorization context of the token from the authorization server to the protected resource.

### [1](https://www.rfc-editor.org/rfc/rfc7662#section-1).  Introduction
This specification defines a protocol that allows authorized protected resources to query the authorization server to determine the set of metadata for a given token that was presented to them by an OAuth 2.0 client.  This metadata includes whether or not the token is currently active (or if it has expired or otherwise been revoked), what rights of access the token carries (usually conveyed through OAuth 2.0 scopes), and the authorization context in which the token was granted (including who authorized the token and which client it was issued to).  Token introspection allows a protected resource to query this information regardless of whether or not it is carried in the token itself, allowing this method to be used along with or independently of structured token values.  Additionally, a protected resource can use the mechanism described in this specification to introspect the token in a particular authorization decision context and ascertain the relevant metadata about the token to make this authorization decision appropriately.

#### [1.2](https://www.rfc-editor.org/rfc/rfc7662#section-1.2).  Terminology
- **Token Introspection**: The act of inquiring about the current state of an OAuth 2.0 token through use of the network protocol defined in this document. 
- **Introspection Endpoint**: The OAuth 2.0 endpoint through which the token introspection operation is accomplished.

### [2](https://www.rfc-editor.org/rfc/rfc7662#section-2).  Introspection Endpoint
The introspection endpoint is an OAuth 2.0 endpoint that takes a parameter representing an OAuth 2.0 token and returns a JSON document representing the meta information surrounding the token, including whether this token is currently active.

#### [2.1](https://www.rfc-editor.org/rfc/rfc7662#section-2.1).  Introspection Request
- **token**: REQUIRED
- token_type_hint: OPTIONAL
The introspection endpoint MAY accept other OPTIONAL parameters to provide further context to the query.  For instance, an authorization server may desire to know the IP address of the client accessing the protected resource to determine if the correct client is likely to be presenting the token.

#### [2.2](https://www.rfc-editor.org/rfc/rfc7662#section-2.2).  Introspection Response
- **active**: REQUIRED.  Boolean indicator of whether or not the presented token is currently active.  The specifics of a token's "active" state will vary depending on the implementation of the authorization server and the information it keeps about its tokens, but a "true" value return for the "active" property will generally indicate that a given token has been issued by this authorization server, has not been revoked by the resource owner, and is within its given time window of validity (e.g., after its issuance time and before its expiration time).
- **scope**: OPTIONAL.  A JSON string containing a space-separated list of scopes associated with this token. 
- **client_id** :OPTIONAL.  Client identifier for the OAuth 2.0 client that requested this token.
- **username**: OPTIONAL.  Human-readable identifier for the resource owner who authorized this token. 
- **token_type**: OPTIONAL.
- **exp**: OPTIONAL.  Integer timestamp, measured in the number of seconds since January 1 1970 UTC, indicating when this token will expire.
- **iat**: OPTIONAL.  Integer timestamp, measured in the number of seconds since January 1 1970 UTC, indicating when this token was originally issued.
- **nbf**: OPTIONAL.  Integer timestamp, measured in the number of seconds since January 1 1970 UTC, indicating when this token is not to be used before.
- **sub**: OPTIONAL.  Subject of the token. Usually a machine-readable identifier of the resource owner who authorized this token. 
- **aud**: OPTIONAL.  Service-specific string identifier or list of string identifiers representing the intended audience for this token.
- **iss**: OPTIONAL.  String representing the issuer of this token.
- **jti**: OPTIONAL.  String identifier for the token.

Specific implementations MAY extend this structure with their own service-specific response names as top-level members of this JSON object.

     HTTP/1.1 200 OK
     Content-Type: application/json

     {
      "active": true,
      "client_id": "l238j323ds-23ij4",
      "username": "jdoe",
      "scope": "read write dolphin",
      "sub": "Z5O3upPC88QrAjx00dis",
      "aud": "https://protected.example.net/resource",
      "iss": "https://server.example.com/",
      "exp": 1419356238,
      "iat": 1419350238,
      "extension_field": "twenty-seven"
     }

