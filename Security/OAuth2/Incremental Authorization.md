[IETF](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz)

### Abstract
OAuth 2.0 authorization requests that include every scope the client might ever need can result in over-scoped authorization and a sub- optimal end-user consent experience.  This specification enhances the OAuth 2.0 authorization protocol by adding incremental authorization, the ability to request specific authorization scopes as needed, when they're needed, removing the requirement to request every possible scope that might be needed upfront.


### [4](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz#section-4).  Incremental Auth for Confidential Clients
For confidential clients, such as web servers that can keep secrets, the authorization endpoint SHOULD treat scopes that the user already granted differently on the consent user interface.  Typically such scopes are hidden for new authorization requests, or at least there is an indication that the user already approved them.
The client indicates they wish the new authorization grant to include previously granted scopes by sending the following additional parameter in the OAuth 2.0 Authorization Request.
- **include_granted_scopes**:  OPTIONAL.  Either "true" or "false".  When "true", the authorization server SHOULD include previously granted scopes for this client in the new authorization grant.

### [5](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz#section-5).  Incremental Auth for Public Clients
Unlike with confidential clients, it is NOT RECOMMEND to automatically approve OAuth requests for public clients without user consent, thus authorization grants shouldn't contain previously authorized scopes in the manner described above for confidential clients.
Public clients (and confidential clients using this technique) should instead track the scopes for every authorization grant, and only request yet to be granted scopes during incremental authorization. In the past, this would result in multiple discrete authorization grants that would need to be tracked.  To enable incrementing a single authorization grant for public clients, the client supplies their existing refresh token during the authorization code exchange, and receives new authorization tokens with the scope of the previous and current authorization grants.

- **existing_grant**:  OPTIONAL.  The refresh token from the existing authorization grant.

### [6](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz#section-6).  Usability Considerations
#### [6.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz#section-6.1).  Handling Denials
In the case of incremental authorization, clients may already have a valid authorization and receive a denial for an incremental authorization request. Clients should SHOULD handle such errors gracefully and not discard any existing authorization grants if the user denies an incremental authorization request. Clients SHOULD NOT immediately request the same incremental authorization again, as this may result in an infinite denial loop (and the end-user feeling badgered).

#### [6.2](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz#section-6.2).  Handling Scope Reductions
A successful response may not always include all the scope that was asked for, a fact indicated by the "scope" response parameter when it happens.
So it's a good practice for clients to retain the current scope of the grant, update it during authorization, incremental authorization and token refreshes, and take action at any time based on the current scope by presenting an incremental authorization if a non-present scope is needed.

### [7](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz#section-7).  Alternative Approaches
#### [7.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz#section-7.1).  Alternative for Public Clients
It is possible for OAuth clients to maintain multiple authorizations per user for feature-specific scopes without needing the feature documented in this specification.  For example, a public client (such as a mobile app) could maintain an authorization for the contacts and one for calendar, and store them separately.

#### [7.2](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz#section-7.2).  Alternative for Confidential Clients
An alternative incremental auth design for confidential clients is to ask for authorization scopes as they are needed and keep a running record of all granted scopes. In this way each incremental authorization request would include all scopes granted so far, plus the new scope needed.

### [8](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz#section-8).  Privacy Considerations
#### [8.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz#section-8.1).  Requesting Authorization In Context
The goal of incremental authorization is to enhance end-user privacy by allowing clients to request only the authorization scopes needed in the context of a particular user action, rather than asking for ever possible scope upfront.


#### [8.2](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz#section-8.2).  Preventing Overbroad Authorization Requests 
When this specification is implemented, clients should have no technical reason to make overbroad authorization requests (i.e. requesting every possible scope they might ever need, rather than ones related to the user's current activity).  To improve privacy, it is therefore RECOMMENDED for authorization servers to limit the authorization scope that can be requested in a single authorization to what would reasonably be needed by a single feature. The authorization server MAY deny such authorization requests with the following error code. - **overbroad_scope**: The scope of the request is considered overbroad by the authorization server.  Consult the documentation of your authorization server to determine acceptable scope combinations.

#### [8.3](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz#section-8.3).  Authorization Correlation
ncremental authorization is designed for use-cases where it's the same user authorizing each request, and thus all incremental authorization grants are correlated to that one user (by being merged into a single authorization grant).  For applications where users may wish to connect different user accounts for different features (e.g. contacts from one account, and calendar from another) it is RECOMMENDED to instead allow multiple unrelated authorizations.

#### [8.4](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz#section-8.4).  Previously Granted Scopes
When the authorization server displays the list of scopes on page and prompts the user to consent to sharing access, users may assume that the displayed list of scopes on such a page is the full and complete list being granted to the application.  It may be desirable for such a consent page to list previously granted scopes, provided that the client is confidential, or one that cannot be impersonated.

### [9](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-incremental-authz#section-9).  Discovery Metadata
Add this field to the metadate
- **incremental_authz_types_supported**: OPTIONAL.  JSON array of OAuth 2.0 client types that are supported for incremental authorization. The possible types are "confidential", and "public".

