[RFC](https://www.rfc-editor.org/rfc/rfc7636)

# [1](https://www.rfc-editor.org/rfc/rfc7636#section-1).  Introduction
In this attack, the attacker intercepts the authorization code returned from the authorization endpoint within a communication path not protected by Transport Layer Security (TLS), such as inter-application communication within the client's operating system. Once the attacker has gained access to the authorization code, it can use it to obtain the access token.

	+~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~+
    | End Device (e.g., Smartphone)  |
    |                                |
    | +-------------+   +----------+ | (6) Access Token  +----------+
    | |Legitimate   |   | Malicious|<--------------------|          |
    | |OAuth 2.0 App|   | App      |-------------------->|          |
    | +-------------+   +----------+ | (5) Authorization |          |
    |        |    ^          ^       |        Grant      |          |
    |        |     \         |       |                   |          |
    |        |      \   (4)  |       |                   |          |
    |    (1) |       \  Authz|       |                   |          |
    |   Authz|        \ Code |       |                   |  Authz   |
    | Request|         \     |       |                   |  Server  |
    |        |          \    |       |                   |          |
    |        |           \   |       |                   |          |
    |        v            \  |       |                   |          |
    | +----------------------------+ |                   |          |
    | |                            | | (3) Authz Code    |          |
    | |     Operating System/      |<--------------------|          |
    | |         Browser            |-------------------->|          |
    | |                            | | (2) Authz Request |          |
    | +----------------------------+ |                   +----------+

   A number of pre-conditions need to hold for this attack to work:
   1. The attacker manages to register a malicious application on the client device and registers a custom URI scheme that is also used by another application.  The operating systems must allow a custom URI scheme to be registered by multiple applications.
   2. The OAuth 2.0 authorization code grant is used.
   3. The attacker has access to the "client_id" and "client_secret" (if provisioned).  All OAuth 2.0 native app client-instances use the same "client_id".  Secrets provisioned in client binary applications cannot be considered confidential.
   4. Either one of the following condition is met:
	   1. The attacker (via the installed application) is able to observe only the responses from the authorization endpoint. When "code_challenge_method" value is "plain", only this attack is mitigated.
	   2. A more sophisticated attack scenario allows the attacker to observe requests (in addition to responses) to the authorization endpoint.  The attacker is, however, not able to act as a man in the middle.  This was caused by leaking http log information in the OS.  To mitigate this, "code_challenge_method" value must be set either to "S256" or a value defined by a cryptographically secure "code_challenge_method" extension.


## [1.1](https://www.rfc-editor.org/rfc/rfc7636#section-1.1).  Protocol Flow
										                                                  +-------------------+
										                                                  |   Authz Server    |
       +--------+                                | +---------------+ |
       |        |--(A)- Authorization Request ---->|               | |
       |        |       + t(code_verifier), t_m  | | Authorization | |
       |        |                                | |    Endpoint   | |
       |        |<-(B)---- Authorization Code -----|               | |
       |        |                                | +---------------+ |
       | Client |                                |                   |
       |        |                                | +---------------+ |
       |        |--(C)-- Access Token Request ---->|               | |
       |        |          + code_verifier       | |    Token      | |
       |        |                                | |   Endpoint    | |
       |        |<-(D)------ Access Token ---------|               | |
       +--------+                                | +---------------+ |
										                                                   +-------------------+

- The client creates and records a secret named the "code_verifier" and derives a transformed version "t(code_verifier)" (referred to as the "code_challenge"), which is sent in the Authorization Request along with the transformation method "t_m".
- The Authorization Endpoint responds as usual but records "t(code_verifier)" and the transformation method.
- The client then sends the authorization code in the Access Token Request as usual but includes the "code_verifier" secret generated at (A).
- The authorization server transforms "code_verifier" and compares it to "t(code_verifier)" from (B). Access is denied if they are not equal.

   An attacker who intercepts the authorization code at (B) is unable to  redeem it for an access token, as they are not in possession of the "code_verifier" secret.

# [3](https://www.rfc-editor.org/rfc/rfc7636#section-3).  Terminology
- **code verifier**: A cryptographically random string that is used to correlate the authorization request to the token request. 
- **code challenge**: A challenge derived from the code verifier that is sent in the authorization request, to be verified against later. 
- **code challenge method**: A method that was used to derive code challenge. 
- **Base64url Encoding**: Base64 encoding using the URL-and filename-safe character set

# [4](https://www.rfc-editor.org/rfc/rfc7636#section-4).  Protocol
#### [4.1](https://www.rfc-editor.org/rfc/rfc7636#section-4.1).  Client creates a Code Verifier
The client first creates a code verifier,
[ABNF](https://en.wikipedia.org/wiki/Augmented_Backus%E2%80%93Naur_form) for "code_verifier" is as follows.
```
   code-verifier = 43*128unreserved
   unreserved = ALPHA / DIGIT / "-" / "." / "_" / "~"
   ALPHA = %x41-5A / %x61-7A
   DIGIT = %x30-39
```

#### [4.2](https://www.rfc-editor.org/rfc/rfc7636#section-4.2).  Client Creates the Code Challenge
The client then creates a code challenge derived from the code verifier by using one of the following transformations on the code verifier: 
- **plain**: code_challenge = code_verifier 
- **S256**: code_challenge = BASE64URL-ENCODE(SHA256(ASCII(code_verifier)))

#### [4.3](https://www.rfc-editor.org/rfc/rfc7636#section-4.3).  Client Sends the Code Challenge with the Authorization Request
- **code_challenge** REQUIRED.  
- **code_challenge_method** OPTIONAL, defaults to "plain" if not present in the request.  Code verifier transformation method is "S256" or "plain".

#### [4.4](https://www.rfc-editor.org/rfc/rfc7636#section-4.4).  Server Returns the Code
When the server issues the authorization code in the authorization response, it MUST associate the "code_challenge" and "code_challenge_method" values with the authorization code, so it can be verified later. 
Typically, the "code_challenge" and "code_challenge_method" values are stored in encrypted form in the "code" itself but could alternatively be stored on the server associated with the code.  The server MUST NOT include the "code_challenge" value in client requests in a form that other entities can extract.

#### [4.5](https://www.rfc-editor.org/rfc/rfc7636#section-4.5).  Client Sends the Authorization Code and the Code Verifier to the Token Endpoint
- **code_verifier**: REQUIRED

#### [4.6](https://www.rfc-editor.org/rfc/rfc7636#section-4.6).  Server Verifies code_verifier before Returning the Tokens
Upon receipt of the request at the token endpoint, the server verifies it by calculating the code challenge from the received "code_verifier" and comparing it with the previously associated "code_challenge", after first transforming it according to the "code_challenge_method" method specified by the client.


