[IETF Tracker](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
#oauth

### [1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics#section-1).  Introduction
While OAuth is used in a variety of scenarios and different kinds of deployments, the following challenges can be observed:
- OAuth implementations are being attacked through known implementation weaknesses and anti-patterns.
- OAuth is being used in environments with higher security requirements than considered initially, such as Open Banking, eHealth, eGovernment, and Electronic Signatures.  Those use cases call for stricter guidelines and additional protection.
- OAuth is being used in much more dynamic setups than originally anticipated, creating new challenges with respect to security.
- Technology has changed.  For example, the way browsers treat fragments when redirecting requests has changed, and with it, the implicit grant's underlying security model.

### [2](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics#section-2).  Best Practices
#### [2.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics#section-2.1).  Protecting Redirect-Based Flows
- When comparing client redirect URIs against pre-registered URIs, authorization servers MUST utilize exact string matching except for port numbers in localhost redirection URIs of native apps
- Clients and AS MUST NOT expose URLs that forward the user's browser to arbitrary URIs obtained from a query parameter ("open redirector").
- Clients MUST prevent Cross-Site Request Forgery (CSRF).
	- May use PKCE
	- In OpenID Connect flows, the nonce parameter provides CSRF protection.  
	- Otherwise, one- time use CSRF tokens carried in the state parameter that are securely bound to the user agent MUST be used for CSRF protection
- When an OAuth client can interact with more than one authorization server, a defense against mix-up attacks is REQUIRED.  To this end, clients SHOULD
	- use the iss parameter, or
	- use an alternative countermeasure based on an iss value in the authorization response (such as the iss Claim in the ID Token

##### [2.1.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics#section-2.1.1).  Authorization Code Grant
Clients MUST prevent authorization code injection attacks
- Public clients MUST use PKCE
- For confidential clients, the use of PKCE [RFC7636] is RECOMMENDED, as it provides a strong protection against misuse and injection of authorization codes and, as a side-effect, prevents CSRF even in presence of strong attackers.
- clients MAY use the nonce parameter and the respective Claim in the ID Token instead.
- Authorization servers MUST support PKCE

#### [2.1.2](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics#section-2.1.2).  Implicit Grant
- The implicit grant (response type "token") and other response types causing the authorization server to issue access tokens in the authorization response are vulnerable to access token leakage and access token replay
- In order to avoid these issues, clients SHOULD NOT use the implicit grant (response type "token") or other response types issuing access tokens in the authorization response, unless access token injection in the authorization response is prevented and the aforementioned token leakage vectors are mitigated.

#### [2.2](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics#section-2.2).  Token Replay Prevention
Access Tokens and Refresh Tokens must be sender constrained. MUST Use TLS

#### [2.3](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics#section-2.3).  Access Token Privilege Restriction
- The privileges associated with an access token SHOULD be restricted to the minimum required for the particular application or use case.
- Additionally, access tokens SHOULD be restricted to certain resources and actions on resource servers or resources.

#### [2.4](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics#section-2.4).  Resource Owner Password Credentials Grant
The resource owner password credentials grant MUST NOT be used.

#### [2.5](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics#section-2.5).  Client Authentication
Authorization servers SHOULD use client authentication if possible. It is RECOMMENDED to use asymmetric (public-key based) methods for client authentication such as mTLS or private_key_jwt.

#### [2.6](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics#section-2.6).  Other Recommendations
The use of OAuth Metadata can help to improve the security of OAuth deployments:
- It ensures that security features and other new OAuth features can be enabled automatically by compliant software libraries.
- It reduces chances for misconfigurations, for example misconfigured endpoint URLs (that might belong to an attacker) or misconfigured security features.
- It can help to facilitate rotation of cryptographic keys and to ensure cryptographic agility.
- 