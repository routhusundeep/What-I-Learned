[RFC](https://www.rfc-editor.org/rfc/rfc8628#section-3.4)

## [1](https://www.rfc-editor.org/rfc/rfc8628#section-1).  Introduction
The operating requirements for using this authorization grant type are: 
- The device is already connected to the Internet.
- The device is able to make outbound HTTPS requests. 
- The device is able to display or otherwise communicate a URI and code sequence to the user.
- The user has a secondary device (e.g., personal computer or smartphone) from which they can process the request.

Instead of interacting directly with the end user's user agent (i.e., browser), the device client instructs the end user to use another computer or device and connect to the authorization server to approve the access request.  Since the protocol supports clients that can't receive incoming requests, clients poll the authorization server repeatedly until the end user completes the approval process.

      +----------+                                +----------------+
      |          |>---(A)-- Client Identifier --->|                |
      |          |                                |                |
      |          |<---(B)-- Device Code,      ---<|                |
      |          |          User Code,            |                |
      |  Device  |          & Verification URI    |                |
      |  Client  |                                |                |
      |          |  [polling]                     |                |
      |          |>---(E)-- Device Code       --->|                |
      |          |          & Client Identifier   |                |
      |          |                                |  Authorization |
      |          |<---(F)-- Access Token      ---<|     Server     |
      +----------+   (& Optional Refresh Token)   |                |
            v                                     |                |
            :                                     |                |
           (C) User Code & Verification URI       |                |
            :                                     |                |
            v                                     |                |
      +----------+                                |                |
      | End User |                                |                |
      |    at    |<---(D)-- End user reviews  --->|                |
      |  Browser |          authorization request |                |
      +----------+                                +----------------+

## [3](https://www.rfc-editor.org/rfc/rfc8628#section-3).  Protocol
### [3.1](https://www.rfc-editor.org/rfc/rfc8628#section-3.1).  Device Authorization Request
This specification defines a new OAuth endpoint: the device authorization endpoint. This is separate from the OAuth authorization endpoint with which the user interacts via a user agent (i.e., a browser).  By comparison, when using the device authorization endpoint, the OAuth client on the device interacts with the authorization server directly without presenting the request in a user agent, and the end user authorizes the request on a separate device.
- **client_id**: REQUIRED
- **scope**: OPTIONAL
Due to the polling nature of this protocol, care is needed to avoid overloading the capacity of the token endpoint.  To avoid unneeded requests on the token endpoint, the client SHOULD only commence a device authorization request when prompted by the user and not automatically, such as when the app starts or when the previous authorization session expires or fails.

### [3.2](https://www.rfc-editor.org/rfc/rfc8628#section-3.2).  Device Authorization Response
- **device_code**: REQUIRED
- **user_code**: REQUIRED
- **verification_uri**: REQUIRED.  The end-user verification URI on the authorization server.  The URI should be short and easy to remember as end users will be asked to manually type it into their user agent.
- **verification_uri_complete**: OPTIONAL.  A verification URI that includes the "user_code" (or other information with the same function as the "user_code"), which is designed for non-textual transmission.
- **expires_in**: The lifetime in seconds of the "device_code" and "user_code".
- **interval**: OPTIONAL. The minimum amount of time in seconds that the client SHOULD wait between polling requests to the token endpoint. If no value is provided, clients MUST use 5 as the default.

      HTTP/1.1 200 OK
      Content-Type: application/json
      Cache-Control: no-store

      {
        "device_code": "GmRhmhcxhwAzkoEqiMEg_DnyEysNkuNhszIySk9eS",
        "user_code": "WDJB-MJHT",
        "verification_uri": "https://example.com/device",
        "verification_uri_complete":
            "https://example.com/device?user_code=WDJB-MJHT",
        "expires_in": 1800,
        "interval": 5
      }

## [3.3](https://www.rfc-editor.org/rfc/rfc8628#section-3.3).  User Interaction

	+-----------------------------------------------+
	|                                               |
	|  Using a browser on another device, visit:    |
	|  https://example.com/device                   |
	|                                               |
	|  And enter the code:                          |
	|  WDJB-MJHT                                    |
	|                                               |
	+-----------------------------------------------+

The authorizing user navigates to the "verification_uri" and authenticates with the authorization server. The authorization server prompts the end user to identify the device authorization session by entering the "user_code" provided by the client. The authorization server should then inform the user about the action they are undertaking and ask them to approve or deny the request. Once the user interaction is complete, the server instructs the user to return to their device.

### [3.3.1](https://www.rfc-editor.org/rfc/rfc8628#section-3.3.1).  Non-Textual Verification URI Optimization
When "verification_uri_complete" is included in the authorization response, clients MAY present this URI in a non-textual manner using any method that results in the browser being opened with the URI, such as with QR (Quick Response) codes or NFC (Near Field Communication), to save the user from typing the URI. 
For usability reasons, it is RECOMMENDED for clients to still display the textual verification URI ("verification_uri") for users who are not able to use such a shortcut. Clients MUST still display the "user_code", as the authorization server will require the user to confirm it to disambiguate devices or as remote phishing mitigation.

If the user starts the user interaction by navigating to "verification_uri_complete", then the user interaction is still followed, with the optimization that the user does not need to type in the "user_code".  The server SHOULD display the "user_code" to the user and ask them to verify that it matches the "user_code" being displayed on the device to confirm they are authorizing the correct device.

	+-------------------------------------------------+
	|                                                 |
	|  Scan the QR code or, using     +------------+  |
	|  a browser on another device,   |[_]..  . [_]|  |
	|  visit:                         | .  ..   . .|  |
	|  https://example.com/device     | . .  . ....|  |
	|                                 |.   . . .   |  |
	|  And enter the code:            |[_]. ... .  |  |
	|  WDJB-MJHT                      +------------+  |
	|                                                 |
	+-------------------------------------------------+


### [3.4](https://www.rfc-editor.org/rfc/rfc8628#section-3.4).  Device Access Token Request
- **grant_type**: REQUIRED.  Value MUST be set to `urn:ietf:params:oauth:grant-type:device_code`. 
- **device_code**: REQUIRED.  The device verification code, "device_code" from the device authorization response
- **client_id** REQUIRED

### [3.5](https://www.rfc-editor.org/rfc/rfc8628#section-3.5).  Device Access Token Response
- authorization_pending
- slow_down
- access_denied
- expired_token
- 