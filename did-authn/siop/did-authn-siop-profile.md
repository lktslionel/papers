# DID AuthN profile for OpenID Connect

**Editors**

| Author          | Company  |
| ------------- | -------- |
| Oliver Terbu  | uPort/ ConsenSys          |

**Status:** 0.1 DRAFT

## 1 Abbreviations
| Short         | Long
| ------------- |--------------------------
| OP            | OpenID Connect Provider
| SIOP          | Self-Issued OP
| RP            | Relying Party/ OIDC Client
| JWE           | JSON Web Encryption
| JWS           | JSON Web Signature
| JWT           | JSON Web Token

## 2 Introduction

An everyday use case that the SSI community identified is the sign-up or login with web
applications. Nowadays, this is often achieved through social login schemes such as
Google Sign-In. While the SSI community has serious concerns about social login,
the underlying protocol, OIDC, does not have these flaws by design. DID AuthN provides
great potential by leveraging an SSI wallet, e.g., as a smartphone app, on the web.
This will increase and preserve the user’s privacy by preventing third-parties from
having the ability to track which web applications a user is interacting with.

This specification defines the SIOP DID AuthN flavor to use OIDC together with
the strong decentralization, privacy and security guarantees of DID for everyone
who wants to have a generic way to integrate SSI wallets into their web applications.

## 3 Purpose and Goals

The main purpose is to sign up with/ login to an RP, i.e., web application. It assumes the user
operates a mobile or desktop browser.

The main goals of this specification are:
- Staying backward compatible with existing OIDC clients that implement the SIOP
  specification which is part of the OIDC core specification to reach a broader community.
- Adding validation rules for OIDC clients that have DID AuthN support to make full use of DIDs.
- Not relying on any intermediary such as a traditional centralized public
or private OP while still being OIDC compliant. 

