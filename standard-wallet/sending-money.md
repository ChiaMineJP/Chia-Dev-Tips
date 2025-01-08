# Sending money

* A sample of xch address (in testnet) is\
  txch19630v7t6huv47yehuwg0eauducux27zx72xc97nnt2dmdah4459syrf8ry
* The chia wallet address adopts `bech32m` as an encoding format.
  * Bech32m is originally proposed by BIP350 ([https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki](https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki))
* The maximum length of entire wallet address is 90 chars
* Chia wallet address can be divided into 2 parts by the right most '1'.
  * In the above example
    * txch1
      * as a hrp: human readable part. In mainnet, this is "xch1".
    * 9630v7t6huv47yehuwg0eauducux27zx72xc97nnt2dmdah4459syrf8ry
      * This is an encoded puzzle hash with checksum
* So, every wallet address is a puzzle hash in encoded format
* This is because in Chia, sending money is done by spending (consuming) a coin which you control to create a new coin which has the target puzzle hash.
  * A coin has only 3 pieces of information
    * parent coin id
    * puzzle hash
    * amount
  * When you want to send your money to someone, you need to know the wallet address (thus target puzzle hash) to send.
  * Once you know the target address (puzzle hash), you can create a new coin with the target puzzle hash and amount by consuming your coin
* So how you can spend a coin? In other words, what makes you the coin owner?
  * Actually, "owner" is not a right word in Chia world.
  * Because everyone can spend a coin if
    1. one knows the original puzzle (program/function) whose hashed value matches the puzzle hash of the unspent coin you want to spend.
       * Yes you read right, program can be uniquely hashed into an integer in Chia. This is the weapon brought by chialisp which expresses everything including program as a list!
    2. and one provides the right solution (program inputs / function arguments) to the puzzle
       * When executing a puzzle (function) with a solution (arguments), the program will fail or output a list of conditions.
       * If executing the program with the solution doesn't fail and does output a list of conditions which are all valid, spending a coin will be successful.
    3. and one provides correct signatures optionally requested by the conditions on valid spend.
  * For standard spend (sending xch/txch), what makes you the "unique" coin owner is the ability to create a signature of a coin id (hash of parent\_coin\_id, puzzle\_hash, amount) and a genesis id (id of network like mainnet/testnet) with a private key corresponding to the public key embedded into the standard puzzle hash.

#### Puzzle of the standard transaction

{% tabs %}
{% tab title="Chialisp (omit comments)" %}
{% code overflow="wrap" fullWidth="true" %}
```
(mod
  (SYNTHETIC_PUBLIC_KEY original_public_key delegated_puzzle solution)

  (defmacro assert items
    (if (r items)
        (list if (f items) (c assert (r items)) (q . (x)))
        (f items)
    )
  )

  (include condition_codes.clib)

  (defun sha256tree1
    (TREE)
    (if (l TREE)
        (sha256 2 (sha256tree1 (f TREE)) (sha256tree1 (r TREE)))
        (sha256 1 TREE)
    )
  )

  (defun-inline is_hidden_puzzle_correct (SYNTHETIC_PUBLIC_KEY original_public_key delegated_puzzle)
    (=
      SYNTHETIC_PUBLIC_KEY
      (point_add
        original_public_key
        (pubkey_for_exp (sha256 original_public_key (sha256tree1 delegated_puzzle)))
      )
    )
  )

  (defun-inline possibly_prepend_aggsig (SYNTHETIC_PUBLIC_KEY original_public_key delegated_puzzle conditions)
    (if original_public_key
        (assert
          (is_hidden_puzzle_correct SYNTHETIC_PUBLIC_KEY original_public_key delegated_puzzle)
          conditions
        )
        (c (list AGG_SIG_ME SYNTHETIC_PUBLIC_KEY (sha256tree1 delegated_puzzle)) conditions)
    )
  )

  (possibly_prepend_aggsig
    SYNTHETIC_PUBLIC_KEY original_public_key delegated_puzzle
  (a delegated_puzzle solution))
)
```
{% endcode %}
{% endtab %}

