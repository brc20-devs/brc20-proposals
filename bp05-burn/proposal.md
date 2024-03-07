## Consensus on Burn Method

There is no specific consensus on the destruction of BRC-20; it is generally believed that transferring the token to an address with a publicly recognized unknown private key constitutes destruction. We hope to recommend a universal method for the destruction of BRC-20 tokens, which is to:

Send the `TRANSFER` transaction to an `OP_RETURN` script that contains no data, to achieve destruction.

There are numerous advantages to this method of destruction for tokens:
* `OP_RETURN` transactions can be proven to be unspendable within the Bitcoin protocol.
* It is the least expensive method since the `OP_RETURN` script only requires a single `6a` byte and merely needs an inclusion of 1 satoshi.

The downside of this approach is the lack of widespread consensus and the absence of ready-made tools to facilitate this destruction process.

Using conventional special addresses for token destruction would result in significant waste.
