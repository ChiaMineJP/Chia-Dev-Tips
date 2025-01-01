# Summary

* BLS -> Boneh, Lynn, Shacham
  * BLS Signatures ≠ BLS12-381 Curve
  * BLS12-381 curve -> Barreto, Lynn, Scott
  * BLS12-381 is not a strict requirement for BLS signature
    * You can choose an elliptic curve other than BLS12-381 for BLS signature
    * In Chia Blockchain BLS12-381 curve is adopted for BLS signature
* The primary components are:
  * `sk`, Secret key (Private key)
  * `pk`, Public key
  * `sig`, Signature
* The primary features are:
  * Key generation
    * Turning a random integer to a `sk`
  * Signature
    * Signature is useless without the signing `message` and signing `public key`
    * A message is assured by a signer who posseses the `sk` corresponding to the `pk`
  * Verification
    * to verify that a `sig` is signed by the owner of `sk` for
  * Private key aggregation
  * Public key aggregation
  * Signature aggregation
  * Hierarchical Key Derivation
* Signature requires `sk`, `message`
* Signature verification requires `pk`, `message` and `signature`

***

### Code samples

```javascript
// ----------------------------------------
// CREATING KEYS AND SIGNATURE
// ----------------------------------------
var loadBls = require("bls-signatures");
var BLS = await loadBls();

var seed = Uint8Array.from([
  0,  50, 6,  244, 24,  199, 1,  25,  52,  88,  192,
  19, 18, 12, 89,  6,   220, 18, 102, 58,  209, 82,
  12, 62, 89, 110, 182, 9,   44, 20,  254, 22
]);

var sk = BLS.AugSchemeMPL.key_gen(seed);
var pk = sk.get_g1();

var message = Uint8Array.from([1,2,3,4,5]);
var signature = BLS.AugSchemeMPL.sign(sk, message);

let ok = BLS.AugSchemeMPL.verify(pk, message, signature);
console.log(ok); // true

// ----------------------------------------
// SERIALIZING KEYS AND SIGNATURES TO BYTES
// ----------------------------------------
var skBytes = sk.serialize();
var pkBytes = pk.serialize();
var signatureBytes = signature.serialize();

console.log(BLS.Util.hex_str(skBytes));
console.log(BLS.Util.hex_str(pkBytes));
console.log(BLS.Util.hex_str(signatureBytes));

// ----------------------------------------
// LOADING KEYS AND SIGNATURES FROM BYTES
// ----------------------------------------
var skc = BLS.PrivateKey.from_bytes(skBytes, false);
var pk = BLS.G1Element.from_bytes(pkBytes);

var signature = BLS.G2Element.from_bytes(signatureBytes);

// ----------------------------------------
// CREATE AGGREGATE SIGNATURES
// ----------------------------------------
// Generate some more private keys
seed[0] = 1;
var sk1 = BLS.AugSchemeMPL.key_gen(seed);
seed[0] = 2;
var sk2 = BLS.AugSchemeMPL.key_gen(seed);
var message2 = Uint8Array.from([1,2,3,4,5,6,7]);

// Generate first sig
var pk1 = sk1.get_g1();
var sig1 = BLS.AugSchemeMPL.sign(sk1, message);

// Generate second sig
var pk2 = sk2.get_g1();
var sig2 = BLS.AugSchemeMPL.sign(sk2, message2);

// Signatures can be non-interactively combined by anyone
var aggSig = BLS.AugSchemeMPL.aggregate([sig1, sig2]);

ok = BLS.AugSchemeMPL.aggregate_verify([pk1, pk2], [message, message2], aggSig);
console.log(ok); // true

// Pubkey and message order matters
not_ok = BLS.AugSchemeMPL.aggregate_verify([pk2, pk1], [message, message2], aggSig);
console.log(not_ok); // false

// ----------------------------------------
// SIGNING A MESSAGE WITH MULTIPLE KEYS
// ----------------------------------------
var sig_sk2_msg = BLS.AugSchemeMPL.sign(sk2, message)
var aggSig2 = BLS.AugSchemeMPL.aggregate([sig1, sig_sk2_msg]);
ok = BLS.AugSchemeMPL.aggregate_verify([pk1, pk2], [message, message], aggSig2);
console.log(ok); // true

// ----------------------------------------
// Private/Public key aggregation
// ----------------------------------------
var aggSk = BLS.PrivateKey.aggregate([sk1, sk2]);
var aggPk = pk1.add(pk2);
var sig3 = BLS.AugSchemeMPL.sign(aggSk, message);
ok = BLS.AugSchemeMPL.aggregate_verify([aggPk], [message], sig3);
console.log(ok); // true

// ----------------------------------------
// HD keys using EIP-2333
// ----------------------------------------
// You can derive 'child' keys from any key, to create arbitrary trees. 4 byte indeces are used.
// Hardened (more secure, but no parent pk -> child pk)
var masterSk = BLS.AugSchemeMPL.key_gen(seed);
var child = BLS.AugSchemeMPL.derive_child_sk(masterSk, 152);
var grandChild = BLS.AugSchemeMPL.derive_child_sk(child, 952);

// Unhardened (less secure, but can go from parent pk -> child pk), BIP32 style
var masterPk = masterSk.get_g1();
var childU = BLS.AugSchemeMPL.derive_child_sk_unhardened(masterSk, 22);
var grandchildU = BLS.AugSchemeMPL.derive_child_sk_unhardened(childU, 0);

var childUPk = BLS.AugSchemeMPL.derive_child_pk_unhardened(masterPk, 22);
var grandchildUPk = BLS.AugSchemeMPL.derive_child_pk_unhardened(childUPk, 0);

ok = (grandchildUPk.equal_to(grandchildU.get_g1()));
console.log(ok); // true
```