{% tab title="Chialisp (with comments)" %}
{% code overflow="wrap" fullWidth="true" %}
```
; build a pay-to delegated puzzle or hidden puzzle
; coins can be unlocked by signing a delegated puzzle and its solution
; OR by revealing the hidden puzzle and the underlying original key

; glossary of parameter names:

; hidden_puzzle: a "hidden puzzle" that can be revealed and used as an alternate
;   way to unlock the underlying funds
;
; synthetic_key_offset: a private key cryptographically generated using the hidden
;   puzzle and as inputs `original_public_key`
;
; SYNTHETIC_PUBLIC_KEY: the public key that is the sum of `original_public_key` and the
;   public key corresponding to `synthetic_key_offset`
;
; original_public_key: a public key, where knowledge of the corresponding private key
;   represents ownership of the file
;
; delegated_puzzle: a delegated puzzle, as in "graftroot", which should return the
;   desired conditions.
;
; solution: the solution to the delegated puzzle


(mod
  ; A puzzle should commit to `SYNTHETIC_PUBLIC_KEY`
  ;
  ; The solution should pass in 0 for `original_public_key` if it wants to use
  ; an arbitrary `delegated_puzzle` (and `solution`) signed by the
  ; `SYNTHETIC_PUBLIC_KEY` (whose corresponding private key can be calculated
  ; if you know the private key for `original_public_key`)
  ;
  ; Or you can solve the hidden puzzle by revealing the `original_public_key`,
  ; the hidden puzzle in `delegated_puzzle`, and a solution to the hidden puzzle.

  (SYNTHETIC_PUBLIC_KEY original_public_key delegated_puzzle solution)

  ; "assert" is a macro that wraps repeated instances of "if"
  ; usage: (assert A0 A1 ... An R)
  ; all of A0, A1, ... An must evaluate to non-null, or an exception is raised
  ; return the value of R (if we get that far)

  (defmacro assert items
    (if (r items)
        (list if (f items) (c assert (r items)) (q . (x)))
        (f items)
    )
  )

  (include condition_codes.clib)

  ;; hash a tree
  ;; This is used to calculate a puzzle hash given a puzzle program.
  (defun sha256tree1
    (TREE)
    (if (l TREE)
        (sha256 2 (sha256tree1 (f TREE)) (sha256tree1 (r TREE)))
        (sha256 1 TREE)
    )
  )

  ; "is_hidden_puzzle_correct" returns true iff the hidden puzzle is correctly encoded

  (defun-inline is_hidden_puzzle_correct (SYNTHETIC_PUBLIC_KEY original_public_key delegated_puzzle)
    (=
      SYNTHETIC_PUBLIC_KEY
      (point_add
        original_public_key
        (pubkey_for_exp (sha256 original_public_key (sha256tree1 delegated_puzzle)))
      )
    )
  )

  ; "possibly_prepend_aggsig" is the main entry point

  (defun-inline possibly_prepend_aggsig (SYNTHETIC_PUBLIC_KEY original_public_key delegated_puzzle conditions)
    (if original_public_key
        (assert
          (is_hidden_puzzle_correct SYNTHETIC_PUBLIC_KEY original_public_key delegated_puzzle)
          conditions
        )
        (c (list AGG_SIG_ME SYNTHETIC_PUBLIC_KEY (sha256tree1 delegated_puzzle)) conditions)
    )
  )

  ; main entry point

  (possibly_prepend_aggsig
    SYNTHETIC_PUBLIC_KEY original_public_key delegated_puzzle
  (a delegated_puzzle solution))
)
```
{% endcode %}
{% endtab %}

{% tab title="JavaScript" %}
{% hint style="warning" %}
Actually there are no official JS runtime/compiler for clvm available. So this JS code sample is just a reference for anyone familiar with JavaScript.
{% endhint %}

{% hint style="warning" %}
This is just a mental model for the chialisp code.
{% endhint %}

