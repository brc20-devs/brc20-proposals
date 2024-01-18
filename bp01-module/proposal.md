# Abstract

This proposal describes a module mechanism for brc-20.

The module mechanism provides a new approach to support various inscription-based applications within the existing brc-20 framework. Modules operate independently of each other, and each Indexer only needs to parse data relevant to their interested module while still maintaining brc-20 balance consistency with other indexers.

# Motivation

The brc-20 itself is a very simple payment system, defining three operations for deployment, minting, and transferring tokens.

We realize that brc-20 also has great potential expansion capabilities and can support a variety of applications running in this basic payment system.

# Description

The module mechanism is a simple way to introduce various applications into brc-20 and ensure their harmonious coexistence.

By deploying a module, an independent space is created within the brc-20 world. Users can deposit any brc-20 tokens into the module and withdraw their brc-20 balance from it.

Users send valid brc-20 transfer inscriptions to the module's ID address to deposit their balance into the module.

In the module, users can generally only use the tokens they have deposited. Users can interact with the module using inscriptions or any other feasible method.

Module designers can define rules at their discretion, such as requiring confirmation of deposits. To facilitate consensus among indexers, a "contract" source code inscription ID must be specified when deploying a module. This content describes the logic of the module's functionality in code but does not require the inscription to exist at the time of module deployment.

From the perspective of the indexer, modules are divided into two types: ignorable and non-ignorable, which we name black module and white module:

### Black Module

Black modules only support deposits and do not support withdrawals. From the perspective of an indexer that does not parse it, brc-20 tokens appear to have been transferred to an address that only receives but does not send out.

### White Module

White modules can support withdrawals but require that all indexers support the module mechanism and implement the consensus algorithm for this module. Withdrawals are illegal before a module transforms into a white module.

A black module can transform into a white module once accepted by all indexers, but a white module can not transform back into a black module.

Consensus can be reached by a vote of all participants to decide whether a module created from a source inscription should transform into a white module.

# Operations

The module mechanism introduces two new inscriptions: `deploy` and `withdraw`, in addition to directly using the brc-20 transfer inscription to deposit tokens.

## Deploying a New Module

The lower case inscription ID serves as the module's ID. You need to set the name, source, and optional initialization data. There is no character limit for the name.

```
{
  "p": "brc20-module",
  "op": "deploy",
  "name": "demo",
  "source": "d2a30f6131324e06b1366876c8c089d7ad2a9c2b0ea971c5b0dc6198615bda2ei0",
  "init": {
      "fee": "0.01"
  }
}
```

## Depositing BRC-20 Tokens into the Module (Deposit by Transfer)

The module's ID address is in the form of an `OP_RETURN ModuleID` locking script address. To indexers that do not support modules, the balance appears to have been transferred to the module's literal address, and no one can unlock it.

```
{
  "p": "brc-20",
  "op": "transfer",
  "tick": "ordi",
  "amt": "10"
}
```

The module ID after `OP_RETURN` is serialized as the 32-byte `TXID`, followed by the four-byte little-endian `INDEX`, with trailing zeroes omitted.

## Withdrawing BRC-20 Tokens from the Module to a Target Address (Withdraw)

* Similar to the transfer inscription, users need to first create a Withdraw inscription and then transfer it to the target address. The inscription can only be used once.

* The user's own brc-20 balance in the module will be withdrawn to the target address.

* The content of inscription must include the module's ID. Withdrawals are only valid if the module exists and is a white module.

```
{
  "p": "brc20-module",
  "op": "withdraw",
  "tick": "ordi",
  "amt": "10",
  "module": "8082283e8f0edfcce4901ff2d271b2eec4a4735b371bb6af53d64770651d8c14i0"
}
```

# FAQ

### Can anyone deploy a module?

* Yes, anyone can define the functionality of a module and deploy an instance of it.

* All other indexers will consider modules with unrecognized source IDs as black modules by default. Deployers must run their own module's indexer.

### After depositing into a black module, will you be unable to withdraw?

Although only white modules support withdrawals, black modules can easily support atomic exchange transactions between internal and external balances, allowing users who want to deposit into the module to exchange balances with users who want to withdraw. This way, users inside the module do not need to worry about being unable to withdraw.

### Can the source code inscription-id change?

* Currently, it cannot be changed and must be specified by the module author during deployment.

### Is it possible to upgrade module code if there are bugs?

* For black modules, indexers can be upgraded to fix bugs.

* For white modules, if bugs are discovered in the code, a new version of the module with the bug fix can be deployed, and tokens can be transferred to the new version of the module. However, new versions of modules generally run in black mode first.

# Acknowledgments

Domo originally mentioned the idea for the module, and the implementation was explored and attempted through multiple versions at UniSat. Thanks to Domo for his contributions, and thanks to everyone for reading.

# Copyright

This proposal is licensed under the MIT License.
