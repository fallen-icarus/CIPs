---
CIP: ?
Title: Beacon Tokens & Distributed Dapps
Category: Tools
Status: Informational
Authors:
    - fallen-icarus <modern.daidalos@gmail.com>
    - zhekson1 <zhekson@nomadpool.io>
Implementors: []
Discussions:
Created: 2023-02-21
License: CC-BY-4.0
---

## Abstract
Creating

Decentralized applications (dApps) have long been 

In the absence of atomic delegation, in order for layer 1 (L1) dApp users to maintain full delegation control of their assets, each user must have his/her own address while using the Dapp. This poses a challenge: if all users have their own addresses while using the Dapp, how can users find (and interact with) each other, without relying on a central order batcher? This informational CIP proposes an NFT standard, called ***Beacon Tokens***, to solve this broadcasting issue. Using beacon tokens, it is possible to create distributed L1 dApps (eg, DEXs, p2p lending, etc) where users not only maintain full delegation control of their assets, but can also maintain full custody of their assets at all times. Beacon tokens can be generalized to other use cases, such as creating an on-chain personal address book tied to - and fully recoverable by - a user's payment pub-key, or trustlessly sharing reference scripts with other blockchain users.

## Motivation: why is this CIP necessary?
To date, there are very few L1 dApps in which users maintain **complete** control of their assets, including staking rights. Those that do support individual staking tend not share how they've overcome the aggregation issue associated with giving each user his/her own dApp address. Other L1 dApps tend to pool user assets together and are therefore forced to delegate the assets together. In Proof-of-Stake (PoS) blockchains, the latter is a significant security concern; the more popular an L1 dApp becomes, the more centralized the underlying stake becomes. Some dApps try to address this concern by:

1. Fractionalizing the asset pools -  instead of all assets being at one address and delegated to only one stake pool, the assets are split among several addresses and delegated to several separate stake pools.
2. Issuing governance tokens - each user gets a vote on where the pooled assets will be delegated.

This general pattern is utilized by all kinds of L1 dApps: DAOs, DEXs, p2p lending, etc. Unfortunately, the above mitigations do not fully solve the issue; the blockchain is still significantly more centralized than if full delegation control was used.

## Specification

### Beacon Tokens

#### What are Beacon Tokens?
All native tokens on Cardano can, at any time, be easily queried for their current address and any transactions they were ever a part of. Here are some examples from off-chain APIs:

