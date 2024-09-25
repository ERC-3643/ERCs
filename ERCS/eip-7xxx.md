---
eip: 7xxx
title: ONCHAINID - An Onchain Identity System
description: Formalizing ONCHAINID, a self-sovereign identity system on Ethereum.
author: Joachim Lebrun (@Joachim-Lebrun), Kevin Thizy (@Nakasar), Matthew Pinnock (@AureliusEth)
discussions-to: //TBD
status: Draft
type: Standards Track
category: ERC
created: 2024-xx-xx
---

## Abstract

ONCHAINID is a blockchain-based identity system that combines the functionalities of a key manager and a claim holder.
The key manager holds keys to sign actions and execute instructions, while the claim holder manages claims that can be
attested by third parties or self-attested. The identity contract defined by ONCHAINID `IIdentity` integrates these
functionalities, providing a comprehensive solution for individuals and organizations to enforce compliance and
access digital assets or protocols with permission mechanisms.

## Motivation

The motivation behind ONCHAINID is to address the inadequacies of the existing Ethereum protocol in managing complex
account structures and verifying onchain claims about an identity. The current protocol lacks a standardized way for
DApps and smart contracts to check the claims about an identity, and there is no formalized way to manage these claims
onchain. Moreover, the protocol does not provide a comprehensive solution for managing keys associated with an account.

ONCHAINID aims to provide a self-sovereign identity system on the blockchain that allows users to create and manage
their own identities. This identification solution enables compliance and identity verifications within the
pseudonymous framework of public blockchain networks. It integrates the functionalities of a key manager and a
claim holder, as proposed in the Key Manager and Claim Holder proposals by Fabian Vogelsteller. However, these
proposals were never formalized as EIPs, remaining at the issue state. This EIP aims to formalize these proposals and
integrate them into the ONCHAINID system.

## Specification

### Key Management
Keys are cryptographic public keys, or contract addresses that have permission to operate the identity or to interact with services in its behalf.

Those permissions are represented by the purposes they key is associated with. The list of purposes is an array of `uint256` and the following purposes are introduced:

- `1`: MANAGEMENT keys, which can manage the identity (considered an Owner of the identity, a MANAGEMENT key have all permissions).
- `2`: EXECUTION keys, which can call the approve method on the identity to execute transaction sent by the identity to other contracts.
- `3`: CLAIM keys, which can add, remove and update claims on the ONCHAINID.

The structure should be as follows:

- `key`: A public key owned by this identity
  - `purpose`: `uint256[]` Array of the key types, like 1 = MANAGEMENT, 2 = EXECUTION, 3 = CLAIM
  - `keyType`: The type of key used, which would be a `uint256` for different key types. e.g. 1 = ECDSA, 2 = RSA, etc.
  - `key`: `bytes32` The public key. // for non-hex and long keys, its the Keccak256 hash of the key
  
  ```solidity
  struct Key {
    uint256[] purposes;
    uint256 keyType;
    bytes32 key;
  }
  ```
#### getKey
Returns the full key data, if present in the identity.

```solidity
function getKey(bytes32 _key) constant returns(uint256[] purposes, uint256 keyType, bytes32 key);
```

#### keyHasPurpose
Returns `TRUE` if a key is present and has the given purpose. If the key is not present it returns `FALSE`.

```solidity
function keyHasPurpose(bytes32 _key, uint256 purpose) constant returns(bool exists);
```

#### getKeysByPurpose
Returns an array of public key `bytes32` held by this identity.

```solidity
function getKeysByPurpose(uint256 _purpose) constant returns(bytes32[] keys);
```

#### addKey
Adds a `_key` to the identity. The `_purpose` specifies the purpose of the key.

**MUST** only be done by keys of purpose `1`, or the identity itself. If it's the identity itself, the approval process will determine its approval.

**Triggers Event**: `KeyAdded`

```solidity
function addKey(bytes32 _key, uint256 _purpose, uint256 _keyType) returns (bool success);
```

#### removeKey
Removes `_key` from the identity.

**MUST** only be done by keys of purpose `1`, or the identity itself. If it's the identity itself, the approval process will determine its approval.

**Triggers Event**: `KeyRemoved`

```solidity
function removeKey(bytes32 _key, uint256 _purpose) returns (bool success);
```

### Claim Management

#### Claim storage

An identity contract can hold claims. Claims are structured data that bear information, being by the content of
the `data` property or by its sole existence.

