# KILT DID Sign API (Spec version 1.2)

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
    getSignedDidCreationExtrinsic: (
        /** KILT address that will be submitting the transaction  */
        submitter: string,
        /** Optional: the pending DID URI, submitting the returned extrinsic should create a DID with the same subject */
        pendingDidUri?: string,
    ) => Promise<SignedDidCreationExtrinsic>

    signWithDid: (
        /** Text to be signed */
        plaintext: string,
        /** Optional: The DID URI the extension should use to sign the text */
        didUri: string,
    ) => Promise<Signed>
    
    signExtrinsicWithDid: (
        /** The extrinsic to be DID-authorized as hex-encoded string */
        extrinsic: `0x${string}`,
        /** KILT address that will be submitting the transaction  */
        submitter: string,
        /** Optional: The DID URI the extension should use to authorize the extrinsic */
        didUri: string,
    ) => Promise<SignedExtrinsic>;
}

interface SignedDidCreationExtrinsic {
    /** The signed DID creation extrinsic as hex-encoded string */
    signedExtrinsic: `0x${string}`;
}

interface Signed {
    /** Signature of the input as hex-encoded string */
    signature: `0x${string}`;

    /** URI of the DID authentication key used to sign the input */
    didKeyUri: string;
}

interface SignedExtrinsic {
    /** The signed extrinsic as hex-encoded string */
    signed: `0x${string}`;

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
    getSignedDidCreationExtrinsic: async (
        submitter: string,
        pendingDidUri?: string,
    ): Promise<SignedDidCreationExtrinsic> => {
        /*...*/
        return { signedExtrinsic }
    },
    signWithDid: async (
        plaintext: string,
        didUri: string,
    ): Promise<Signed> => {
        /*...*/
        return { signature, didKeyUri };
    },
    signExtrinsicWithDid: async (
        extrinsic: `0x${string}`,
        submitter: string,
        didUri: string,
    ): Promise<SignedExtrinsic> => {
        /*...*/
        return { signed, didKeyUri };
    },
} as InjectedWindowProvider;
```

The extension SHOULD perform the following tasks in `getSignedDidCreationExtinsic`, `signWithDid` and `signExtrinsicWithDid`:
- protect against Denial-of-Service attacks where the dApp floods the extension with requests
- ensure that the user has previously authorized interaction with this dApp
- otherwise, request user authorization for this interaction


## Cryptography

The signing of `getSignedDidCreationExtinsic` is done using a keypair.
The allowed types for these keys are sr25519 and ed25519.

The signing of `signWithDid` and `signExtrinsicWithDid` is done using an authorization key of a DID.
The allowed types for these keys are sr25519, ed25519, and ecdsa.

* The type sr25519 uses Schnorr signature scheme with Curve25519 keypair.
* The type ed25519 uses Ed25519 signature scheme with Curve25519 keypair.
* The type ecdsa uses ECDSA signature scheme with secp256k1 curve keypair and blake2 as the hashing algorithm.


## DID Creation

1. The dApp calls `getSignedDidCreationExtinsic` passing the submitter as a parameter.
2. The extension presents an interface for generating and signing the on-chain DID creation extrinsic.
   It SHOULD allow the user to reject the creation and signing of the extrinsic.
   It SHOULD allow the user to choose the keypair for signing in case the `pendingDidUri` parameter was not provided.
   When `pendingDidUri` was provided, the extension SHOULD only let the user sign with such a keypair
   that the resulting extrinsic will create a DID with the same subject as `pendingDidUri`.
3. The extension generates and signs the on-chain DID creation extrinsic.
   It MUST include an authentication key.
   It MAY include other keys and services.
4. The extension resolves the promise returned from `getSignedDidCreationExtinsic` with the signed extrinsic.


## The signing flow

1. The dApp calls `signWithDid` passing the plaintext as a parameter 
   or `signExtrinsicWithDid` passing the extrinsic and the submitter.
2. The extension presents to the user the interface to sign. 
   If `signWithDid` was called, it MUST display the plaintext for user to inspect.
   If `signExtrinsicWithDid` was called, it MUST display the extrinsic details (e.g. section, method, and parameters) for user to inspect.
   It SHOULD allow the user to choose the DID to use for signing in case the `didUri` parameter was not provided.
   When `didUri` was provided, the extension SHOULD only let the user sign with this DID.
   It SHOULD alert the user that this DID will be exposed.
   It SHOULD allow the user to reject the signing.
3. The extension creates the signature of the plaintext using the authorization key of the chosen DID.
4. The extension resolves the promise returned from `signWithDid` with the signature and the URI of the key used
   or the promise returned from `signExtrinsicWithDid` with the signed extrinsic and also the URI of the key used.


## Errors

The promise returned by a call to `getSignedDidCreationExtinsic`, `signWithDid` or `signExtrinsicWithDid` MUST be rejected with an instance of `Error`
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
