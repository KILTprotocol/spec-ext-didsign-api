# KILT DID Sign API (Draft Spec version 1.0)

## Definitions

### Extension

A browser extension that stores and uses the DIDs of the user.
When the user visits a webpage, the extension injects its API into this webpage.

### dApp

Decentralized application – a website that can interact with the extension via the API it exposes.
The example dApp in this specification is Signer.


## Specification of requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and
"OPTIONAL" in this document are to be interpreted as described in RFC 2119.


## Setting up the communication session

### Types

```typescript
interface GlobalKilt {
    /** `extensionId` references the extension on the `GlobalKilt` object but is not used by the dApp */
    [extensionId: string]: InjectedWindowProvider
}

interface InjectedWindowProvider {
    signWithDid: (
        /** Text to be signed */
        plaintext: string,
    ) => Promise<Signed>
}

interface Signed {
    /** Signature of the input as hex-encoded string */
    signature: string;

    /** URI of the DID authentication key used to sign the input */
    didKeyUri: string;
}
```


### DApp consumes the API exposed by extension

The dApp MUST create the `window.kilt` object as early as possible to indicate its support of the API to the extension.

```typescript
window.kilt = {}
```

The dApp can afterwards get all available extensions by iterating over the `window.kilt` object.

```typescript
function getWindowExtensions(): InjectedWindowProvider[] {
    return Object.values(window.kilt);
}
```

The dApp should list all available extensions it can work with.
The user selects an extension from this list, and the communication starts from there.


### Extension injects its API into a webpage

The extension MUST only inject itself into pages having the `window.kilt` object.

```typescript
(window.kilt as GlobalKilt).myDidExtension = {
    signWithDid: async (
        plaintext: string,
    ): Promise<Signed> => {
        /*...*/
        return { signature, didKeyUri };
    },
} as InjectedWindowProvider;
```

The extension SHOULD perform the following tasks in `signWithDid`:
- protect against Denial-of-Service attacks where the dApp floods the extension with requests
- ensure that the user has previously authorized interaction with this dApp
- otherwise, request user authorization for this interaction


## Cryptography

The signing is done using an authorization key of a DID.
The allowed types for these keys are sr25519, ed25519, and ecdsa. TODO: link to spec.

* The type sr25519 uses Schnorr signature scheme with Curve25519 keypair.
* The type ed25519 uses Ed25519 signature scheme with Curve25519 keypair.
* The type ecdsa uses ECDSA signature scheme with secp256k1 curve keypair and blake2 as the hashing algorithm.
The signature consists of R, S, and the recovered bit. // TODO: is this sentence necessary?


## The signing flow

1. The dApp calls `signWithDid` passing the plaintext as a parameter.
2. The extension presents to the user the interface to sign. It MUST display the plaintext for user to inspect.
   It SHOULD allow the user to choose the DID to use for signing.
   It SHOULD alert the user that this DID will be exposed.
   It SHOULD allow the user to reject the signing.
3. The extension creates the signature of the plaintext using the authorization key of the chosen DID.
4. The extension resolves the promise returned from `signWithDid` with the signature and the URI of the key used.


## Errors

The promise returned by a call to `signWithDid` MUST be rejected with an instance of `Error`
if an error happens during the flow. In case the user explicitly rejects the signing of the plaintext,
the name and the message properties of the error SHOULD include `Rejected`.
In case the extension was closed otherwise, the name and the message properties of the error SHOULD include `Closed`.

```typescript
interface Error {
    /** optional machine-readable type of the error */
    name?: string

    /** optional human-readable description of the error */
    message?: string
}
```


## Security considerations

**This API does not provide strong security guarantees to reduce friction for developers using it. Be aware that
man-in-the-middle attacks may change the plaintext to be signed, as well as the signature data returned to the dApp.**

The plaintext MUST be presented to the user before signing, so that they can verify it to the extent of
their capabilities. The extension SHOULD alert the user that signing will expose their DID.

The cryptographic strength of the signature algorithms is considered high.
