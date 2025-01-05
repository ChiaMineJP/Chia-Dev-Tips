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
  * Coin can be spent only if
    * you know the original puzzle (program/function) whose hashed value matches the puzzle hash of the unspent coin you want to spend.
    * you provide the right solution (program inputs / function arguments) to the puzzle
      * When executing a puzzle (function) with a solution (arguments), the program will fail or output a list of conditions.
      * If executing the program with the solution doesn't fail and does output a list of conditions which are all valid, spending a coin will be successful.
    * you provide correct signatures optionally requested by the conditions on valid spend.

