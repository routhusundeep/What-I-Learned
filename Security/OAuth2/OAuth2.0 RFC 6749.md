[RFC](https://www.rfc-editor.org/rfc/rfc6749)

### Abstract
The OAuth 2.0 authorization framework enables a third-party application to obtain limited access to an HTTP service, either on behalf of a resource owner by orchestrating an approval interaction between the resource owner and the HTTP service, or by allowing the third-party application to obtain access on its own behalf.


### Introduction
In traditional client-server model, drawbacks are
- Password should be shared with the third-party application
- Servers should support password authentication
- Overly broad access is given to the third party
- There is no fine-grained revocation
- Comprimise of one third party leads to total comprimise

### [1.1](https://www.rfc-editor.org/rfc/rfc6749#section-1.1). Roles
OAuth defines four roles:
- **Resource owner**
An entity capable of granting access to a protected resource. When the resource owner is a person, it is referred to as an end-user.
- **Resource server**
The server hosting the protected resources, capable of accepting and responding to protected resource requests using access tokens.
- **Client**
An application making protected resource requests on behalf of the resource owner and with its authorization. 
- **Authorization server**
The server issuing access tokens to the client after successfully authenticating the resource owner and obtaining authorization.

### [1.2](https://www.rfc-editor.org/rfc/rfc6749#section-1.2). Protocol Flow

     +--------+                               +---------------+
     |        |--(A)- Authorization Request ->|   Resource    |
     |        |                               |     Owner     |
     |        |<-(B)-- Authorization Grant ---|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(C)-- Authorization Grant -->| Authorization |
     | Client |                               |     Server    |
     |        |<-(D)----- Access Token -------|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(E)----- Access Token ------>|    Resource   |
     |        |                               |     Server    |
     |        |<-(F)--- Protected Resource ---|               |
     +--------+                               +---------------+

### [1.3](https://www.rfc-editor.org/rfc/rfc6749#section-1.3). Authorization Request
An authorization grant is a credential representing the resource
owner's authorization (to access its protected resources) used by the
client to obtain an access token.
- Authorization Code
- Implicit
- Resource Owner Password Credendials
- Client Credentials

### [1.4](https://www.rfc-editor.org/rfc/rfc6749#section-1.4). Access Token
Access tokens are credentials used to access protected resources.  An access token is a string representing an authorization issued to the client.  The string is usually opaque to the client.  Tokens represent specific scopes and durations of access, granted by the resource owner, and enforced by the resource server and authorization server.

### [1.5](https://www.rfc-editor.org/rfc/rfc6749#section-1.5). Refresh Token
Refresh tokens are credentials used to obtain access tokens.  Refresh tokens are issued to the client by the authorization server and are used to obtain a new access token when the current access token becomes invalid or expires, or to obtain additional access tokens with identical or narrower scope (access tokens may have a shorter lifetime and fewer permissions than authorized by the resource owner).

	  +--------+                                           +---------------+
	  |        |--(A)------- Authorization Grant --------->|               |
	  |        |                                           |               |
	  |        |<-(B)----------- Access Token -------------|               |
	  |        |               & Refresh Token             |               |
	  |        |                                           |               |
	  |        |                            +----------+   |               |
	  |        |--(C)---- Access Token ---->|          |   |               |
	  |        |                            |          |   |               |
	  |        |<-(D)- Protected Resource --| Resource |   | Authorization |
	  | Client |                            |  Server  |   |     Server    |
	  |        |--(E)---- Access Token ---->|          |   |               |
	  |        |                            |          |   |               |
	  |        |<-(F)- Invalid Token Error -|          |   |               |
	  |        |                            +----------+   |               |
	  |        |                                           |               |
	  |        |--(G)----------- Refresh Token ----------->|               |
	  |        |                                           |               |
	  |        |<-(H)----------- Access Token -------------|               |
	  +--------+           & Optional Refresh Token        +---------------+



### [2](https://www.rfc-editor.org/rfc/rfc6749#section-2). Client Registration
Before initiating the protocol, the client registers with the authorization server.
When registering a client, the client developer SHALL:
- specify the client type as described in [Section 2.1](https://www.rfc-editor.org/rfc/rfc6749#section-2.1),
- provide its client redirection URIs as described in [Section 3.1.2](https://www.rfc-editor.org/rfc/rfc6749#section-3.1.2), and
- include any other information required by the authorization server (e.g., application name, website, description, logo image, the acceptance of legal terms).

### [2.1](https://www.rfc-editor.org/rfc/rfc6749#section-2.1). Client Types
- **Confidential**
Clients capable of maintaining the confidentiality of their credentials or capable of secure client authentication using other means.
- **Public**
Clients incapable of maintaining the confidentiality of their credentials

This specification has been designed around the following client profiles:
- **Web Application**
A web application is a confidential client running on a web server.  Resource owners access the client via an HTML user interface rendered in a user-agent on the device used by the resource owner.  The client credentials as well as any access token issued to the client are stored on the web server and are not exposed to or accessible by the resource owner.
- **User-agent-based application**
A user-agent-based application is a public client in which the client code is downloaded from a web server and executes within a user-agent on the device used by the resource owner.  Protocol data and credentials are easily accessible (and often visible) to the resource owner.
- **Native Application**
A native application is a public client installed and executed on the device used by the resource owner.  Protocol data and credentials are accessible to the resource owner.  It is assumed that any client authentication credentials included in the application can be extracted.  On the other hand, dynamically issued credentials such as access tokens or refresh tokens can receive an acceptable level of protection.

### [2.2](https://www.rfc-editor.org/rfc/rfc6749#section-2.2). Client Identifier
The authorization server issues the registered client a client identifier -- a unique string representing the registration information provided by the client.  The client identifier is not a secret; it is exposed to the resource owner and MUST NOT be used alone for client authentication.  The client identifier is unique to the authorization server.

### [2.3](https://www.rfc-editor.org/rfc/rfc6749#section-2.3). Client Authentication
If the client type is confidential, the client and authorization server establish a client authentication method suitable for the security requirements of the authorization server.  The authorization server MAY accept any form of client authentication meeting its security requirements. Confidential clients are typically issued (or establish) a set of client credentials used for authenticating with the authorization server (e.g., password, public/private key pair).

### [2.3.1](https://www.rfc-editor.org/rfc/rfc6749#section-2.3.1). Client Password
Clients in possession of a client password MAY use the HTTP Basic authentication scheme as defined in [RFC2617] to authenticate with the authorization server.
Alternatively, the authorization server MAY support including the client credentials in the request-body using the following parameters: 
- client_id REQUIRED.  The client identifier issued to the client during the registration process. 
- client_secret REQUIRED.  The client secret.  The client MAY omit the parameter if the client secret is an empty string.

Including the client credentials in the request-body using the two parameters is NOT RECOMMENDED and SHOULD be limited to clients unable to directly utilize the HTTP Basic authentication scheme

### [2.4](https://www.rfc-editor.org/rfc/rfc6749#section-2.4).  Unregistered Clients
This specification does not exclude the use of unregistered clients. However, the use of such clients is beyond the scope of this specification and requires additional security analysis and review of its interoperability impact.

## [3](https://www.rfc-editor.org/rfc/rfc6749#section-3).  Protocol Endpoints
The authorization process utilizes two authorization server endpoints (HTTP resources): 
- Authorization endpoint - used by the client to obtain authorization from the resource owner via user-agent redirection. 
- Token endpoint - used by the client to exchange an authorization grant for an access token, typically with client authentication.
As well as one client endpoint: 
- Redirection endpoint - used by the authorization server to return responses containing authorization credentials to the client via the resource owner user-agent. 
Not every authorization grant type utilizes both endpoints. Extension grant types MAY define additional endpoints as needed.

### [3.1](https://www.rfc-editor.org/rfc/rfc6749#section-3.1).  Authorization Endpoint
The authorization endpoint is used to interact with the resource owner and obtain an authorization grant.  The authorization server MUST first verify the identity of the resource owner.
The authorization server MUST support the use of the HTTP "GET" method for the authorization endpoint, and MAY support the use of the "POST" method as well.
Parameters sent without a value MUST be treated as if they were omitted from the request.  The authorization server MUST ignore unrecognized request parameters.  Request and response parameters MUST NOT be included more than once.

#### [3.1.1](https://www.rfc-editor.org/rfc/rfc6749#section-3.1.1).  Response Type
The authorization endpoint is used by the authorization code grant type and implicit grant type flows.
Parameter `resonse_type` is REQUIRED.  The value MUST be one of "code" for requesting an authorization code, "token" for requesting an access token (implicit grant), or a registered extension value.
If an authorization request is missing the "response_type" parameter, or if the response type is not understood, the authorization server MUST return an error response

#### [3.1.2](https://www.rfc-editor.org/rfc/rfc6749#section-3.1.2).  Redirection Endpoint
After completing its interaction with the resource owner, the authorization server directs the resource owner's user-agent back to the client.  The authorization server redirects the user-agent to the client's redirection endpoint previously established with the authorization server during the client registration process or when making the authorization request. The redirection endpoint URI MUST be an absolute URI.  The endpoint URI MAY include an "application/x-www-form-urlencoded" formatted query component, which MUST be retained when adding additional query parameters.  The endpoint URI MUST NOT include a fragment component.

##### [3.1.2.1](https://www.rfc-editor.org/rfc/rfc6749#section-3.1.2.1).  Endpoint Request Confidentiality
The redirection endpoint SHOULD require the use of TLS when the requested response type is "code" or "token", or when the redirection request will result in the transmission of sensitive credentials over an open network. If TLS is not available, the authorization server SHOULD warn the resource owner about the insecure endpoint prior to redirection

##### [3.1.2.2](https://www.rfc-editor.org/rfc/rfc6749#section-3.1.2.2).  Registration Requirements
The authorization server MUST require the following clients to register their redirection endpoint:
- Public clients.
- Confidential clients utilizing the implicit grant type.

The authorization server SHOULD require all clients to register their redirection endpoint prior to utilizing the authorization endpoint.

The authorization server SHOULD require the client to provide the complete redirection URI (the client MAY use the "state" request parameter to achieve per-request customization).  If requiring the registration of the complete redirection URI is not possible, the authorization server SHOULD require the registration of the URI scheme, authority, and path (allowing the client to dynamically vary only the query component of the redirection URI when requesting
authorization).
The authorization server MAY allow the client to register multiple redirection endpoints.
Lack of a redirection URI registration requirement can enable an attacker to use the authorization endpoint as an open redirector

##### [3.1.2.3](https://www.rfc-editor.org/rfc/rfc6749#section-3.1.2.3).  Dynamic Configuration
If multiple redirection URIs have been registered, if only part of the redirection URI has been registered, or if no redirection URI has been registered, the client MUST include a redirection URI with the authorization request using the "redirect_uri" request parameter. When a redirection URI is included in an authorization request, the authorization server MUST compare and match the value received against at least one of the registered redirection URIs, if any redirection URIs were registered.  If the client registration included the full redirection URI, the authorization server MUST compare the two URIs using simple string comparison.

##### [3.1.2.4](https://www.rfc-editor.org/rfc/rfc6749#section-3.1.2.4).  Invalid Endpoint
If an authorization request fails validation due to a missing, invalid, or mismatching redirection URI, the authorization server SHOULD inform the resource owner of the error and MUST NOT automatically redirect the user-agent to the invalid redirection URI.

##### [3.1.2.5](https://www.rfc-editor.org/rfc/rfc6749#section-3.1.2.5).  Endpoint Content
The redirection request to the client's endpoint typically results in an HTML document response, processed by the user-agent.  If the HTML response is served directly as the result of the redirection request, any script included in the HTML document will execute with full access to the redirection URI and the credentials it contains. The client SHOULD NOT include any third-party scripts (e.g., third-party analytics, social plug-ins, ad networks) in the redirection endpoint response.  Instead, it SHOULD extract the credentials from the URI and redirect the user-agent again to another endpoint without exposing the credentials (in the URI or elsewhere).  If third-party scripts are included, the client MUST ensure that its own scripts (used to extract and remove the credentials from the URI) will execute first.

### [3.2](https://www.rfc-editor.org/rfc/rfc6749#section-3.2).  Token Endpoint
The token endpoint is used by the client to obtain an access token by presenting its authorization grant or refresh token.
The endpoint URI MAY include a query component, which MUST be retained when adding additional query parameters. The endpoint URI MUST NOT include a fragment component.
The client MUST use the HTTP "POST" method when making access token requests.
Parameters sent without a value MUST be treated as if they were omitted from the request.  The authorization server MUST ignore unrecognized request parameters. Request and response parameters MUST NOT be included more than once.


##### [3.2.1](https://www.rfc-editor.org/rfc/rfc6749#section-3.2.1).  Client Authentication
- Enforcing the binding of refresh tokens and authorization codes to the client they were issued to.  Client authentication is critical when an authorization code is transmitted to the redirection endpoint over an insecure channel or when the redirection URI has not been registered in full.
- Recovering from a compromised client by disabling the client or changing its credentials, thus preventing an attacker from abusing stolen refresh tokens. Changing a single set of client credentials is significantly faster than revoking an entire set of refresh tokens.
- Implementing authentication management best practices, which require periodic credential rotation.  Rotation of an entire set of refresh tokens can be challenging, while rotation of a single set of client credentials is significantly easier.

A client MAY use the "client_id" request parameter to identify itself when sending requests to the token endpoint.  In the "authorization_code" "grant_type" request to the token endpoint, an unauthenticated client MUST send its "client_id" to prevent itself from inadvertently accepting a code intended for a client with a different "client_id".

### [3.3](https://www.rfc-editor.org/rfc/rfc6749#section-3.3).  Access Token Scope
The authorization and token endpoints allow the client to specify the scope of the access request using the "scope" request parameter.
The value of the scope parameter is expressed as a list of space-delimited, case-sensitive strings.  The strings are defined by the authorization server.
The authorization server MAY fully or partially ignore the scope requested by the client, based on the authorization server policy or the resource owner's instructions.  If the issued access token scope is different from the one requested by the client, the authorization server MUST include the "scope" response parameter to inform the client of the actual scope granted.
If the client omits the scope parameter when requesting authorization, the authorization server MUST either process the request using a pre-defined default value or fail the request indicating an invalid scope.  The authorization server SHOULD document its scope requirements and default value (if defined).

### [4.1](https://www.rfc-editor.org/rfc/rfc6749#section-4.1).  Authorization Code Grant
	+----------+
	 | Resource |
	 |   Owner  |
	 |          |
	 +----------+
		  ^
		  |
		 (B)
	 +----|-----+          Client Identifier      +---------------+
	 |         -+----(A)-- & Redirection URI ---->|               |
	 |  User-   |                                 | Authorization |
	 |  Agent  -+----(B)-- User authenticates --->|     Server    |
	 |          |                                 |               |
	 |         -+----(C)-- Authorization Code ---<|               |
	 +-|----|---+                                 +---------------+
	   |    |                                         ^      v
	  (A)  (C)                                        |      |
	   |    |                                         |      |
	   ^    v                                         |      |
	 +---------+                                      |      |
	 |         |>---(D)-- Authorization Code ---------'      |
	 |  Client |          & Redirection URI                  |
	 |         |                                             |
	 |         |<---(E)----- Access Token -------------------'
	 +---------+       (w/ Optional Refresh Token)

#### [4.1.1](https://www.rfc-editor.org/rfc/rfc6749#section-4.1.1).  Authorization Request
 The following parameters may be present
 - **response_type**: REQUIRED.  Value MUST be set to "code".
- **client_id**: REQUIRED.  The client identifier as described in [Section 2.2](https://www.rfc-editor.org/rfc/rfc6749#section-2.2).
- **redirect_uri**: OPTIONAL.  As described in [Section 3.1.2](https://www.rfc-editor.org/rfc/rfc6749#section-3.1.2).
- **scope**:OPTIONAL.  The scope of the access request as described by [Section 3.3](https://www.rfc-editor.org/rfc/rfc6749#section-3.3).
- **state**: RECOMMENDED.  An opaque value used by the client to maintain state between the request and callback.  The authorization server includes this value when redirecting the user-agent back to the client.  The parameter SHOULD be used for preventing cross-site request forgery as described in [Section 10.12](https://www.rfc-editor.org/rfc/rfc6749#section-10.12).

#### [4.1.2](https://www.rfc-editor.org/rfc/rfc6749#section-4.1.2).  Authorization Response
- **code**: REQUIRED.  The authorization code generated by the authorization server.  The authorization code MUST expire shortly after it is issued to mitigate the risk of leaks.  A maximum authorization code lifetime of 10 minutes is RECOMMENDED.  The client MUST NOT use the authorization code more than once.  If an authorization code is used more than once, the authorization server MUST deny the request and SHOULD revoke (when possible) all tokens previously issued based on that authorization code.  The authorization code is bound to the client identifier and redirection URI.
- **state**: REQUIRED if the "state" parameter was present in the client authorization request.  The exact value received from the client.

##### [4.1.2.1](https://www.rfc-editor.org/rfc/rfc6749#section-4.1.2.1).  Error Response
- **error**: REQUIRED.  A single ASCII error code from the following:
	- **invalid_request**: The request is missing a required parameter, includes an invalid parameter value, includes a parameter more than once, or is otherwise malformed.
	- **unauthorized_client**: The client is not authorized to request an authorization code using this method. 
	- **access_denied**: The resource owner or authorization server denied the request.
	- **unsupported_response_type**: The authorization server does not support obtaining an authorization code using this method. 
	- **invalid_scope**: The requested scope is invalid, unknown, or malformed. 
	- **server_error**: The authorization server encountered an unexpected condition that prevented it from fulfilling the request. (This error code is needed because a 500 Internal Server Error HTTP status code cannot be returned to the client via an HTTP redirect.)
	- **temporarily_unavailable**: The authorization server is currently unable to handle the request due to a temporary overloading or maintenance of the server.  (This error code is needed because a 503 Service Unavailable HTTP status code cannot be returned to the client via an HTTP redirect.)
- **error_description**: OPTIONAL.  Human-readable ASCII text providing additional information, used to assist the client developer in understanding the error that occurred.
- **error_uri**: OPTIONAL.  A URI identifying a human-readable web page with information about the error, used to provide the client developer with additional information about the error.
- **state**: REQUIRED if a "state" parameter was present in the client authorization request.  The exact value received from the client.

#### [4.1.3](https://www.rfc-editor.org/rfc/rfc6749#section-4.1.3).  Access Token Request
- **grant_type**: REQUIRED.  Value MUST be set to "authorization_code". 
- **code**: REQUIRED.  The authorization code received from the authorization server.
- **redirect_uri**: REQUIRED, if the "redirect_uri" parameter was included in the authorization request, and their values MUST be identical. 
- **client_id**: REQUIRED, if the client is not authenticating with the authorization server
If the client type is confidential or the client was issued client credentials (or assigned other authentication requirements), the client MUST authenticate with the authorization server

The authorization server MUST:
- Require client authentication for confidential clients or for any client that was issued client credentials (or with other authentication requirements).
- Authenticate the client if client authentication is included.
- Ensure that the authorization code was issued to the authenticated confidential client, or if the client is public, ensure that the code was issued to "client_id" in the request.
- Verify that the authorization code is valid
- Ensure that the "redirect_uri" parameter is present if the "redirect_uri" parameter was included in the initial authorization request, and if included ensure that their values are identical.

#### [4.1.4](https://www.rfc-editor.org/rfc/rfc6749#section-4.1.4).  Access Token Response
Refer to "Successful Response" section

### [4.2](https://www.rfc-editor.org/rfc/rfc6749#section-4.2).  Implicit Grant
	+----------+
     | Resource |
     |  Owner   |
     |          |
     +----------+
          ^
          |
         (B)
     +----|-----+          Client Identifier     +---------------+
     |         -+----(A)-- & Redirection URI --->|               |
     |  User-   |                                | Authorization |
     |  Agent  -|----(B)-- User authenticates -->|     Server    |
     |          |                                |               |
     |          |<---(C)--- Redirection URI ----<|               |
     |          |          with Access Token     +---------------+
     |          |            in Fragment
     |          |                                +---------------+
     |          |----(D)--- Redirection URI ---->|   Web-Hosted  |
     |          |          without Fragment      |     Client    |
     |          |                                |    Resource   |
     |     (F)  |<---(E)------- Script ---------<|               |
     |          |                                +---------------+
     +-|--------+
       |    |
      (A)  (G) Access Token
       |    |
       ^    v
     +---------+
     |         |
     |  Client |
     |         |
     +---------+

#### [4.2.1](https://www.rfc-editor.org/rfc/rfc6749#section-4.2.1).  Authorization Request
- **response_type**: REQUIRED.  Value MUST be set to "token".
- **client_id**: REQUIRED.
- **redirect_uri**: OPTIONAL. 
- **scope**: OPTIONAL. 
- **state**: RECOMMENDED.

#### [4.2.2](https://www.rfc-editor.org/rfc/rfc6749#section-4.2.2).  Access Token Response
If the resource owner grants the access request, the authorization server issues an access token and delivers it to the client by adding the following parameters to the fragment component of the redirection URI
- **access_token**: REQUIRED.
- **token_type**: REQUIRED.
- **expires_in**: RECOMMENDED. 
- **scope**: OPTIONAL. 
- **state**: RECOMMENDED.
The authorization server MUST NOT issue a refresh token.

##### [4.2.2.1](https://www.rfc-editor.org/rfc/rfc6749#section-4.2.2.1).  Error Response
Refer the RFC

### [4.3](https://www.rfc-editor.org/rfc/rfc6749#section-4.3).  Resource Owner Password Credentials Grant
	+----------+
     | Resource |
     |  Owner   |
     |          |
     +----------+
          v
          |    Resource Owner
         (A) Password Credentials
          |
          v
     +---------+                                  +---------------+
     |         |>--(B)---- Resource Owner ------->|               |
     |         |         Password Credentials     | Authorization |
     | Client  |                                  |     Server    |
     |         |<--(C)---- Access Token ---------<|               |
     |         |    (w/ Optional Refresh Token)   |               |
     +---------+                                  +---------------+

The authorization server should take special care when enabling this grant type and only allow it when other flows are not viable. This grant type is suitable for clients capable of obtaining the resource owner's credentials (username and password, typically using an interactive form).  It is also used to migrate existing clients using direct authentication schemes such as HTTP Basic or Digest authentication to OAuth by converting the stored credentials to an access token.

#### [4.3.2](https://www.rfc-editor.org/rfc/rfc6749#section-4.3.2).  Access Token Request
- **grant_type**: REQUIRED. Value MUST be set to "password".
- **user_name**: REQUIRED.
- **password**: REQUIRED. 
- **scope**: OPTIONAL. 
The authorization server MUST: 
- require client authentication for confidential clients or for any client that was issued client credentials (or with other authentication requirements). 
- authenticate the client if client authentication is included.
- validate the resource owner password credentials using its existing password validation algorithm. 
Since this access token request utilizes the resource owner's password, the authorization server MUST protect the endpoint against brute force attacks (e.g., using rate-limitation or generating alerts).

#### [4.3.3](https://www.rfc-editor.org/rfc/rfc6749#section-4.3.3).  Access Token Response
	HTTP/1.1 200 OK
     Content-Type: application/json;charset=UTF-8
     Cache-Control: no-store
     Pragma: no-cache
     {
       "access_token":"2YotnFZFEjr1zCsicMWpAA",
       "token_type":"example",
       "expires_in":3600,
       "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
       "example_parameter":"example_value"
     }

### [4.4](https://www.rfc-editor.org/rfc/rfc6749#section-4.4).  Client Credentials Grant
	 +---------+                                  +---------------+
     |         |                                  |               |
     |         |>--(A)- Client Authentication --->| Authorization |
     | Client  |                                  |     Server    |
     |         |<--(B)---- Access Token ---------<|               |
     |         |                                  |               |
     +---------+                                  +---------------+

#### [4.4.1](https://www.rfc-editor.org/rfc/rfc6749#section-4.4.1).  Authorization Request and Response
Since the client authentication is used as the authorization grant, no additional authorization request is needed.

#### [4.4.2](https://www.rfc-editor.org/rfc/rfc6749#section-4.4.2).  Access Token Request
- **grant_type**: REQUIRED. Value MUST be set to "client_credentials".
- **scope**: REQUIRED.

#### [4.4.3](https://www.rfc-editor.org/rfc/rfc6749#section-4.4.3).  Access Token Response
	 HTTP/1.1 200 OK
     Content-Type: application/json;charset=UTF-8
     Cache-Control: no-store
     Pragma: no-cache
     {
       "access_token":"2YotnFZFEjr1zCsicMWpAA",
       "token_type":"example",
       "expires_in":3600,
       "example_parameter":"example_value"
     }


### [4.5](https://www.rfc-editor.org/rfc/rfc6749#section-4.5).  Extension Grants
The client uses an extension grant type by specifying the grant type using an absolute URI (defined by the authorization server) as the value of the "grant_type" parameter of the token endpoint, and by adding any additional parameters necessary.

	POST /token HTTP/1.1
     Host: server.example.com
     Content-Type: application/x-www-form-urlencoded
     
     grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Asaml2-
     bearer&assertion=PEFzc2VydGlvbiBJc3N1ZUluc3RhbnQ9IjIwMTEtMDU
     [...omitted for brevity...]aG5TdGF0ZW1lbnQ-PC9Bc3NlcnRpb24-

If the access token request is valid and authorized, the authorization server issues an access token and optional refresh token


## [5](https://www.rfc-editor.org/rfc/rfc6749#section-5).  Issuing an Access Token
#### [5.1](https://www.rfc-editor.org/rfc/rfc6749#section-5.1).  Successful Response
- **access_token**: REQUIRED.
- **token_type**: REQUIRED.
- **expires_in**: RECOMMENDED. 
- **refresh_token**: OPTIONAL.
- **scope**: OPTIONAL. 
The authorization server MUST include the HTTP "Cache-Control" response header field with a value of "no-store" in any response containing tokens, credentials, or other sensitive information, as well as the "Pragma" response header field with a value of "no-cache".

#### [5.2](https://www.rfc-editor.org/rfc/rfc6749#section-5.2).  Error Response
Refer the RFC

## [6](https://www.rfc-editor.org/rfc/rfc6749#section-6).  Refreshing an Access Token
- **grant_type**: REQUIRED.
- **refresh_token**: REQUIRED.
- **scope**: OPTIONAL, The requested scope MUST NOT include any scope not originally granted by the resource owner, and if omitted is treated as equal to the scope originally granted by the resource owner.

The authorization server MUST: 
- require client authentication for confidential clients or for any client that was issued client credentials (or with other authentication requirements), 
- authenticate the client if client authentication is included and ensure that the refresh token was issued to the authenticated client, and 
- validate the refresh token. 
If valid and authorized, the authorization server issues an access token.  If the request failed verification or is invalid, the authorization server returns an error response. 
The authorization server MAY issue a new refresh token, in which case the client MUST discard the old refresh token and replace it with the new refresh token.  The authorization server MAY revoke the old refresh token after issuing a new refresh token to the client.  If a new refresh token is issued, the refresh token scope MUST be identical to that of the refresh token included by the client in the request.


## [7](https://www.rfc-editor.org/rfc/rfc6749#section-7).  Accessing Protected Resources
The client accesses protected resources by presenting the access token to the resource server.  The resource server MUST validate the access token and ensure that it has not expired and that its scope covers the requested resource.

#### [7.1](https://www.rfc-editor.org/rfc/rfc6749#section-7.1).  Access Token Types
The access token type provides the client with the information required to successfully utilize the access token to make a protected resource request (along with type-specific attributes).
- Bearer [RFC6750](https://www.rfc-editor.org/rfc/rfc6750 ""The OAuth 2.0 Authorization Framework: Bearer Token Usage"")
- MAC [OAuth-HTTP-MAC](https://www.rfc-editor.org/rfc/rfc6749#ref-OAuth-HTTP-MAC)

#### [7.2](https://www.rfc-editor.org/rfc/rfc6749#section-7.2).  Error Response
Refer the RFC

## [8](https://www.rfc-editor.org/rfc/rfc6749#section-8).  Extensibility
Refer the spec for 
- [8.1](https://www.rfc-editor.org/rfc/rfc6749#section-8.1).  Defining Access Token Types
- [8.2](https://www.rfc-editor.org/rfc/rfc6749#section-8.2).  Defining New Endpoint Parameters
- [8.3](https://www.rfc-editor.org/rfc/rfc6749#section-8.3).  Defining New Authorization Grant Types
- [8.4](https://www.rfc-editor.org/rfc/rfc6749#section-8.4).  Defining New Authorization Endpoint Response Types
- [8.5](https://www.rfc-editor.org/rfc/rfc6749#section-8.5).  Defining Additional Error Codes

## [9](https://www.rfc-editor.org/rfc/rfc6749#section-9).  Native Applications
Native applications are clients installed and executed on the device used by the resource owner (i.e., desktop application, native mobile application).
The authorization endpoint requires interaction between the client and the resource owner's user-agent.  Native applications can invoke an external user-agent or embed a user-agent within the application. For example:
- External user-agent - the native application can capture the response from the authorization server using a redirection URI with a scheme registered with the operating system to invoke the client as the handler, which in turn makes the response available to the native application.
- Embedded user-agent - the native application obtains the response by directly communicating with the embedded user-agent by monitoring state changes emitted during the resource load, or accessing the user-agent's cookies storage.

When choosing between an external or embedded user-agent, developers should consider the following:
- An external user-agent may improve completion rate, as the resource owner may already have an active session with the authorization server, removing the need to re-authenticate.  It provides a familiar end-user experience and functionality.
- An embedded user-agent may offer improved usability, as it removes the need to switch context and open new windows.
- An embedded user-agent poses a security challenge because resource owners are authenticating in an unidentified window without access to the visual protections found in most external user-agents.  An embedded user-agent educates end-users to trust unidentified requests for authentication (making phishing attacks easier to execute).

When choosing between the implicit grant type and the authorization code grant type, the following should be considered:
- Native applications that use the authorization code grant type SHOULD do so without using client credentials, due to the native application's inability to keep client credentials confidential.
- When using the implicit grant type flow, a refresh token is not returned, which requires repeating the authorization process once the access token expires.

## [10](https://www.rfc-editor.org/rfc/rfc6749#section-10).  Security Considerations

HIGHLY ENCOURAGE YOU TO READ THE WHOLE THING. VERY INSIGHTFUL.