> **NOTE:** The SIOP flow is conducted peer-to-peer between the RP and the Identity Wallet.
This could be used to authenticate holders based on their DID, to setup/ bootstrap a
DID Comm connection with any DID routing that you may need, or to provide the
`login_hint` to an OpenID Connect service in the DID Document supporting the
 [Client-Initiated Backend Channel (CIBA)]([https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html)
(CIBA) flow.


## 4 Protocol Flow

The user operating a mobile or desktop browser visits a web application and clicks on
sign up or login. The RP will then generate the redirect
to `openid://<SIOP Request>` which will be handled by the identity wallet. On the mobile
browser, this would open the identity wallet
app, e.g., uport, connect.me. On the desktop browser, this would either show a QR code
which can be scanned by the identity wallet app or a redirect to `openid://<SIOP Request>`
that for instance could be handled by a browser plugin representing an identity wallet.

The identity wallet will generate the `<SIOP Response>` based on the specific DID method
that is supported. The `<SIOP Response>` will be signed and optionally
encrypted and will be provided according to the requested `response_mode`.  

This SIOP does not explicitly support any intermediate cloud agents, although the callback
could be used to point to a cloud agent. It is meant to be a protocol to
exchange the DID. You could then interact with a cloud agent using the service endpoint.

Unlike the Authorization Code Flow, SIOP will not return an access token to the RP.
If this is desired, this could be achieved by following the aforementioned [CIBA]([https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html)
flow in addition. SIOP also differs from Authorization Code Flow by not relying on a
centralized and known OP. The SIOP can be unknown to the RP until users start to
interact with the RP using their Identity Wallet. Authorization Code Flow is still
a useful approach and should be used whenever the OP is known, and OP discovery is
possible. An integration of Identity Wallets would be able to interact with plain
OIDC clients if they implemented the SIOP protocol. In contrast, using DID AuthN
in the authentication step of the Authorization Code Flow must be done with every
OP vendor such as ForgeRock, etc.

![DID AuthN SIOP Profile](assets/did_authn_siop_profile_flow.png)

> **NOTE:** In the diagram above, the "Identity Wallet" represents the SIOP, and the "RP" represents the requesting party.

### Generate &lt;SIOP Request&gt;

#### Redirect Request
The request contains `response_type` and `client_id` as query string parameters for backward compatibility with the OAuth2 specification. `response_type` MUST be
`id_token` and `client_id` MUST specify the callback URL of the RP (as per [SIOP](https://openid.net/specs/openid-connect-core-1_0.html#SelfIssued)). All other OIDC request parameters MUST be provided in an [Request Object](https://openid.net/specs/openid-connect-core-1_0.html#RequestObject) which is encoded as a JWT. This enables the RP to authenticate against the SIOP using the RP's DID. The Base64-URL-encoded [Request Object](https://openid.net/specs/openid-connect-core-1_0.html#RequestObject) can be passed by value in the `request` request parameter, or by reference using the `request_uri` parameter.

The following is a non-normative example of an DID AuthN &lt;SIOP Request&gt; initiated by the RP using [Request Object](https://openid.net/specs/openid-connect-core-1_0.html#RequestObject) by value:
```
  openid://?response_type=id_token
    &client_id=https%3A%2F%2Frp.example.com%2Fcb
    &request=<JWT>
```

The following is a non-normative example of an DID AuthN &lt;SIOP Request&gt; initiated by the RP using [Request Object](https://openid.net/specs/openid-connect-core-1_0.html#RequestObject) by reference:
```
  openid://?response_type=id_token
    &client_id=https%3A%2F%2Frp.example.com%2Fcb
    &request_uri=https%3A%2F%2Frp.example.com%2F90ce0b8a-a910-4dd0
```

#### Request Object
The [Request Object](https://openid.net/specs/openid-connect-core-1_0.html#RequestObject) follows the OIDC specification, e.g., adding `nonce`, `state`, `response_type`, and `client_id` parameters.

This specification introduces additional constraints for request parameters:
- `iss` MUST contain the DID of the RP that can be resolved to a DID Document containing the verification key of the [Request Object](https://openid.net/specs/openid-connect-core-1_0.html#RequestObject).

> **NOTE:** By default, the `iss` attribute of the [Request Object](https://openid.net/specs/openid-connect-core-1_0.html#RequestObject) refers to the `client_id` but SIOP assumes that `client_id` is the callback URL of the RP. That is the reason why the DID is not encoded in the `client_id`. Note, it is compliant with the OIDC specification to use different values for `iss` and `client_id`.

- `scope` MUST include `did_authn` to indicate the DID AuthN profile is used.
- `kid` MUST be a DID URL referring to a public key in the RP's DID Document, e.g., `did:example:0xab#key1`.

#### RP Meta-data

In contrast to other OIDC flows, e.g., Authorization Code Flow, RPs can provide client meta-data in the `registration` request parameter. The `registration` parameter MUST be included in the [Request Object](https://openid.net/specs/openid-connect-core-1_0.html#RequestObject).

The `registration` parameter MUST indicate the signing algorithm in the `request_object_signing_alg` attribute. The SIOP MUST support `Ed25519` and `ES256K` in addition to `RS256`. RPs implementing the DID AuthN profile MUST not use `none`.

The JWS of the [Request Object](https://openid.net/specs/openid-connect-core-1_0.html#RequestObject) MUST be verifiable by a key in the RP's DID Document. Additionally, `jwks_uri` and `jwks` MUST contain this key and MUST use the same `kid` to identify the key.



Due to the fact that the signing algorithm of JWS of the `id_token` depends on the DID method used
by the SIOP, and not all SIOP can support all signing algorithms, an RP implementing the SIOP DID
AuthN profile MUST support all of the following signature algorithms and MUST set the
`id_token_signed_response_alg` in the `registration` parameter accordingly to `["RS256", "Ed25519", "ES256K"]`.

> **NOTE:** `request_object_signing_alg`, `jwks_uri` and `jwks` are used for backward compatibility reasons.

RPs can decide to receive the &lt;SIOP Response&gt; encrypted. To enable encryption, the `registration` parameter MUST use `id_token_encrypted_response_alg` and `id_token_encrypted_response_enc` according to [OIDC Client Metadata](https://openid.net/specs/openid-connect-registration-1_0.html#ClientMetadata).

#### Response Modes

The `reponse_mode` request parameter specifies how the response is returned to the callback URL by the SIOP. SIOP implementing the DID AuthN specification MAY set the `response_mode` to `query`, or `form_post`. `fragment` is the default Response Mode. RPs MUST take into consideration the platform of the User Agent when specifying this request parameter.

See [OAuth 2.0 Form Post Response Mode](https://openid.net/specs/oauth-v2-form-post-response-mode-1_0.html) and [OAuth 2.0 Multiple Response Type Encoding Practices]( https://openid.net/specs/oauth-v2-multiple-response-types-1_0.html) for more information about `response_mode`.

#### Non-normative Examples of a &lt;SIOP Request&gt;

The following is a non-normative example of the JWT header of a [Request Object](https://openid.net/specs/openid-connect-core-1_0.html#RequestObject):
```json=
{
   "alg": "ES256K",
   "typ": "JWT",
   "kid": "did:example:0xab#veri-key1"
}
```

The following is a non-normative example of the JWT payload of a [Request Object](https://openid.net/specs/openid-connect-core-1_0.html#RequestObject) without requesting &lt;SIOP Response&gt; encryption:
```json=
{
    "iss": "did:example:0xab",
    "response_type": "id_token",
    "client_id": "https://my.rp.com/cb",
    "scope": "openid did_authn",
    "state": "af0ifjsldkj",
    "nonce": "n-0S6_WzA2Mj",
    "response_mode" : "query",
    "registration" : {
        "request_object_signing_alg" : "ES256K",
        "jwks_uri" : "did:example:0xab",
        "id_token_signed_response_alg" : [ "ES256K", "Ed25519", "RS256" ],
    }
}
```

### &lt;SIOP Request&gt; Validation

The SIOP MUST validate the &lt;SIOP Request&gt; by following the [Self-Issued ID Token Validation](https://openid.net/specs/openid-connect-core-1_0.html#SelfIssuedValidation) rules.

> **NOTE:** The validation rules will verify the JWS using `kid`, `jwks_uri`, `jwks` and `request_object_signing_alg`.

If `scope` contains the `did_authn` scope, the receiving SIOP MUST further validate the &lt;SIOP Request&gt; as follows:

- Resolve the DID Document from the RP's DID specified in the `iss` request parameter.
- Verify that the `kid` in the &lt;SIOP Request&gt; corresponds to the key in the RP's DID Document. This step depends on the [publicKey property value](https://w3c-ccg.github.io/did-spec/#public-keys) in the DID Document and is out-of-scope of this specification.

> **NOTE:** The DID Document MAY use a [publicKey property value](https://w3c-ccg.github.io/did-spec/#public-keys) other than `publicKeyJwk`. In that case, the SIOP has to transform the encoding of the key to verify that the key corresponds to the key used to sign the &lt;SIOP Request&gt;.

### Generate &lt;SIOP Response&gt;

The SIOP MUST generate and send the &lt;SIOP Response&gt; to the RP as described in the [Self-Issued OpenID Provider Response](https://openid.net/specs/openid-connect-core-1_0.html#SelfIssuedResponse) section. The `id_token` represents the &lt;SIOP Response&gt; encoded as a JWS, or nested JWS/JWE.

The `id_token` MUST be signed by a key that corresponds to a key in the DID Document of the SIOP. As a consequence, the `sub_jwk` attribute including the `kid` MUST refer to the same key. Additionally, the `id_token` MAY include a `did` claim. In that case, the `did` claim MUST be the SIOP's DID.

> **NOTE:** The `sub_jwk` attribute has to be provided for backward compatibility reasons. The key in the DID Document MAY use a [publicKey property value](https://w3c-ccg.github.io/did-spec/#public-keys) other than `publicKeyJwk`.

The following is a non-normative example of the JWT header of an `id_token` using no encryption:
```json=
{
   "alg": "ES256K",
   "typ": "JWT",
   "kid": "did:example:0xab#key-1"
}
```

The following is a non-normative example of the unencrypted JWT payload of an `id_token`:
```json=
{
   "iss": "https://self-issued.me",
   "nonce": "n-0S6_WzA2Mj",
   "exp": 1311281970,
   "iat": 1311280970,
   "sub_jwk" : {
      "crv":"P-256K",
      "kid":"did:example:0xcd#verikey-1",
      "kty":"EC",
      "x":"7KEKZa5xJPh7WVqHJyUpb2MgEe3nA8Rk7eUlXsmBl-M",
      "y":"3zIgl_ml4RhapyEm5J7lvU-4f5jiBvZr4KgxUjEhl9o"
   },
   "sub": "9-aYUQ7mgL2SWQ_LNTeVN2rtw7xFP-3Y2EO9WV22cF0",
   "did_comm" : {
     "did" : "did:example:0xcd",
     "did_doc" : "...."
   }
}
```

### &lt;SIOP Response&gt; Validation

The RP MUST validate the &lt;SIOP Response&gt; as described in the [Self-Issued ID Token Validation](https://openid.net/specs/openid-connect-core-1_0.html#SelfIssuedValidation) section. This includes:
- Optionally decrypting the JWE to obtain the JWS which contains the `id_token`.
- Verifying that the `id_token` was signed by the key specified in the `sub_jwk` attribute.

If the `did` attribute is present, the RP MUST verify that the `id_token` was signed by a key in the SIOP's DID Document as follows:

- Resolve the `did` attribute value to a DID Document.
- Verify that `sub_jwk` refers to a key in the DID Document, or
- verify the signature of the `id_token` using one of the keys in the DID Document. The RP MAY use the `kid` from the `sub_jwk` to identify which key to use.

### SIOP Discovery

The SIOP specification assumes the following OP discovery meta-data:
```json
"id_token_signing_alg_values_supported": ["RS256"],
"request_object_signing_alg_values_supported": ["none", "RS256"]
```

The DID AuthN profile assumes the following OP discovery meta-data:
```json
"id_token_signing_alg_values_supported": ["RS256", "ES256K", "Ed25519"],
"request_object_signing_alg_values_supported":
   ["none", "RS256", "ES256K", "Ed25519"]
```

This change will allow DID AuthN enabled RPs to use additional signature algorithms commonly used amongst members of the SSI community.

> **NOTE:** "Self-Issued OpenID Provider Discovery" **IS NOT** normative and
does not contain any *MUST*, *SHOULD*, or *MAY* statements. Therefore, using a different signing algorithmn than `RS256` shouldn't break the SIOP specification. An DID AuthN enabled RP would provide `id_token_signed_response_alg` to indicate which signature algorithms other than `RS256` are supported, and can assume that SIOP implementing the DID AuthN profile support any of the additional algorithms.

## 5 UX Considerations

SIOP uses the custom URL scheme `openid://. Mobile browsers would open the app that 
registered that scheme. Desktop Browser plugins have support for similar functionality.
It is out of the scope of the spec under which circumstances a QR code will be rendered.
One option will be to provide the QR code if the user is using the desktop browser.

On Android, the user can choose which app should open if multiple apps registered the
same custom URL scheme. On iOS, the behavior is undefined. One approach would be to
check if the user is on an iOS device and then, won't render the button if this
is a concern. A fallback on iOS could be the use of custom mime types, but bad UX
expert has to be considered because the user is not used to this type of flow.

> **NOTE:** This issue is not specific to SIOP only but affects all apps using custom URL schemes.

> **NOTE:** In case a QR Code is used where the user has to open the app first and has
 to scan the QR Code, there won't be an issue.





