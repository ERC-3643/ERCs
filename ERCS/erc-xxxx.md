---
eip: xxxx
title: ONCHAINID - An Onchain Identity System
description: Formalizing ONCHAINID, a self-sovereign identity system on Ethereum.
author: Joachim Lebrun (@Joachim-Lebrun), Luc Falempin (@lfalempin), Tony Malghem (@TonyMalghem)
discussions-to: //TBD
status: Draft
type: Standards Track
category: ERC
created: 2025-xx-xx
---

## Abstract

This ERC defines a minimal interface for on-chain identity contracts (ONCHAINID) that can hold and manage cryptographic claims from trusted issuers. The standard provides the essential functionality needed for identity verification in decentralized applications while maintaining compatibility with existing smart wallet architectures and the ERC-3643 ecosystem.

## Motivation

The blockchain ecosystem has seen significant growth in identity-based applications, particularly in regulated token standards like ERC-3643 which has enabled the tokenization of billions of dollars in assets. Current identity solutions reference informal standards (ERC734/ERC735) that were never formalized through the EIP process, creating uncertainty and fragmentation in implementations.

With the expanding adoption of smart wallets and the need for standardized identity verification in DeFi protocols, there is a critical need for a minimal, flexible on-chain identity standard that:

1. Enables any contract to function as an identity holder
2. Supports claim-based attestations from trusted issuers
3. Maintains compatibility with existing smart wallet architectures
4. Provides sufficient functionality for compliance and verification use cases
5. Remains lightweight to encourage broad adoption

## Specification

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

### Core Interface

Every compliant contract MUST implement the following interface:

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.0;

interface IERC_XXXX_OnChainIdentity {
    
    // Events
    event ClaimAdded(
        bytes32 indexed claimId,
        uint256 indexed topic,
        uint256 scheme,
        address indexed issuer,
        bytes signature,
        bytes data,
        string uri
    );
    
    event ClaimChanged(
        bytes32 indexed claimId,
        uint256 indexed topic,
        uint256 scheme,
        address indexed issuer,
        bytes signature,
        bytes data,
        string uri
    );
    
    // Required Functions
    function addClaim(
        uint256 _topic,
        uint256 _scheme,
        address _issuer,
        bytes calldata _signature,
        bytes calldata _data,
        string calldata _uri
    ) external returns (bytes32 claimId);
    
    function getClaim(bytes32 _claimId) external view returns (
        uint256 topic,
        uint256 scheme,
        address issuer,
        bytes memory signature,
        bytes memory data,
        string memory uri
    );
    
    function getClaimIdsByTopic(uint256 _topic) external view returns (bytes32[] memory claimIds);
}
```

### Claim Issuer Interface

Contracts that issue claims to other identities MUST implement this interface:

``` solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.0;

interface IERC_XXXX_ClaimIssuer {
    
    function isClaimValid(
        address _identity,
        uint256 _claimTopic,
        bytes calldata _signature,
        bytes calldata _data
    ) external view returns (bool claimValid);
}
```

### Optional Execution Interface

Contracts that support nested execution patterns (particularly claim issuers) SHOULD implement:

```solidity
interface IERC_XXXX_Execution {
    
    // Events
    event ExecutionRequested(
        uint256 indexed executionId,
        address indexed to,
        uint256 indexed value,
        bytes data
    );
    
    event Executed(
        uint256 indexed executionId,
        address indexed to,
        uint256 indexed value,
        bytes data
    );
    
