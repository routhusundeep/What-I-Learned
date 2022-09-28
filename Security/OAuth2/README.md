This is not my own, I just copied it from [https://oauth.net/2/](https://oauth.net/2/)

### OAuth 2.0

-   [OAuth 2.0 Framework](https://tools.ietf.org/html/rfc6749) - RFC 6749
    -   [Access Tokens](https://oauth.net/2/access-tokens/)
    -   [Refresh Tokens](https://oauth.net/2/refresh-tokens/)
    -   [OAuth Scope](https://oauth.net/2/scope/)
-   [OAuth Grant Types](https://oauth.net/2/grant-types/)
    -   [Authorization Code](https://oauth.net/2/grant-types/authorization-code/)
    -   [PKCE](https://oauth.net/2/pkce/)
    -   [Client Credentials](https://oauth.net/2/grant-types/client-credentials/)
    -   [Device Code](https://oauth.net/2/grant-types/device-code/)
    -   [Refresh Token](https://oauth.net/2/grant-types/refresh-token/)
    -   Legacy: [Implicit Flow](https://oauth.net/2/grant-types/implicit/)
    -   Legacy: [Password Grant](https://oauth.net/2/grant-types/password/)
-   [Client Types - Confidential and Public Applications](https://oauth.net/2/client-types/)
-   [Client Authentication](https://oauth.net/2/client-authentication/)
-   [Bearer Tokens](https://oauth.net/2/bearer-tokens/) - RFC 6750
-   [Threat Model and Security Considerations](https://oauth.net/2/security-considerations/) - RFC 6819
-   [OAuth Security Best Current Practice](https://oauth.net/2/oauth-best-practice/)
-   [ID Tokens vs Access Tokens](https://oauth.net/id-tokens-vs-access-tokens/)

#### Mobile and Other Devices

-   [Native Apps](https://oauth.net/2/native-apps/) - Recommendations for using OAuth with native apps
-   [Browser-Based Apps](https://oauth.net/2/browser-based-apps/) - Recommendations for using OAuth with browser-based apps (e.g. an SPA)
-   [Device Authorization Grant](https://oauth.net/2/device-flow/) - OAuth for devices with no browser or no keyboard

#### Token and Token Management

-   [JWT Profile for Access Tokens](https://oauth.net/2/jwt-access-tokens/) - RFC 9068, a standard for structured access tokens
-   [Token Introspection](https://oauth.net/2/token-introspection/) - RFC 7662, to determine the active state and meta-information of a token
-   [Token Revocation](https://oauth.net/2/token-revocation/) - RFC 7009, to signal that a previously obtained token is no longer needed
-   [JSON Web Token](https://oauth.net/2/jwt/) - RFC 7519
-   [Token Exchange](https://oauth.net/2/token-exchange/) - RFC 8693

#### Discovery and Registration

-   [Authorization Server Metadata](https://oauth.net/2/authorization-server-metadata/) - RFC 8414, for clients to discover OAuth endpoints and authorization server capabilities
-   [Dynamic Client Registration](https://oauth.net/2/dynamic-client-registration/) - RFC 7591, to programmatically register OAuth clients
-   [Dynamic Client Registration Management](https://oauth.net/2/dynamic-client-management/) - Experimental RFC 7592, for updating and managing dynamically registered OAuth clients

#### High Security OAuth

These specs are used to add additional security properties on top of OAuth 2.0.

-   [Pushed Authorization Requests (PAR)](https://oauth.net/2/pushed-authorization-requests/) - RFC 9126
-   [Demonstration of Proof of Possession (DPoP)](https://oauth.net/2/dpop/)
-   [Mutual TLS](https://oauth.net/2/mtls/) - RFC 8705
-   [Private Key JWT](https://oauth.net/private-key-jwt/) - (RFC 7521, RFC 7521, OpenID)
-   [FAPI](https://oauth.net/fapi/)

### Experimental and Draft Specs

The specs below are either experimental or in draft status and are still active working group items. They will likely change before they are finalized as RFCs or BCPs.

-   [Rich Authorization Requests (RAR)](https://oauth.net/2/rich-authorization-requests/)
-   [Incremental Authorization](https://tools.ietf.org/html/draft-ietf-oauth-incremental-authz)
-   [Step-up Authentication Challenge](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-step-up-authn-challenge)
-   [All OAuth Working Group Documents](https://oauth.net/specs/)

### Additional Extensions

-   [OAuth Extension Parameter Registry](https://www.iana.org/assignments/oauth-parameters/oauth-parameters.xhtml)
-   [OAuth Assertions Framework](http://tools.ietf.org/html/rfc7521) - RFC 7521
-   [SAML2 Bearer Assertion](http://tools.ietf.org/html/rfc7522) - RFC 7522, for integrating with existing identity systems
-   [JWT Bearer Assertion](http://tools.ietf.org/html/rfc7523) - RFC 7523
-   [Authorization Server Issuer Identification](http://tools.ietf.org/html/rfc9207) - RFC 9207, indicates the authorization server identifier in the authorization response

### Related Work from Other Communities

-   [FAPI](https://oauth.net/fapi/) (OpenID Foundation)
-   [WebAuthn - Web Authentication](https://oauth.net/webauthn/)
-   [Signing HTTP Messages](https://oauth.net/http-signatures/) - A generic HTTP message signing spec

### Community Resources

-   [OAuth 2.0 Simplified](https://aaronparecki.com/oauth-2-simplified/)
-   [Books about OAuth](https://oauth.net/books/)
    -   [OAuth 2.0 Simplified](https://oauth2simplified.com/) by Aaron Parecki
    -   [OAuth 2 in Action](https://www.amazon.com/OAuth-2-Action-Justin-Richer/dp/161729327X/?tag=oauthnet-20) by Justin Richer and Antonio Sanso
    -   [Mastering OAuth 2.0](https://www.amazon.com/Mastering-OAuth-2-0-Charles-Bihis/dp/1784395404?tag=oauthnet-20) by Charles Bihis
    -   [OAuth 2.0 Cookbook](https://www.amazon.com/dp/178829596X?tag=oauthnet-20) by Adolfo Eloy Nascimento
-   [The Nuts and Bolts of OAuth](https://oauth2simplified.com/course) - video course by Aaron Parecki

### Protocols Built on OAuth 2.0

-   [OpenID Connect](https://openid.net/connect/) (OpenID Foundation)
-   [UMA 2.0](https://docs.kantarainitiative.org/uma/wg/rec-oauth-uma-grant-2.0.html) (Kantara)
-   [IndieAuth](https://indieauth.spec.indieweb.org/) (W3C)

### Code and Services

-   [OAuth 2.0 Code and Services](https://oauth.net/code/)

### OAuth 2.1

-   [OAuth 2.1](https://oauth.net/2.1/) - An in-progress update to consolidate and simplify OAuth 2.0
-   [It's Time for OAuth 2.1](https://aaronparecki.com/2019/12/12/21/its-time-for-oauth-2-dot-1) (by Aaron Parecki)

### Legacy

-   [OAuth 1.0 and 1.0a](https://oauth.net/1/)