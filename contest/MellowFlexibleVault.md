## Mellow Flexible Vaults
[Contest Details] (https://audits.sherlock.xyz/contests/964/report)

### [High-01] Duplicate Signatures Allow Single Signer to Bypass Valid Signer Threshold Requirement

**Summary**
The Consensus::_checkSignatures function verifies that a set of signatures meets the protocol's threshold and are from authorized signers, but does not enforce that each signature comes from a unique signer. This allows a single signer to submit their signature multiple times, bypassing the intended multi-signature threshold and unilaterally authorizing privileged actions.

**Root Cause**
https://github.com/sherlock-audit/2025-07-mellow-flexible-vaults/blob/main/flexible-vaults/src/permissions/Consensus.sol#L21

 /// @notice Internal storage layout for Consensus contract  
```javascript 
    struct ConsensusStorage {  
        EnumerableMap.AddressToUintMap signers; // Mapping of signer => SignatureType  
        uint256 threshold; // Required number of valid signatures  
    }  

function checkSignatures(bytes32 orderHash, Signature[] calldata signatures) public view returns (bool) {  
        ConsensusStorage storage $ = _consensusStorage();  
        if (signatures.length == 0 || signatures.length < $.threshold) {  
            return false;  
        }  
        for (uint256 i = 0; i < signatures.length; i++) {  
            address signer = signatures[i].signer;  
            (bool exists, uint256 signatureTypeValue) = $.signers.tryGet(signer);  
            if (!exists) {  
                return false;  
            }  
            SignatureType signatureType = SignatureType(signatureTypeValue);  
            if (signatureType == SignatureType.EIP712) {  
                address recoveredSigner = ECDSA.recover(orderHash, signatures[i].signature);  
                if (recoveredSigner == address(0) || recoveredSigner != signer) {  
                    return false;  
                }  
            } else if (signatureType == SignatureType.EIP1271) {  
                bytes4 magicValue = IERC1271(signer).isValidSignature(orderHash, signatures[i].signature);  
                if (magicValue != IERC1271.isValidSignature.selector) {  
                    return false;  
                }  
            } else {  
                return false;  
            }  
        }  
        return true;  
    }  
```    
The function iterates over the provided signatures, checking validity and authorization, but does not track or enforce uniqueness of the signer field. As a result, duplicate signatures from the same signer are counted toward the threshold.

**Internal Pre-conditions**
nil

**External Pre-conditions**
validateOrder function in the signatureQueue.sol contracts

**Attack Path**
- Protocol is configured with a threshold of 3 signers.
- Alice is an authorized signer.
- Alice signs the order 3 times and submits all 3 signatures in the array.
- Consensus::checkSignatures counts all 3 signatures, allowing Alice to execute any privileged action unilaterally.

**Impact**
- Any privileged action (SignatureRedeemQueue::redeem, SignatureDepositQueue:Deposit) can be executed by a single authorized signer, as long as they submit their signature enough times to meet the threshold.
- Multi-signature security is completely bypassed

**PoC**
nil

**Mitigation**
Sort the signatures array by signer address before the loop, then check that no two consecutive signers are the same.
