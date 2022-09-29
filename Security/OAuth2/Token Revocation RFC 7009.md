[RFC](https://www.rfc-editor.org/rfc/rfc7009)

### [1](https://www.rfc-editor.org/rfc/rfc7009#section-1).  Introduction
The OAuth 2.0 core specification defines several ways for a client to obtain refresh and access tokens.  This specification supplements the core specification with a mechanism to revoke both types of tokens.  A token is a string representing an authorization grant issued by the resource owner to the client.  A revocation request will invalidate the actual token and, if applicable, other tokens based on the same authorization grant and the authorization grant itself.
Token revocation prevents abuse of abandoned tokens and facilitates a better end-user experience since invalidated authorization grants will no longer turn up in a list of authorization grants the authorization server might present to the end-user.

### [2](https://www.rfc-editor.org/rfc/rfc7009#section-2).  Token Revocation
Implementations MUST support the revocation of refresh tokens and SHOULD support the revocation of access tokens (see Implementation Note).

#### [2.1](https://www.rfc-editor.org/rfc/rfc7009#section-2.1).  Revocation Request
- **token**: REQUIRED
- **token_type_hint**: OPTIONAL.  A hint about the type of the token submitted for revocation.  Clients MAY pass this parameter in order to help the authorization server to optimize the token lookup.  If the server is unable to locate the token using the given hint, it MUST extend its search across all of its supported token types.  An authorization server MAY ignore this parameter, particularly if it is able to detect the token type automatically.  This specification defines two such values:
	- **access_token**
	- **refresh_token**

- The authorization server first validates the client credentials (in case of a confidential client) and then verifies whether the token was issued to the client making the revocation request.
- When revocation happens, The invalidation takes place immediately, and the token cannot be
   used again after the revocation.
- Depending on the authorization server's revocation policy, the revocation of a particular token may cause the revocation of related tokens and the underlying authorization grant.  If the particular token is a refresh token and the authorization server supports the revocation of access tokens, then the authorization server SHOULD also invalidate all access tokens based on the same authorization grant. If the token passed to the request is an access token, the server MAY revoke the respective refresh token as well.
- A client compliant must be prepared to handle unexpected token invalidation at any time

#### [2.2](https://www.rfc-editor.org/rfc/rfc7009#section-2.2).  Revocation Response
The authorization server responds with HTTP status code 200 if the token has been revoked successfully or if the client submitted an invalid token.

##### [2.2.1](https://www.rfc-editor.org/rfc/rfc7009#section-2.2.1).  Error Response
Check the RFC
