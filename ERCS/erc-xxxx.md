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

### Execution Interface

Identity contracts SHOULD implement the execution interface to maintain compatibility with existing claim issuer workflows:

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

#### Execution (for Identity Compatibility)

1. **Transaction Wrapping**: SHOULD support wrapping calls for nested operations
2. **Access Control**: MAY implement custom authorization logic for executions
3. **Event Emission**: MUST emit appropriate events for execution requests and completions

## Rationale

### Separation of Concerns
This standard separates identity holding from claim issuing to provide maximum flexibility:
**Core Identity Interface**:
- Minimal requirements for any contract to be considered an identity
- Focused on claim storage and retrieval
- Compatible with diverse smart wallet architectures

**Claim Issuer Interface**:
- Only required for contracts that issue claims to other identities
- Allows specialized validation logic per issuer
- Enables trust relationships between different identity types

**Execution Interface**:
- SHOULD be implemented by identities for ecosystem compatibility
- Many existing claim issuers expect to interact with identities through execution patterns
- Allows claims to be added through the identity's own access control mechanisms
- Identity contracts that do not implement this interface may break compatibility with existing claim issuer workflows

### Minimal Interface Design
This standard intentionally excludes several features from informal ERC734/ERC735 specifications to maximize compatibility and adoption:
**Excluded from Core Interface:**
- **Key Management (ERC734)**: Different smart wallets have varying access control mechanisms. Requiring specific key management would prevent many existing wallets from implementing this standard.
- **Claim Validation**: Moved to separate interface since only claim issuers need this functionality
- **Claim Removal**: Basic compliance use cases rarely require claim removal. Revocation can be handled by claim issuers through their validation logic.
- **Approval Mechanisms**: Complex approval workflows add unnecessary overhead for simple claim management.

**Included in Core Interface:**
- **Claim Storage and Retrieval**: Essential for any identity verification system
- **Event Logging**: Necessary for off-chain monitoring and compliance tracking

### Smart Wallet Compatibility
By focusing only on claim management and avoiding key management requirements, this standard allows:
- Existing smart wallets to become identity holders without architectural changes
- Custom access control implementations to coexist with identity functionality
- Progressive adoption without breaking existing systems

### ERC-3643 Ecosystem Integration
The standard provides sufficient functionality for regulated token compliance:
- Identity verification through claims
- Trusted issuer attestations via the claim issuer interface
- Compliance monitoring through events
- Integration with existing infrastructure

### Universal Attestation Compatibility
The claim structure defined in this standard is designed to be compatible with established verifiable credential frameworks, including W3C Verifiable Credentials standards. The flexible `scheme`, `data` and `uri` fields enable ONCHAINID to function as an attestation gateway and aggregator, capable of bridging various attestation services such as Ethereum Attestation Service (EAS), eIDAS-compliant identity providers, and traditional KYC/AML solutions. This design allows identity holders to consolidate attestations from multiple sources into a single on-chain identity, while maintaining interoperability with existing credential verification systems and enabling seamless integration with diverse compliance workflows.

### Execution Interface Importance
While the execution interface is technically optional, it is strongly recommended for practical adoption because:
- Many claim issuers in the existing ecosystem expect to add claims via execution patterns
- This pattern allows claim issuers to work with identities that have complex permission structures
- It enables claims to be processed through the identity's own access control mechanisms
- Without this interface, identities may face compatibility issues with established claim issuer workflows

## Backwards Compatibility

This standard is designed to be compatible with existing ERC734/ERC735 implementations by providing a subset of their functionality. Existing implementations can easily adopt this standard by exposing the required interface methods.

## Reference Implementation

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.0;

import "./IERC_XXXX_OnChainIdentity.sol";
import "./IERC_XXXX_ClaimIssuer.sol";
import "./IERC_XXXX_Execution.sol";

contract OnChainIdentity is IERC_XXXX_OnChainIdentity, IERC_XXXX_Execution {
    
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
    uint256 private executionNonce;
    
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
        
        // Validate claim if issuer is external and implements claim issuer interface
        if (_issuer != address(this) && _supportsInterface(_issuer, type(IERC_XXXX_ClaimIssuer).interfaceId)) {
            require(
                IERC_XXXX_ClaimIssuer(_issuer).isClaimValid(address(this), _topic, _signature, _data),
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
    
    function execute(
        address _to,
        uint256 _value,
        bytes calldata _data
    ) external payable override returns (uint256 executionId) {
        executionId = executionNonce++;
        
        emit ExecutionRequested(executionId, _to, _value, _data);
        
        (bool success,) = _to.call{value: _value}(_data);
        require(success, "Execution failed");
        
        emit Executed(executionId, _to, _value, _data);
        
        return executionId;
    }
    
    function _supportsInterface(address _contract, bytes4 _interfaceId) internal view returns (bool) {
        // Simple interface detection - could use ERC165 if needed
        try IERC165(_contract).supportsInterface(_interfaceId) returns (bool supported) {
            return supported;
        } catch {
            return false;
        }
    }
}

contract ClaimIssuer is OnChainIdentity, IERC_XXXX_ClaimIssuer {
    
    mapping(address => bool) private authorizedSigners;
    
    function isClaimValid(
        address _identity,
        uint256 _claimTopic,
        bytes calldata _signature,
        bytes calldata _data
    ) external view override returns (bool claimValid) {
        bytes32 dataHash = keccak256(abi.encode(_identity, _claimTopic, _data));
        bytes32 prefixedHash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", dataHash));
        
        address recovered = _recoverSigner(prefixedHash, _signature);
        return authorizedSigners[recovered];
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
