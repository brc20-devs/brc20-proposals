# Whitelist And Scripting

This doc proposes the following features:
- The option to enable access control for token operations, e.g.
  1. only whitelisted addresses can perform mint
  2. only certain block height has reached can one transfer token
  3. ...
- Upgradability of token state
- Potential extension to more generic scripting capabilities

This motivation for the above features comes from bitSmiley, a stablecoin protocol on top of bitcoin L1. The proposal was first 
experimented as `bitrc-20`, but the use cases can be extended to many other projects. This doc will first explain the motivation
behind the above features and then proceed to discuss the high-level architecture to achieve them.

## Motivation
bitSmiley is a stablecoin protocol running on bitcoin L1, the stablecoin is called `bitusd`. Users deposit `BTC` on L1 and a 
corresponding amount of `bitusd` will be minted to the user.

The token standard that powers `bitusd`, which was called `bitrc-20` in bitSmiley's whitepaper, needs to:
1. Perform access control on mint/burn
2. Increasing the `max` amount of token issued according to the amount of `BTC` deposited

The above two requirements are critical because:
1. The mint and burn of `bitusd` is controlled by an over-collateralized algorithm. The exact amount and who can perform the above two action are determined by the amount of `BTC` deposited and the price of `BTC`. If there is no access control, the protocol simply won't work.
2. Similar to the above point, with more `BTC` deposited, the `max` supply of `bitusd` will increase accordingly. It's difficult to predetermine what `max` should be when the token is deployed.

There are other use cases aside from bitSmiley, to just name a few:
1. Unlock the minting of a token after certain block height
2. Allows only a certain whitelisted address to mint
3. Allows minting of a specific value of token for a specific address
4. ...

## Approach
This doc proposes an extension to the existing BRC-20 token standard with a lightweight scripting language that:
1. optimized for most common token operations 
2. potentially extendable to more generic operations
3. upgradable

The script is deployed together when the token is created. The scripts are optional. If the deployer does not specify any 
scripts, then it's just a BRC-20 token without current extension. If scripts are provided, it will be executed before the 
corresponding token operation is performed. Only when the execution of those scripts is successful, one can continue with the token operation.

### Token State Model
This proposal models (only the relevant fields) existing BRC-20 token as follows:
```
{
  "balance": "0",             # the current total balance in sats
  "max": "21000000000000000", # the max total circulation of the token in sats
  "lim": "10000000",          # the limit per mint operation
  "deployer": "address",      # the deployer
  "holders": { ... }          # the mapping of each address's balances    
}
```
With the scripting capability, the following fields are added:
```
{
  "balance": "0",             # the current total balance in sats
  "deployer": "address",      # the deployer
  
  # changes
  "max": "21000000000000000", # the max total circulation of the token in sat, optional
  "lim": "10000000",          # the limit per mint operation, optional
  
  # new fields  
  "methods": {
    "mint": "0xABC...",         # the mint script        
    "burn": "0xABC...",         # the burn script        
    "transfer": "0xABC...",     # the transfer script
  }

  "holders": { ... }          # the mapping of each address's balances    
}
```
When an address performs an token operation, such as a mint, the indexer will load the token state into memory, then checks 
if `methods.mint` is present. If no, proceed with minting, if yes, then executes the script. If the execution fails, stops the 
token execution, else proceed with minting.

### Scripting
This proposal proposes a series of OP_CODEs as an initial implementation. The set of OP_CODEs can be extended to more generic 
operations in the future. The initial set of OP_CODEs are:
```
OP_SET <STATE_FIELD> <VALUE>
OP_HEIGHT <HEIGHT>
OP_WHITELIST <ADDRESS> <ADDRESS> ...
OP_MERKLE <ROOT>
```
The details of the above OP_CODES are still debatable and can be discussed further.

### Deploy
The payload to deploy a new token is shown below:
```
{
  "p": "brc-20",         # protocol name
  "op": " deploy",       # operation
  "tick": "bitusd",      # token name
  "init": "0xABC..",     # the initial script to run after execution, nullalble,
  "v": "1",              # the version of the brc-20 script
  "methods": {
    "mint": "0xABC...",       # the script to run for mint, nullable,
    "burn": "0xABC...",       # the script to run for burn, nullable
    "transfer": "0xABC...",   # the script to run for transfer, nullable
  }
}
```
The `init` script will be executed in vm before `mint`/`burn`/`transfer` scripts are stored. The `init` script
allows one to run bootstrapping on the initial state of the token. If `init` is null, then no script will be executed.
One thing to note that this proposal does not impose a `max` nor `lim` on the token. It's all controlled in the scripts
by the deployer.

### Mint/Burn/Transfer
Mint and transfer follows that of `brc-20`, burn is basically the reverse of `mint`. The inscription payload is:
```
{
  "p": "brc-20",               # protocol name
  "op": " mint/burn/transfer", # operation
  "tick": "bitusd",            # token name
  "amt": "100",                # the amount,
  "payload": "0xABC..."        # extra payload attached to the token operation
}
```
`payload` is an extra data optional for the token operation scripts. For example, if the `mint` script
is `OP_MERKLE`, then the payload would be the merkle path. The script for `mint`/`burn`/`tranfer` will be executed before
the balances of each address is updated. If the script does not produce any error, the token operation will be granted.

### Upgrade
As for version 1, only the deployer can perform `upgrade`. The payload is:
```
{
  "p": "brc-20",         # protocol name
  "op": "upgrade",       # operation
  "tick": "bitusd",      # token name
  "exec": "0xABC.."      # the script to run for upgrade
}
```
The script in `exec` will be executed after the `deployer` check is passed. The `exec` will update the state of the token,
such as updating the `max` amount of the token or setting the `mint` scripts. One sample script that updates the `mint` and
`burn` scripts is:
```
OP_SET
    STATE.METHODS.MINT SCRIPT_LENGTH 0xABC...
    STATE.METHODS.BURN SCRIPT_LENGTH 0xABC...
    STATE.MAX ...
```
It is also possible to impose a `methods.upgrade` field, that executes the script before upgrade is performed.

## Extensions
The above mechanism can be extended to more complex but also generic operations. One can for example replace the method naming 
scheme to something more generic, similar to evm's method selector. 

Also the OP_CODE can extend to EVM or BTC scripting languages such as `OP_EVM <EVM SCRIPTS>` or `OP_BTC <BTC SCRIPTS>`. However, 
doing the above might be more bulky to the indexer and the txn fee associated with the inscription will be higher.