| Task | Koios API | Blockfrost API |
|--|--|--|
| Addresses with asset | [API](https://API.koios.rest/#get-/asset_address_list) | [API](https://docs.blockfrost.io/#tag/Cardano-Assets/paths/~1assets~1%7Basset%7D~1addresses/get) |
| Txs including asset | [API](https://API.koios.rest/#get-/asset_txs) | [API](https://docs.blockfrost.io/#tag/Cardano-Assets/paths/~1assets~1%7Basset%7D~1transactions/get) |

Knowing *exactly* which addresses/transactions have the token, the same off-chain APIs can be used to take a closer look at those addresses/transactions:

| Task | Koios API | Blockfrost API |
|--|--|--|
| UTxO info of address | [API](https://API.koios.rest/#post-/address_info) | [API](https://docs.blockfrost.io/#tag/Cardano-Addresses/paths/~1addresses~1%7Baddress%7D~1utxos/get)|
| Tx metadata | [API](https://API.koios.rest/#post-/tx_metadata) | [API](https://docs.blockfrost.io/#tag/Cardano-Transactions/paths/~1txs~1%7Bhash%7D~1metadata/get)|

Here is a short list of easily queryable info:

1. Any reference scripts stored at the address and their associated UTxO TxIx.
2. Any datums stored at the address.
3. The stake address associated to that payment address.
4. The metadata history of each transaction that had the token.
5. The datum history of each UTxO that held the token.

These queries are possible for *all* Cardano native tokens. However, while all native tokens *can* "broadcast" this information, the broadcasting itself is usually not the intended purpose of the native token (eg, governance tokens). In this CIP, **a *Beacon Token* is defined as a native token whose ONLY purpose is to "broadcast" information. They can be used to group together all kinds of otherwise unrelated on-chain data into efficiently queryable sets.** For example, to group certain related transactions together, the beacon token must simply have been a part of those transactions. To group certain addresses together, the beacon token must be stored at those addresses. To group certain UTxOs together, the beacon token must stored in those UTxOs, e.t.c.

#### Using Beacon Tokens
Every native token has two configurable fields: the policy ID and the token name. The policy ID should be application-specific while the token name should be data-specific. To use a beacon token, simply create a native token that is unique to your application and, if necessary, to control how UTxOs with beacons can be spent.

Here are a few basic reference implementations:

- [Cardano-Address-Book](https://github.com/fallen-icarus/cardano-address-book) - a personal payment address book on Cardano that is tied to a user's payment pub-key. It uses the Tx metadata API to broadcast all metadata attached to the transactions containing the beacon. The address book itself is the aggregation of all of the metadata. This application can be generalized to store *any* information in a way that is unique to, and protected by, the user's payment pub-key. The beacon token name is the user's payment pub-key hash. Minting the beacon requires the signature of the payment pub-key hash used for the token name. This guarantees that only Alice can mint Alice's personal beacon.

- [Cardano-Reference-Scripts](https://github.com/fallen-icarus/cardano-reference-scripts) - trustless p2p sharing of Cardano reference scripts. It uses the UTxO info API to broadcast all of the UTxOs that contain each beacon. The beacon token name is the script hash of the reference script being shared. Minting the beacon requires that the beacon is stored with the reference script whose hash matches the beacon's token name. This example also uses a helper plutus spending script to guarantee the burning of the beacon when a reference script UTxO is consumed. 

### Distributed dApps
As already alluded to, a *distributed dApp* is one where all users have their own personal address while using the dApp. This stands in contrast to *concentrated dApps*, where all users share one or a few addresses while using the dApp.

Users of distributed dApps can use beacon tokens to aggregate all of the necessary information they need to interact with peers who are using the same dApp, all without relying on a centralized batcher/discoverer. Such distributed dApps would follow a similar design pattern:

1. All user addresses use the same spending script for a given use case.
2. All user addresses *must* have a staking credential.
3. The spending script delegates the authorization of owner related actions to the address' staking credential - i.e., the staking key must sign (or the staking script must be successfully executed) in the same transaction.
4. The spending script's hash is hard-coded into the beacon minting policy and the minting policy enforces that the beacons can only be minted to an address of that spending script.
5. The datums for the spending script must have the policy ID of the beacons so that the script can enforce proper usage of the beacons (such as burning when necessary).

#### Advantages
Since each user has his/her own address for the dApp, the following are now possible:

- **Full delegation control**
- Users maintain full custody of their assets while using the dApp, since owner related actions must be approved by the staking credential.
- The address itself can act as the User ID. There is no need to place a User ID in a datum and guard its usage.

Furthermore, the dApp itself gains some nice features:

1. **Organic Concurrency** - since there must be at *least* as many UTxOs as there are users, the distributed dApp is naturally concurrent and gets *more* concurrent as the number of users increases.
2. **Democratic Upgradability** - users can choose whether to move their assets to an address using a newer version of the Dapp's spending script.
3. **Frontend Agnosticism** - the dApp is easily integratabtle into existing wallet frontends. Coordination between wallet providers is not required.
4. Since the address itself can act as the User ID, in some cases, the dApp's logic can be dramatically simplified.

#### Cardano-Swaps: A L1 DEX with full delegation control
Using these design principles, it was possible to create [cardano-swaps](https://github.com/fallen-icarus/cardano-swaps): a PoC reference implementation of a distributed DEX with full delegation control. It is fully operational, open-source, and can be tested on either the PreProd Testnet or Mainnet. In addition to full delegation control, it has the following features:

1. Composable atomic swaps
2. Users maintain full custody of their assets at all times.
3. Naturally concurrent and gets more concurrent the more users there are. No batchers required.
4. Liquidity organically spreads to all trading pairs instead of being siloed into specific trading pairs.
5. No discrete/central liquidity providers - liquidity is instead an emergent property of high volumes of swaps occurring at similar price points. Arbitragers may provide liquidity in a permissionless fashion.
6. Capital efficiency due to users' precise swap specifications
7. With enough liquidity, the DEX can eventually serve as a price oracle.
8. No governance/utility/LP tokens necessary (ADA required to pay for fees)
9. Upgrades are accepted democratically.
10. Easy to integrate into any frontend.

## Rationale: how does this CIP achieve its goals?
By being able to easily broadcast the necessary information for a Dapp, user assets can be segregated into separate addresses and, therefore, enable full delegation control while using the Dapp. Thanks to broadcasting, the segregated addresses can act as if all the assets were pooled together (checkout the `Cardano-Swaps` README to see how liquidity is achieved; it arises naturally).

The fact that beacon tokens are generalizable for aggregating any information on the blockchain is a bonus.

### Does the reliance on off-chain APIs create *more* centralization?
The short answer is no. It is important to look at what the limiting centralization factors are for any design. 

#### Centralization Bottlenecks for concentrated dApps:

1. Full delegation control is not possible without atomic delegation. This is baked into the on-chain design of the Dapp. 

2. Batchers are needed since there aren't enough UTxOs for each user. For most Dapps, it is not easy to become a batcher due to needing to be selected by some entity. Furthermore, these batchers are effectively middlemen that can take advantage of their unique positions. While new innovations in batcher protocols are improving this situation ([spectrum-finance](https://docs.spectrum.fi/docs/protocol-overview/bots) batchers can be anyone), this is a work in progress and is currently an off-chain centralization bottleneck for concentrated Dapps.

#### Centralization Bottlenecks for distributed L1 Dapps:

1. Reliance on off-chain API services. This is the only centralization bottleneck for distributed L1 Dapps; similarly to batchers, new innovations are improving this. For example, Koios is more decentralized than Blockfrost.

#### We can therefore extract the following conclusions:

- Concentrated L1 Dapps have centralization bottlenecks both in their on-chain design and their off-chain design.

- Distributed L1 Dapps only have a centralization bottleneck in their off-chain design.

This means that distributed L1 Dapps are better positioned to naturally grow more decentralized as the off-chain services become more decentralized.

## Path to Active
It is already possible to use beacon tokens and create distributed L1 Dapps. No updates to Cardano are necessary.

## Copyright
[CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode)