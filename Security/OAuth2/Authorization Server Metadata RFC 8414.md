[RFC](https://www.rfc-editor.org/rfc/rfc8414)

### [1](https://www.rfc-editor.org/rfc/rfc8414#section-1).  Introduction
This specification generalizes the metadata format defined by "OpenID Connect Discovery 1.0" in a way that is compatible with OpenID Connect Discovery while being applicable to a wider set of OAuth 2.0 use cases.  This is intentionally parallel to the way that "OAuth 2.0 Dynamic Client Registration Protocol" generalized the dynamic client registration mechanisms defined by "OpenID Connect Dynamic Client Registration 1.0" in a way that is compatible with it.

This metadata can be communicated either in a self-asserted fashion by the server origin via HTTPS or as a set of signed metadata values represented as claims in a JSON Web Token (JWT).  In the JWT case, the issuer is vouching for the validity of the data about the authorization server.


Fields are 
- **issuer**: REQUIRED
- **authorization_endpoint**: This is REQUIRED unless no grant types are supported that use the authorization endpoint.
- **token_endpoint**: This is REQUIRED unless only the implicit grant type is supported.
- **jwks_uri**
- **registration_endpoint**: OPTIONAL
- **scopes_supported**: RECOMMENDED.  JSON array. Servers MAY choose not to advertise some supported scope values even when this parameter is used.
- **response_types_supported**: REQUIRED. 
- **response_modes_supported**: OPTIONAL. If omitted, the default is "query", "fragment".
- **grant_types_supported**: OPTIONAL. If omitted, the default value is "authorization_code", "implicit". 
- **token_endpoint_auth_methods_supported**: OPTIONAL. If omitted, the default is "client_secret_basic"
- **token_endpoint_auth_signing_alg_values_supported**: OPTIONAL. No default algorithms are implied if this entry is omitted.  Servers SHOULD support "RS256".  The value "none" MUST NOT be used.
- **service_documentation**: URL of a page containing human-readable information that developers might want or need to know when using the authorization server.
- **ui_locales_supported**: OPTIONAL.  Languages and scripts supported for the user interface, represented as a JSON array of language tag values
- **op_policy_uri**: OPTIONAL.  URL that the authorization server provides to the person registering the client to read about the authorization server's requirements on how the client can use the data provided by the authorization server.
- **op_tos_uri**: OPTIONAL.  URL that the authorization server provides to the person registering the client to read about the authorization server's terms of service.
- **revocation_endpoint**: OPTIONAL. 
- **revocation_endpoint_auth_methods_supported**: OPTIONAL. If omitted, the default is "client_secret_basic"
- **revocation_endpoint_auth_signing_alg_values_supported**: OPTIONAL
- **introspection_endpoint**: OPTIONAL.
- **introspection_endpoint_auth_methods_supported**: OPTIONAL
- **introspection_endpoint_auth_signing_alg_values_supported**: OPTIONAL 
- **code_challenge_methods_supported**: OPTIONAL. JSON array containing a list of Proof Key for Code Exchange (PKCE) code challenge methods supported by this authorization server.


#### [2.1](https://www.rfc-editor.org/rfc/rfc8414#section-2.1).  Signed Authorization Server Metadata
In addition to JSON elements, metadata values MAY also be provided as a "signed_metadata" value, which is a JSON Web Token (JWT) that asserts metadata values about the authorization server as a bundle.
Signed metadata is included in the authorization server metadata JSON object using this OPTIONAL member:
- **signed_metadata**

### [3](https://www.rfc-editor.org/rfc/rfc8414#section-3).  Obtaining Authorization Server Metadata
By default, the well-known URI string used is "/.well-known/oauth-authorization-server".  This path MUST use the "https" scheme.
Some OAuth applications will choose to use the well-known URI suffix "openid-configuration".  As described in Section 5, despite the identifier "/.well-known/openid-configuration", appearing to be OpenID specific, its usage in this specification is actually referring to a general OAuth 2.0 feature that is not specific to OpenID Connect.