{% code overflow="wrap" %}
```javascript
function puzzle_standard_transaction (
  SYNTHETIC_PUBLIC_KEY, // byteArray
  original_public_key, // null or byteArray
  delegated_puzzle, // a function
  solution // arguments of `delegated_puzzle`
) {
  // `puzzle_output` should be a list of conditions
  // e.g. [[CREATE_COIN, puzHash, amount], ...]
  const puzzle_output = delegated_puzzle(solution);
  
  if (original_public_key) {
    if(!is_hidden_puzzle_correct(
          SYNTHETIC_PUBLIC_KEY,
          original_public_key,
          delegated_puzzle
        )
    ){
      throw new Error();
    }
    
    return puzzle_output;
  }
  else {
    return [
      // `sha256tree` will be defined in the following code
      [AGG_SIG_ME, SYNTHETIC_PUBLIC_KEY, sha256tree(delegated_puzzle)],
      ...puzzle_output,
    ];
  }
}

function is_hidden_puzzle_correct(
  SYNTHETIC_PUBLIC_KEY,
  original_public_key,
  delegated_puzzle
){
  // `sha256` is an predefined func which combines bytes of arguments
  // and hash with sha256.
  const challenge = sha256(
    original_public_key,
    sha256tree(delegated_puzzle)
  );
  // `pubkey_for_exp` is predefined func
  const offset = pubkey_for_exp(challenge);
  // `poin_add` is predefined func.
  const challenging_pk = point_add(original_public_key, offset);
  
  return challenging_pk === SYNTHETIC_PUBLIC_KEY;
}

function sha256tree(
  atom_or_pair // a byte value or a list(array)
){
  if(Array.isArray(atom_or_pair)){
    return sha256(
      2,
      sha256tree(atom_or_pair[0]), // Call `sha256tree` recursively
      sha256tree(atom_or_pair[1]) // Call `sha256tree` recursively
    );
  }
  return sha256(1, atom_or_pair);
}

// Currying in (pre-allocate) SYNTHETIC_PUBLIC_KEY
function create_puzzle_for_synthetic_pub_key(
  SYNTHETIC_PUBLIC_KEY, // byteArray
){
  return function(
    original_public_key,
    delegated_puzzle,
    solution
  ){
    return puzzle_standard_transaction(
      SYNTHETIC_PUBLIC_KEY,
      original_public_key,
      delegated_puzzle,
      solution
    );
  };
}
```
{% endcode %}
{% endtab %}
{% endtabs %}

* Official implementation of standard wallet sets empty `original_public_key` , so actually puzzle of standard transaction can be extremely simplified
  * Get arbitral conditions by running `(a delegated_puzzle solution)`
  * Append `(AGG_SIG_ME SYNTHETIC_PUBLIC_KEY sha256tree(delegated_puzzle)` to the conditions `(a puzzle solution)`generates
* You may notice that `delegated_puzzle` is included as an input of `AGG_SIG_ME` while `solution` is not.
* This means the content of `delegated_puzzle`is guaranteed by the signature while `solution` is not.
* So, in almost cases standard wallet puts every condition it wants to output into `delegated_puzzle`and set `solution`to `()`(empty).
  * delegated\_puzzle: `(list (list CREATE_COIN puzHash amount) ...)`&#x20;
  * solution: `()`
* But why not include solution to `AGG_SIG_ME`?
  * Because there are situations where the output of `(a delegated_puzzle solution)` wants to be controlled partially by the signature (thus the owner of embedded public key) and partially by other entity like `ASSERT_PUZZLE_ANNOUNCEMENT`
  * The fact `solution` is not included in `AGG_SIG_ME` means that `solution` could be forged/replaced by malicious middle men.
  * But we can prevent unexpected forging by
    * Checking allowed content of solution in `delegated_puzzle`
    * Validating the output of `(a delegated_puzzle solution)` by other coin's `ASSERT_XXX_ANNOUNCEMENT` / `RECEIVE_MESSAGE`&#x20;
      * In this case, it allows forging but in a limited/expected way. But it's OK because what actually important to blockchain is the output conditions.
  * So by not including `solution` to `AGG_SIG_ME` in the standard puzzle, it leaves a margin for non-signers to have the ability to control the output conditions (in other words, how to spend a coin).