A claim is structured as follow:
```solidity
struct claim {
  uint256 topic;
  uint256 scheme;
  address issuer;
  bytes signature;
  bytes data;
  string uri;
}
```

| Field       | Description                                                                                                                               |
|-------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| `topic`     | An integer representing the subject of the claim.                                                                                         |
| `scheme`    | An integer specifying the format of the claim data property.                                                                              |
| `issuer`    | The contract address of the Claim Issuer that issued the claim (for self-attested claims, this will be the identity contract address).    |
| `signature` | The signature of the claim by the claim issuer (signature method is not standardized).                                                    |
| `data`      | Data property of the claim (digest of the data is included in the signature). The content is formatted according to the specified scheme. |
| `uri`       | Claims may reference an external storage (IPFS, API, etc) for additional content.                                                         |

> This specification does not cover the algorithm and methods for claim signature. Each claim issuer may use different keys format.

##### Adding a claim

To store or update a claim on the Identity, the method `addClaim()` is called.

This method MUST emit a `ClaimAdded` event when a claim is stored, and `ClaimChanged` when a claim is updated.

```solidity
/**
 * @dev Emitted when a claim was added.
 *
 * Specification: MUST be triggered when a claim was successfully added.
 */
event ClaimAdded(
  bytes32 indexed claimId,
  uint256 indexed topic,
  uint256 scheme,
  address indexed issuer,
  bytes signature,
  bytes data, 
  string uri
);
```

```solidity
/**
 * @dev Emitted when a claim was changed.
 *
 * Specification: MUST be triggered when addClaim was successfully called on an existing claimId.
 */
event ClaimChanged(
  bytes32 indexed claimId,
  uint256 indexed topic,
  uint256 scheme,
  address indexed issuer,
  bytes signature,
  bytes data,
  string uri
);
```

An Identity must store only one claim of a given topic for a claim issuer. Attempting to add another claim with the
same pair of topic and issuer MUST result in replacing the existing claim with the new one (and emits the
`ClaimChanged` instead of the `ClaimAdded`).

##### Removing a claim

To remove a claim from the Identity, the method `removeClaim()` is called.

The method MUST emit a `ClaimRemoved` event.

```solidity
/**
 * @dev Emitted when a claim was removed.
 *
 * Specification: MUST be triggered when removeClaim was successfully called.
 */
event ClaimRemoved(
  bytes32 indexed claimId,
  uint256 indexed topic,
  uint256 scheme,
  address indexed issuer,
  bytes signature,
  bytes data,
  string uri
);
```

##### Accessing claims

Identity contracts MUST expose methods to retrieve claims.

To retrieve a claim by its ID, the method `getClaim()` is called.
The ID of the claim is computed with `keccak256(abi.encode(issuer, topic))`. One issuer can have at most one active
claim of a given topic for an identity.
This method MUST return the complete claim structure:

```solidity
/**
 * @dev Get a claim by its ID.
 *
 * Claim IDs are generated using `keccak256(abi.encode(address issuer_address, uint256 topic))`.
 */
function getClaim(bytes32 _claimId)
external view returns(
    uint256 topic,
    uint256 scheme,
    address issuer,
    bytes memory signature,
    bytes memory data,
    string memory uri
);
```

For convenience, the method `getClaimIdsByTopic()` is introduced as a mandatory implementation.
This method MUST return an array of claim IDs for a given topic.

```solidity
/**
 * @dev Returns an array of claim IDs by topic.
 */
function getClaimIdsByTopic(uint256 _topic) external view returns(bytes32[] memory claimIds);
```

> An identity MAY not send all claims of a given topics but only a subset of them. An implementation COULD let the
> identity owner select only a few claims to be presented each time.

#### Self-attested claims

Claims are usually issued by third parties about an identity, but some claims are intended to be created by the
identity owner themselves. These claims are called self-attested claims.

Self-attested claims addition, updates and removals MUST trigger the `ClaimAdded`, `ClaimChanged` and `ClaimRemoved`
events, and they SHOULD be managed by the same methods `addClaim` and `removeClaim`.

Self-attested claims uses the same validity check regarding signatures (algorithms are left to the appreciation of
implementers). Specifications does not include a revoke mechanism, however the Identity contract exposes
a `isClaimValid` method that MUST verify the claim signature and return true if it is valid, or false otherwise.

