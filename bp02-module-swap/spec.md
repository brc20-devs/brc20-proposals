# brc20-swap specification

## Table of Contents

- [Summary](#summary)
- [OP](#op)
  - [Deploy](#deploy)
  - [Deposit](#deposit)
  - [Withdraw](#withdraw)
  - [Approve](#approve)
  - [Conditional-Approve](#conditional-approve)
- [Sub-OP](#sub-op)
  - [DeployPool](#deploypool)
  - [AddLiquidity](#addliquidity)
  - [RemoveLiquidity](#removeliquidity)
  - [Swap](#swap)
  - [DecreaseApproval](#decreaseapproval)
  - [Send](#send)
- [Source](#source)
- [Formulas](#formulas)
  - [swap](#swap-1)
  - [addLiq](#addliq)
  - [removeLiq](#removeliq)
  - [slippage](#slippage)
  - [platform fee](#platform-fee)

## Summary

This document defines a BRC20 Swap Module.

Based on the ordinals script system and the model of Uniswap V2, we have implemented a module that allows for smooth and secure BRC20 token swaps.

- User balances are divided into three categories: BRC20 balance, module balance, and approved balance. We have defined a set of OPs to facilitate the transfer of balances between these categories.
  - `deposit` allows users to transfer BRC20 balances to the approved balance.
  - `approve` and `conditional-approve` allows users to transfer module balances to the approved balance.
  - `withdraw` allow users to withdraw module balances to the BRC20 balance.
- Once the user's balance is in the approved balance, they can perform swap operations, which will be executed by the sequencer through a commit operation.
  - We have defined a set of Sub-OPs that allow users to create and operate trading pairs using `DeployPool`, `AddLiquidity`, `RemoveLiquidity`, and `Swap`.
  - As the sequencer is responsible for the final script forging and on-chain execution, the sequencer will charge a certain amount of sats for each Sub-OP to cover their costs.

## OP

This proposal includes the following ops: `deploy`, `deposit(transfer)`, `withdraw`, `approve`, `conditional-approve`, `commit`

### Deploy

Deploy a brc20-module instance

- gas_tick: The swap fee rate, which is collected as a fee and added to the pool for liquidity providers.
- gas_to: The BRC20 token used as fees during aggregation. Required.
- fee_to: Address to collect LP issuance fees. Must be specified.
- sequencer: Address of the sequencer. Must be specified.

```json
{
  "p": "brc20-module",
  "op": "deploy",
  "name": "swap",
  "source": "d2a30f6131324e06b1366876c8c089d7ad2a9c2b0ea971c5b0dc6198615bda2ei0",
  "init": {
    "swap_fee_rate": "0.003",
    "gas_tick": "sats",
    "gas_to": "bc1q...",
    "fee_to": "bc1q...",
    "sequencer": "bc1q...."
  }
}
```

### Deposit

Deposit BRC20 tokens into the module.

Instead of introducing a new OP, the transfer OP is reused.

The module ID address is the locking script address in the form of `OP_RETURN ModuleID`. To indexers that do not support modules, it appears that the balance has been transferred to the module and cannot be unlocked by anyone.

The module ID after OP_RETURN needs to be binary serialized. The serialization method involves including the 32-byte original TXID, followed by a 4-byte little-endian encoding of the INDEX, and finally removing any trailing 0 bytes from the INDEX.

After depositing, it will be automatically approved for the swap module.

```json
{
  "p": "brc-20",
  "op": "transfer",
  "tick": "ordi",
  "amt": "10"
}
```

### Withdraw

Withdraw BRC20 tokens from the module to a target address

- Users need to first inscribe a Withdraw inscription and then transfer it to the target address to take effect. The inscription can only be used once.
- BRC20 tokens owned by the user in the module will be withdrawn to the target address.
- The inscription must include the module ID. If the module does not exist, the withdraw operation is invalid.
- Only white modules support withdrawals. Users must have withdrawable balances in the module.

```json
{
  "p": "brc20-module",
  "op": "withdraw",
  "tick": "ordi",
  "amt": "10",
  "module": "66801a4a8352e84ed8485ec231aee88c20983bf442aa04d398ac6c89c92abc8ci0"
}
```

### Approve

Authorize the BRC20 token balance in the module to the target address unconditionally.

- Users need to first inscribe an Approve inscription and then transfer it to the target address to take effect. The inscription can only be used once.
- The balance will be allocated to the authorized balance of the specified address in the module.
- The inscription must include the module ID. If the module does not exist, the Approve operation is invalid.
- The commit operation mentioned later can only use the authorized balance of the token.
- It is recommended that the sequencer wait for at least 3 confirmations before allowing users to operate this portion of the balance to avoid the impact of block reorganization.

```json
{
  "p": "brc20-swap",
  "op": "approve",
  "tick": "ordi",
  "amt": "10",
  "module": "66801a4a8352e84ed8485ec231aee88c20983bf442aa04d398ac6c89c92abc8ci0"
}
```

### Conditional-Approve

Authorize the conditional balance of BRC20 tokens in the module to the target address.

- Users need to first inscribe a Conditional-Approve inscription to lock the specified module balance.
- Users transfer the inscription to an address S different from their own, which acts as a proxy for processing. The inscription can be used multiple times, but each subsequent transfer must maintain S as the holder. Otherwise, the remaining locked balance will be returned to the user's authorized balance.
- When the transfer is made by address S, if a valid brc20 transfer inscription is also sent to the user's address, the Approve inscription will return an equal amount of module balance to the sender's address in the module's authorized balance. Until the balance is exhausted, the Approve inscription will be useless.
- There is no limit to the number of transfer and conditional-approve inscriptions in a transaction.
- The inscription must include the module ID. If the module does not exist, the Approve operation is invalid.
- The commit event functionality mentioned later can only use the authorized balance of the token.
- It is recommended that the sequencer wait for at least 3 confirmations before allowing users to operate this portion of the balance to avoid the impact of block reorganization.

```json
{
  "p": "brc20-swap",
  "op": "conditional-approve",
  "tick": "ordi",
  "amt": "10",
  "module": "66801a4a8352e84ed8485ec231aee88c20983bf442aa04d398ac6c89c92abc8ci0"
}
```

### Commit

All events that occur in the swap pool are described using a set of functions. To ensure the order of swap events, these functions are not individually inscribed, but instead included in a module's aggregate inscription (commit).

The basic rules are as follows:

- When aggregating, each operation must be signed by the address of the operator, covering all preceding content (serialized, including module and parent).
- The aggregate inscription must be inscribed and transferred to the module ID address by the module sequencer to take effect. The sequencer is the initial recipient address for inscribing the module's inscriptions. The transfer of sequencer's authority is yet to be determined.
- The module sequencer must charge a fee to the user for each function aggregated, based on the number of bytes. Users must deposit and authorize enough fee tokens in the module. Otherwise, the sequencer must reject the user's event request. Insufficient user fees will result in an invalid commit inscription. The length of a function refers to the number of bytes from "{" to "}" in the inscription, excluding the comma after "}".

To avoid blocking when sending a large number of commit inscriptions due to the maximum 100KB memory pool and the maximum 25 descendants limit, commit inscriptions can be initially inscribed to the sequencer using unrelated UTXOs in any order. The sequencer can then periodically transfer a commit inscription to the module address in logical order, ensuring the effectiveness of all preceding logical commit inscriptions. Since the transfer is a simple and small transaction, periodic transfer of commits will greatly reduce the number of commit inscriptions to be transferred, and the 25 limit is expected to no longer significantly hinder the effectiveness of commit inscriptions.

1. The keys in the inscription fields must be lowercase characters, and the field values are case-sensitive except for "tick".
2. Duplicate fields are allowed, and additional fields are allowed.

```json
{
  "p": "brc20-swap",
  "op": "commit",
  "module": "66801a4a8352e84ed8485ec231aee88c20983bf442aa04d398ac6c89c92abc8ci0",
  "parent": "xxxxi0", // The commit inscription state to be appended after, leave empty if it is the first commit inscription
  "gas_price": "100", // The fee amount charged per byte for each function
  "data": [
    {
      "addr": "bc1q...",
      "func": "deployPool",
      "params": ["ordi", "pepe"],
      "ts": 12345,

      "sig": "xxx"
    },
    {
      "addr": "bc1q...",
      "func": "addLiq",
      "params": ["ordi/pepe", "100", "200", "100", "0.005"],
      "ts": 12345,

      "sig": "xxxx"
    },
    {
      "addr": "bc1q...",
      "func": "swap",
      "params": ["ordi/pepe", "ordi", "200", "exactIn", "12.324", "0.005"],
      "ts": 12345,

      "sig": "xxxx"
    }
  ]
}
```

#### User Signature Authorization

Each function in the aggregate inscription of the commit requires the user's address and signature to authorize the sequencer for operation.

In the initial stage, only taproot and p2wpkh address formats are supported for the address. Other address formats such as p2wsh, p2sh, p2pkh, or any locking script encoded in hex will be gradually opened in the future.
The signature follows the bip-322 signature format for the following content:

```yaml
id: idxxxx
addr: bc1q...
func: addLiq
params: ordi/pepe 100 200 12.324
ts: 12345
```

Each function has a function hashid, which is calculated as sha256(msg), where msg is the serialized content as follows:

```yaml
module: 66801a4a8352e84ed8485ec231aee88c20983bf442aa04d398ac6c89c92abc8ci0
parent: idxxxxi0
quit: idxxxxi0
gas_price: 100
prevs: id1 id2 id3
addr: bc1q...
func: addLiq
params: ordi/pepe 100 200 12.324
ts: 12345
```

Note: List the data for each field in the order shown above, ending each line with a newline character '\n'. Separate the parameters in the params field with spaces. If a field has no content, exclude that line from the serialized data.

The function id is a fully computable intermediate result that does not need to be stored on-chain after testing. The prevs field contains all the function ids submitted by the same user in the current commit inscription, sorted in the order accepted by the sequencer.

In the entire commit inscription, only the last signature of each address is stored on-chain, and the intermediate function data is overwritten by the last signature.

## Sub-OP

Sub-OP is an OP that can only be used in the commit inscription.
It supports the following OPs: `DeployPool`, `AddLiquidity`, `RemoveLiquidity`, `Swap`, `DecreaseApproval`, `Send`.

### DeployPool

Deploy a trading pair

Specify the two tokens, token0 and token1, to participate in the swap pool. They must be different and have no order relationship.

The same pair of tokens can only be deployed once within a module.

The liquidity token name is "token0/token1". It is not supported to use a liquidity token to deploy another liquidity pool.

The liquidity token is always restricted within the module that deployed it and cannot be withdrawn. The module records the distribution of liquidity token balances for each pool.

Anyone can deploy a liquidity pool, but the deployer does not have any additional privileges.

```json
{
  "func": "deployPool",
  "params": [
    "ordi", // token0
    "pepe" // token1
  ],
  "addr": "bc1q...",
  "ts": 12345,
  "sig": "xxx"
}
```

### AddLiquidity

Add liquidity, can only operate on authorized module balances.

```json
{
  "func": "addLiq",
  "params": [
    "ordi/pepe", // swap pool name. It can be written as pepe/ordi, and the subsequent parameters will correspond to this token order
    "100", // The amount of ordi injected by the user
    "200", // The amount of pepe injected by the user

    "100", // The amount of LP tokens the user expects to receive
    "0.005" // Slippage
  ],
  "addr": "bc1q...",
  "ts": 12345,
  "sig": "xxx"
}
```

### RemoveLiquidity

Remove liquidity, the obtained tokens belong to the authorized module balances.

```json
{
  "func": "removeLiq",
  "params": [
    "ordi/pepe", // The pair ticker. It can be written as pepe/ordi, and the subsequent parameters will correspond to this token order
    "100", // The amount of LP tokens withdrawn by the user

    "12.324", // The expected amount of ordi to receive
    "321.3", // The expected amount of pepe to receive
    "0.005" // Slippage
  ],
  "addr": "bc1q...",
  "ts": 12345,
  "sig": "xxxx"
}
```

### Swap

Perform a swap exchange, can only be done using authorized balance, and the obtained tokens belong to the authorized balance.

```json
{
  "func": "swap",
  "params": [
    "ordi/pepe", // Ther pair ticker
    "ordi", // The ticker of token to spent
    "200", // The amount of token spent
    "exactIn", //  exactIn/exactOut

    "12.324", // The expected amount of the other token to receive (exactIn) or pay (exactOut)
    "0.005" // Slippage
  ],
  "addr": "bc1q...",
  "ts": 12345,
  "sig": "xxxx"
}
```

### DecreaseApproval

Decrease the approved amount of a token. Increase the withdrawable amount.
This will transfer the balance from the authorized area to the module area.

```json
{
  "func": "decreaseApproval",
  "params": [
    "ordi", // tick
    "10" // amount
  ],
  "addr": "bc1q...",
  "ts": 12345,
  "sig": "xxx"
}
```

### Send

Balances of various tokens are sent within the module. Only the authorized amount can be sent, and the recipient's balance also becomes authorized. Liquidity token balances can be sent using the `send` function.

```json
{
  "func": "send",
  "params": [
    "bc1q...", // target address
    "ordi/pepe", // tick
    "10" // amount
  ],
  "addr": "bc1q...",
  "ts": 12345,
  "sig": "xxx"
}
```

## Source

The Source script explains the actual operational logic of the swap module.
By using the Source script as input, the swap contract performs calculations to determine the current swap result.

```typescript
class SwapContract{
    function deployPool(params:{ token0: string; token1: string }){
    // todo
    }

    function addLiq(params: {
        token0: string;
        token1: string;
        amount0: string;
        amount1: string;
        address: string;
    }){
        // todo
    }

    function removeLiq(params:{liqAmount:string}){
        // todo
    }

    function swap(params:{
        token0: string;
        token1: string;
        token: string;
        amount: string;
        address: string;
    }){
        // todo
    }

    function withdraw(params:{token:string; amount:string}){
        // todo
    }

    function send(params:{address:string; token:string; amount:string}){
        // todo
    }


}


```

## Fee

### Swap Fee

During the swap process, a 0.3% fee is charged on the user's transaction amount, and 1/6 of the fee is allocated to the development team, while the remaining 5/6 is rewarded to liquidity providers.
Note: Assuming the user exchanges 1000 ordi for sats, when swapping, 1000 ordi is injected into the liquidity pool, and a fee of 3 ordi is charged. The actual amount of sats received by the user is calculated based on 997 ordi as input.

### Network Fee

The sequencer is responsible for collecting each aggregation operation. After summarizing, it will forge the aggregated inscription on the chain. During this process, a certain amount of network fee will be generated to be used by miners for packaging. We can calculate the network fee charged to users based on the current network fee rate and the user's aggregation operations using the formula:

$$fee = gas \cdot gasPrice$$

- gas: the number of bytes occupied by each aggregation operation on the chain. Taking the deployPool aggregation operation as an example, the number of bytes calculated after JSON serialization is 82.

```json
{
  "func": "deployPool",
  "params": ["ordi", "pepe"],
  "addr": "xxx",
  "ts": 12345,
  "sig": "xxx"
}
```

- gasPrice: The network fee charged per byte, determined by two factors, the BTC network fee rate (a) and the market price (b) for charging Ticks:
  $$gasPrice = a / b$$
- fee: measured in Ticks collected

## Formulas

### swap

- Assumptions
  - amountIn: Input token amount
  - amountOut: Output token amount
  - reserveIn: Input token reserve in the pool
  - reserveOut: Output token reserve in the pool
  - Assuming a 0.3% transaction fee

exactIn

- $$amountInWithFee = amountIn \cdot 997$$
- $$amountOut = \frac{amountInWithFee \cdot reserveOut }{reverseIn \cdot 1000 + amountInWithFee}  $$

exactOut

- $$amountIn = \frac{reserveIn \cdot amountOut \cdot 1000 }{(reserveOut - amountOut) \cdot 997}  + 1$$

### addLiq

- Assumptions
  - amount0: Amount of token 0 to add
  - amount1: Amount of token 1 to add
  - poolLp: Number of LP tokens in the pool
  - reserve0: Reserve of token 0 in the pool
  - reserve1: Reserve of token 1 in the pool
- Initial liquidity addition: $$lp = \sqrt{amount0 \cdot amount1} - 1000$$
- Adding liquidity later: $$lp = \min(\frac{amount0 \cdot poolLp}{reserve0}, \frac{amount1 \cdot poolLp}{reserve1} ) $$

### removeLiq

- Get the quantity of token 0: $$amount0 = \frac{lp \cdot reserve0}{poolLp}$$
- Get the quantity of token 1: $$amount1 = \frac{lp \cdot reserve1}{poolLp}$$

### slippage

- exactIn: $$1/(1+slippage) \cdot quoteAmount$$
- exactOut: $$(1+slippage) \cdot quoteAmount$$

### platform fee

- Assuming the platform collects 1/6 of the fees, we need to calculate how many LP tokens the platform should mint in order to receive exactly 1/6 of the incremental wealth.
- Trigger: Calculate the accumulated amount of LP tokens to be minted before each liquidity addition/removal calculation.
- poolLp: Number of LP tokens in the pool
- reserve0: Reserve of token 0 in the pool
- reserve1: Reserve of token 1 in the pool
- kLast: The value of k calculated after the previous liquidity addition/removal.
- RootK calculated for the current pool assets:$$rootK = \sqrt{reserve0 \cdot reserve1}$$
- RootK calculated for the previous liquidity addition/removal: $$rootKLast = \sqrt{kLast}$$
- Platform Minting: $$lp = \frac{poolLp \cdot (rootK - rootKLast)}{rootK \cdot 5 + rootKLast  } $$
