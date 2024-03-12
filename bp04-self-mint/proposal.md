
## Self Issuance Mechanism

The default BRC-20 standard uses a public issuance method for deployment and distribution of assets. After the asset is deployed, anyone can begin to mint their own portion. We need a new way that allows only the deployer to participate in minting after the asset deployment.

## Changes to Deploy Inscription

A new field called (self_mint) can be added to the original BRC-20 protocol's deploy operation. This setting ensures that only the deployer can issue the current asset. The meanings of the other fields remain the same as in the original BRC-20 rules, such as setting (max) for the maximum issuance and (lim) for the limit per issuance, etc.
Under this mode, when issuing the mint inscription, the deploy inscription must be used as the parent of the mint inscription; otherwise, the mint is invalid.

* BRC-20 deploy inscription content:
```
{
  "p": "brc-20",
  "op": "deploy",
  "tick": "ordi",
  "max": "21000000",
  "lim": "1000"
}
```
* Inscription content for assets that can only be issued by the deployer:
```
{
  "p": "brc-20",
  "op": "deploy",
  "self_mint": "true",
  "tick": "ordi",
  "max": "21000000",
  "lim": "1000"
}
```
* When (max=0), BRC-20 rules do not allow this situation, but it is reasonable to define (max=0) as allowing an unlimited maximum issuance for self_mint. (BRC-20 itself requires the maximum asset limit to be max_uint64, which we continue to use here.)

For BRC20, the meaning of max is the total upper limit of all mints. BRC20 assets cannot truly disappear. No matter if the assets are transferred to any non-spendable address, it will not affect the max restriction rule for mints.
```
{
  "p": "brc-20",
  "op": "deploy",
  "self_mint": "true",
  "tick": "ordi",
  "max": "0",
  "lim": "1000"
}
```

## Adopting 5-bytes Tickers

Thanks to @SeeSharp's suggestion, we propose self-issuing an asset with a 5-bytes ticker to isolate existing assets. In this way indexers not prepared yet will simply ignore these assets.

## FAQ

### Alternative Option B:

Assets with self_mint could also be created by adding a new operation (op) instead of a new field. The inscription format would be:
```
{
  "p": "brc-20",
  "op": "deploy-self-mint",
  "tick": "ordi",
  "max": "21000000",
  "lim": "1000"
}
```



#### Advantages:

Introducing a new operation will isolate it from the current indexing, and indexers that don't upgrade will not recognize this deploy inscription, which to some extent reduces confusion.

#### Disadvantages:

Adding a new operation involves a significant change.