The `isClaimValid` method MAY implement a revocation behavior based on timestamp, revocation lists or any other
method deemed necessary by the implementer, returning true or false depending on the claim validity status.

```solidity
/**
 * @dev Checks if a claim is valid.
 * @param identity the identity contract related to the claim.
 * @param claimTopic the claim topic of the claim
 * @param sig the signature of the claim
 * @param data the data field of the claim
 * @return claimValid true if the claim is valid, false otherwise
 */
function isClaimValid(
    address identity,
    uint256 claimTopic,
    bytes calldata sig,
    bytes calldata data)
external view returns (bool);
```

### Identity usage

Apart from keys and claims management, the Identity contract specified in ONCHAINID allows for execution requests and
approvals. A contract (or a signer) may create a new execution request by calling the `execute()` method.
Execute may be native transfers of value from the identity contract balance to another address, or execution of methods
on other contracts. When an execution request is created, it awaits approval from authorized key. Upon validation, the
operation is performed.

This behavior allows Identity to hold tokens or to act as signers or simplified "smart wallets". The complexity and
details of the approval process are left to the discretion of implementers.

#### execute
Passes an execution instruction to the Key Manager.
**SHOULD** require approve to be called with one or more keys of purpose `1` or `2` to approve this execution.

Execute **COULD** be used as the only accessor for `addKey`, `removeKey` and `addClaim` and `removeClaim`.

Returns `executionId`: SHOULD be sent to the `approve ` function, to approve or reject this execution.

**Triggers Event**: `ExecutionRequested`

**Triggers on direct execution Event**: `Executed`

```solidity
function execute(address _to, uint256 _value, bytes _data) returns (uint256 executionId);
```

#### approve
Approves an execution or claim addition.
This **SHOULD** require the approval of key purpose `1`, if the `_to` of the execution is the identity contract itself, to successfully approve an execution.
And **COULD** require the approval of key purpose `2`, if the `_to` of the execution is another contract, to successfully approve an execution.

**Triggers Event**: `Approved`

**Triggers on successful execution Event**: `Executed`

```solidity
function approve(uint256 _id, bool _approve) returns (bool success);
```

### Events

Events are very important because most Identity management application will need these to display appropriate
information about the identity state.

#### `KeyAdded`

MUST be triggered when `addKey` was successfully called.

```solidity
event KeyAdded(bytes32 indexed key, uint256 indexed purpose, uint256 indexed keyType);
```

#### `KeyRemoved`

MUST be triggered when `removeKey` was successfully called.

```solidity
event KeyRemoved(bytes32 indexed key, uint256 indexed purpose, uint256 indexed keyType);
```

#### `ExecutionRequested`

This events means an execution request was created and is awaiting approval (note: if the execution was immediately
approved and then performed, the ExecutionRequest would be followed by an Executed event in the same transaction).

MUST be triggered when `execute` was successfully called.

```solidity
event ExecutionRequested(uint256 indexed executionId, address indexed to, uint256 indexed value, bytes data);
```

#### `Executed`

MUST be triggered when `approve` was called and the execution was successfully approved.

```solidity
event Executed(uint256 indexed executionId, address indexed to, uint256 indexed value, bytes data);
```

#### `Approved`

MUST be triggered when `approve` was successfully called.

```solidity
event Approved(uint256 indexed executionId, bool approved);
```

## Rationale



## Backwards Compatibility

There are no known standard using the previous KeyHolder and ClaimHolder proposals. Most methods used are inspired by
specifications of KeyHolder and ClaimHolder introduces by Fabian Vogelsteller (@frozeman), hence application leveraging
claims and keys functionalities should still be compatible.

## Security Considerations

### Privacy

Because claims may be related to personal information, developers and especially claim issuers are expected to consider
the privacy implications of the claims they issue. Especially:
- The content of the claim `data` property are to be considered public and should never contain sensitive information,
even encrypted. For such information, it is recommended to use an integrity hash of information stored offchain
combined with a random string and to include the hash in the claim data. Digest algorithms choice is left to claim
issuers and they may differ from one to another.
- The existence of a claim with a given topic could reveal information about the identity. Claim issuers should not
issue claims too specific and avoid claims when not necessary.

When using an Identity, identity owners (and services managing identities) should keep in mind that:
- identity information should as much as possible be kept off-chain. On-chain claims should only be added to the
identity when they are necessary (usually to interact with permission smart contracts).

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
