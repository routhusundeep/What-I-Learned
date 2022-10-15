[IETF](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-step-up-authn-challenge)
#oauth

### Abstract 
It is not uncommon for resource servers to require different authentication strengths or freshness according to the characteristics of a request. This document introduces a mechanism for a resource server to signal to a client that the authentication event associated with the access token of the current request doesn't meet its authentication requirements and specify how to meet them.

### Introduction
Read this [blog](https://auth0.com/blog/what-is-step-up-authentication-when-to-use-it/) to understand step-up.

### [2](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-step-up-authn-challenge#section-2).  Protocol Overview
The scenario assumes that, before the sequence described below takes place, the client already obtained an access token for the protected resource.
  
    +----------+                                +--------------+
    |          |                                |              |
    |          |-----(1) resource request------>|              |
    |          |                                |              |
    |          |<-------(2) challenge ----------|   Resource   |
    |          |                                |    Server    |
    |          |                                |              |
    |          |-----(5) resource request ----->|              |
    |          |                                |              |
    |          |<---(6) protected resource -----|              |
    |          |                                +--------------+
    |  Client  |
    |          |
    |          |                                +---------------+
    |          |                                |               |
    |          |---(3) authorization request--->|               |
    |          |                                |               |
    |          |<-------------...-------------->| Authorization |
    |          |                                |     Server    |
    |          |<------ (4) access token -------|               |
    |          |                                |               |
    +----------+                                +---------------+

1. Requests Resource
2. RS determines that the auth strenght is insufficient, and returns a challenge (using a combination of **acr_values** and **max_age**) which should be completed to access the resource.
3. Client sends an auth request with acr_values and max_age
4. After client completes the challenge, he will get the access token
5. Requests resource with the new access token
6. Gets the resource

### [3](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-step-up-authn-challenge#section-3).  Authentication Requirements Challenge
This specification introduces a new error code value for the error parameter
- **insufficient_user_authentication**:  The authentication event associated with the access token presented with the request doesn't meet the authentication requirements of the protected resource.

Furthermore, this specification defines additional WWW-Authenticate auth-param values to convey the authentication requirements back to the client.
- **acr_values**:  A space-separated string listing the authentication context class reference values, in order of preference, one of which the protected resource requires for the authentication event associated with the access token.
- **max_age**:  Indicates the allowable elapsed time in seconds since the last active authentication event associated with the access token.

### [4](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-step-up-authn-challenge#section-4).  Authorization Request
A client receiving an authorization error from the resource server carrying the error code insufficient_user_authentication MAY parse the WWW-Authenticate header for acr_values and max_age and use them, if present, in a request to the authorization server to obtain a new access token complying with the corresponding requirements.  Both acr_values and max_age authorization request parameters are OPTIONAL

### [5](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-step-up-authn-challenge#section-5).  Authorization Response
AS should keep the following claims in the access token, whether they should be kept in the ID token or not is left to the implementation
- **acr**: The authorization performed by the AS, the authorization server SHOULD include in the acr claim a value reflecting the authentication level of the current session.
- **auth_time**: When was this authorization performed

If the authorization fails, AS can return the error code
- unmet_authentication_requirements

### [6](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-step-up-authn-challenge#section-6).  Authentication Information Conveyed via Access Token
- RS can read the claims in JWT
- RS can request AS to introspect the access token