    // Function
    function execute(
        address _to,
        uint256 _value,
        bytes calldata _data
    ) external payable returns (uint256 executionId);
}
```


### Behavior Specifications

#### Claim Management

1. **Claim IDs**: MUST be generated using `keccak256(abi.encode(_issuer, _topic))`
2. **Adding Claims**: 
   - MUST emit `ClaimAdded` for new claims
   - MUST emit `ClaimChanged` for updates to existing claims
   - MUST validate claims from external issuers using `isClaimValid`
   - MAY allow self-attested claims without validation
3. **Claim Retrieval**: MUST return complete claim data including metadata
4. **Topic Filtering**: MUST support retrieval of all claims by topic

#### Claim Validation

1. **Signature Verification**: MUST verify signatures against expected message format: `keccak256(abi.encode(_identity, _claimTopic, _data))`
2. **Issuer Authority**: MUST validate that the signer has appropriate authority to issue claims
3. **Revocation**: SHOULD check revocation status if the issuer supports it

#### Execution (Optional)

1. **Transaction Wrapping**: SHOULD support wrapping calls for nested operations
2. **Access Control**: MAY implement custom authorization logic for executions
3. **Event Emission**: MUST emit appropriate events for execution requests and completions

## Rationale

### Minimal Interface Design

This standard intentionally excludes several features from informal ERC734/ERC735 specifications to maximize compatibility and adoption:

**Excluded Features:**
- **Key Management (ERC734)**: Different smart wallets have varying access control mechanisms. Requiring specific key management would prevent many existing wallets from implementing this standard.
- **Claim Removal**: Basic compliance use cases rarely require claim removal. Revocation can be handled by claim issuers through their validation logic.
- **Approval Mechanisms**: Complex approval workflows add unnecessary overhead for simple claim management.

**Included Features:**
- **Claim Storage and Retrieval**: Essential for any identity verification system
- **Claim Validation**: Required for trust and verification between parties
- **Flexible Execution**: Enables advanced use cases while remaining optional
- **Event Logging**: Necessary for off-chain monitoring and compliance tracking

### Smart Wallet Compatibility

By focusing only on claim management and avoiding key management requirements, this standard allows:
- Existing smart wallets to become identity holders without architectural changes
- Custom access control implementations to coexist with identity functionality
- Progressive adoption without breaking existing systems

### ERC-3643 Ecosystem Integration

The standard provides sufficient functionality for regulated token compliance:
- Identity verification through claims
- Trusted issuer attestations
- Compliance monitoring through events
- Integration with existing infrastructure

## Backwards Compatibility

This standard is designed to be compatible with existing ERC734/ERC735 implementations by providing a subset of their functionality. Existing implementations can easily adopt this standard by exposing the required interface methods.

## Reference Implementation

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.0;

import "./IERC_XXXX_OnChainIdentity.sol";

contract OnChainIdentity is IERC_XXXX_OnChainIdentity {
    
    struct Claim {
        uint256 topic;
        uint256 scheme;
        address issuer;
        bytes signature;
        bytes data;
        string uri;
    }
    
    mapping(bytes32 => Claim) private claims;
    mapping(uint256 => bytes32[]) private claimsByTopic;
    mapping(address => bool) private authorizedClaimAdders;
    
    modifier onlyAuthorized() {
        require(authorizedClaimAdders[msg.sender] || msg.sender == address(this), "Unauthorized");
        _;
    }
    
    function addClaim(
        uint256 _topic,
        uint256 _scheme,
        address _issuer,
        bytes calldata _signature,
        bytes calldata _data,
        string calldata _uri
    ) external override onlyAuthorized returns (bytes32 claimId) {
        claimId = keccak256(abi.encode(_issuer, _topic));
        
        if (_issuer != address(this)) {
            require(
                IERC_XXXX_OnChainIdentity(_issuer).isClaimValid(address(this), _topic, _signature, _data),
                "Invalid claim"
            );
        }
        
        bool isNew = claims[claimId].issuer == address(0);
        
        claims[claimId] = Claim({
            topic: _topic,
            scheme: _scheme,
            issuer: _issuer,
            signature: _signature,
            data: _data,
            uri: _uri
        });
        
        if (isNew) {
            claimsByTopic[_topic].push(claimId);
            emit ClaimAdded(claimId, _topic, _scheme, _issuer, _signature, _data, _uri);
        } else {
            emit ClaimChanged(claimId, _topic, _scheme, _issuer, _signature, _data, _uri);
        }
        
        return claimId;
    }
    
    function getClaim(bytes32 _claimId) external view override returns (
        uint256 topic,
        uint256 scheme,
        address issuer,
        bytes memory signature,
        bytes memory data,
        string memory uri
    ) {
        Claim storage claim = claims[_claimId];
        return (claim.topic, claim.scheme, claim.issuer, claim.signature, claim.data, claim.uri);
    }
    
    function getClaimIdsByTopic(uint256 _topic) external view override returns (bytes32[] memory claimIds) {
        return claimsByTopic[_topic];
    }
    
    function isClaimValid(
        address _identity,
        uint256 _claimTopic,
        bytes calldata _signature,
        bytes calldata _data
    ) external view override returns (bool claimValid) {
        bytes32 dataHash = keccak256(abi.encode(_identity, _claimTopic, _data));
        bytes32 prefixedHash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", dataHash));
        
        address recovered = _recoverSigner(prefixedHash, _signature);
        return authorizedClaimAdders[recovered];
    }
    
    function _recoverSigner(bytes32 _hash, bytes memory _signature) private pure returns (address) {
        if (_signature.length != 65) return address(0);
        
        bytes32 r;
        bytes32 s;
        uint8 v;
        
        assembly {
            r := mload(add(_signature, 32))
            s := mload(add(_signature, 64))
            v := byte(0, mload(add(_signature, 96)))
        }
        
        if (v < 27) v += 27;
        
        return ecrecover(_hash, v, r, s);
    }
}
```


## Security Considerations

### Claim Validation
- Implementations MUST properly validate signatures to prevent impersonation
- Claim issuers SHOULD implement revocation mechanisms for compromised claims
- Applications SHOULD verify claim freshness and validity before relying on them

### Access Control
- Implementations MUST implement appropriate access control for claim addition
- Smart wallets SHOULD integrate identity functions with their existing permission systems
- Execution functions (if implemented) MUST have proper authorization checks

### Signature Replay
- The message format includes the identity address to prevent cross-identity replay attacks

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
