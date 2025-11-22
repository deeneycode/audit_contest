## Next Generation
[Contest Details] (https://code4rena.com/reports/2025-10-sequence)

### [Medium-01] Missing wallet binding in session call hash enables cross-wallet replay of session signatures

**Finding Description and Impact**
Session signatures are computed over (chain id, space, nonce, call index, call data) but not the target wallet address. Because the same session image/config can be reused across wallets, a valid session signature for Wallet A can be replayed and accepted by Wallet B. This allows an attacker who obtains a session signature for one wallet to execute the same session-authorized call on other wallets sharing the same session image, effectively escalating privileges across wallets.

Write a detailed description of the root cause and impact(s) of this finding.
https://github.com/code-423n4/2025-10-sequence/blob/b0e5fb15bf6735ec9aaba02f5eca28a7882d815d/src/extensions/sessions/SessionSig.sol#L406
```javascript
 function hashCallWithReplayProtection(
    Payload.Decoded calldata payload,
    uint256 callIdx
  ) public view returns (bytes32 callHash) {
    return keccak256(
      abi.encodePacked(
        payload.noChainId ? 0 : block.chainid,
        payload.space,
        payload.nonce,
        callIdx,
        Payload.hashCall(payload.calls[callIdx])
      )
    );
  }
```
The preimage for the signed call is assembled using:

- chain id (conditionally),
- payload.space,
- payload.nonce,
- callIdx,
- and the hashed call payload.
There is no inclusion of the executing wallet's identity (for example, the wallet address that will accept and act upon the session signature). Because the wallet address is omitted, the same (chain id, space, nonce, callIdx, call data) tuple maps to identical callHash values across different wallets.

`SessionSig.recoverSignature` (and all call-site uses of `hashCallWithReplayProtection`) thus accept signatures that were created without being bound to a specific wallet, allowing replay across wallets that share the same session image/configuration.
**Impact**

- Cross-wallet replay: a session signature generated for one wallet (Wallet A) can be used to authorize the exact same call on any other wallet (Wallet B) that shares the same session image/config, same chain id/space/nonce and matching permissions.
- Privilege escalation: attackers who capture or coerce a single session signature (e.g., by social engineering, compromised client, or leaked signatures) can replay that signature across multiple wallets to perform actions those wallets' owners never intended.
- Breaks the intended scoping of session signatures: session authorizations should be scoped to a specific wallet execution context; without wallet binding they are global to the image/space/nonce tuple.
**Proof of Concept**
Here is a coded proof of concept written in foundry, to run the test a foundry environment should be set up.

```javascript
function test_SessionSignature_ReplayAcrossWallets_succeeds() public {
    // Arrange: create topology and first wallet
    string memory topology = _createDefaultTopology();
    (Stage1Module wallet1, string memory config, bytes32 imageHash) = _createWallet(topology);

    // Deploy a second factory and deploy a second wallet with the same imageHash (different deployer)
    Factory factory2 = new Factory();
    Stage1Module wallet2 = Stage1Module(payable(factory2.deploy(address(module), imageHash)));

    // Build a simple payload (single call)
    Payload.Decoded memory payload = _buildPayload(1);
    payload.calls[0].to = address(mockTarget);

    // Sign the session call once with the session signer
    string[] memory callSignatures = new string[](1);
    bytes32 callHash = SessionSig.hashCallWithReplayProtection(payload, 0);
    string memory sessionSignature = _signAndEncodeRSV(callHash, sessionWallet);
    callSignatures[0] = _explicitCallSignatureToJSON(0, sessionSignature);

    address[] memory explicitSigners = new address[](1);
    explicitSigners[0] = sessionWallet.addr;
    address[] memory implicitSigners = new address[](0);

    bytes memory sessionSignatures =
      PrimitivesRPC.sessionEncodeCallSignatures(vm, topology, callSignatures, explicitSigners, implicitSigners);
    string memory signatures =
      string(abi.encodePacked(vm.toString(address(sessionManager)), ":sapient:", vm.toString(sessionSignatures)));
    bytes memory signature = PrimitivesRPC.toEncodedSignature(vm, config, signatures, !payload.noChainId);
    bytes memory packedPayload = PrimitivesRPC.toPackedPayload(vm, payload);

    // Demonstrate missing wallet binding in the signed preimage:
    // The call hash is independent of the wallet, so the same signature recovers the same session signer.
    bytes32 callHash1 = SessionSig.hashCallWithReplayProtection(payload, 0);
    bytes32 callHash2 = SessionSig.hashCallWithReplayProtection(payload, 0);
    assertEq(callHash1, callHash2);

    // Sign the call hash and recover the signer off-chain (vm.sign used above produced r/s/v)
    (uint8 v, bytes32 r, bytes32 s) = vm.sign(sessionWallet.privateKey, callHash1);
    address recovered = ecrecover(callHash1, v, r, s);
    assertEq(recovered, sessionWallet.addr);

    // The same callHash and signature will recover the same signer irrespective of which wallet will later use it
    // (i.e., the preimage does not include any wallet-specific address).
    bytes32 callHashForWallet2 = SessionSig.hashCallWithReplayProtection(payload, 0);
    (uint8 v2, bytes32 r2, bytes32 s2) = (v, r, s);
    address recovered2 = ecrecover(callHashForWallet2, v2, r2, s2);
    assertEq(recovered2, sessionWallet.addr);
  }
```

**Recommended Mitigation**
Bind the wallet address into the session call preimage used for signatures.