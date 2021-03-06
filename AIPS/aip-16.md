```
AIP: 16
title: Dynamic Fees
author: Kristjan KOSIC <chris@ark.io>, François-Xavier THOORENS <fx.thoorens@ark.io>, Alex BARNSLEY <alex@ark.io>
type: Standards Track
category: Core
status: Active
created: 2018-05-01
updated: 2018-08-21
```

History
========
- 2018-05-01 inital content (@kristjank)
- 2018-08-21 added more detailed explanation and related information to new settings available in node configuration (@kristjank)

Abstract
========
The purpose of dynamic fees is to enable a healthy market dynamics with options for delegates and users (transaction senders) to choose from. Acceptance fee threshold is decided by the delegate(s) - if fees are below threshold - they won't accept the payload into the transaction pool. The transaction fee is defined by the transaction sender.

Motivation
==========
The transaction size defines market dynamics - you pay more for storage, computation, additional information. To optimize fees, a delegate puts minimum limits in the delegate config.  To reduce spamming forger rules need to be setup and implemented in the configs (starting with fixed ones).

Dynamic Fee calculation represents the rule for including the transactions in the delegate pool and eventually forging them, or just broadcasting them if they don't fit to the peer conditions.

Actors
===============
## Users/Customers:
A user sets his exact fee he is willing to pay when sending a transaction (setting a custom fee in the transaction payload). An insanely low fee would result in transaction never being forged (fee will not make it into delegate pools). This opens new doors to fee processing in the future, for example processing of transaction with higher fees first.

## Delegates:
Define their own C (constant) known as the `feeMultiplier`. See [Formula calculation](##formula-calculation) for fee calculation according to the formula on API11, that is related to:
- Type of the transaction
- Size of the serialised transaction

Specifications
===============
## Node:
A client API can retrieve history values of fees, so market monitoring can be done based on the node config endpoint and new services can be provided to users to monitor the market behaviour. By calling [node configuration endpoint](https://docs.ark.io/api/public/v2/node.html#retrieve-the-configuration) a `feeStatistics` parameter is returned, where minimum, maximum and average fee for the last 30 days is returned, calculated by transaction type. 
```json
    "feeStatistics": [
      {
        "type": 0,
        "fees": {
          "minFee": 268421,
          "maxFee": 597781,
          "avgFee": 404591
        }
      }
    ],       
```

Returned values will be used in wallets and other GUI client application to help users choose optmial fees, when sending transations. 

## Formula calculation:
Dynamic fee calculation is related to the:
- Type of the transaction (offset value defined by network)
- Size of the serialised transaction

The calculation formula: `Fee = (T+S) * C`
- T: offset value depending on transaction type, defined by the network. T is here to account for extra processing power to process special transaction whose transfer value is null, and thus reducing economic interest to spam the network.

- S: size of the serialised transaction. For instance, for transfer we could have offset T = 0, C = 1000 Arktoshi/byte. For a classic transfer transaction with empty VendorField size is 153 bytes, the fee is:

`Fee:= (0 + 153) * 1000 = 153000 ARKTOSHI === 0.00153000 ARK`.

- C: fee multiplier constant (Arktoshi/byte) defined by the delegates for including the transaction in his forged block/transaction pool

For more information about fomula calculation paramatere see [Formula calculation network parameters](###formula-calculation-network-parameters).


Specifications
==============
## Client SDK(s)
All client libraries will enable users to set a custom fee for the transaction(s), before the transaction is signed and sent. Some optional limits or security checks are recommended on client GUI level, to inform the users if the fee is above average or above current static fees defined in node configuration.

## Core/Node level configuration
### Delegate settings
A delegate can define his formula parameters for `C-feeMultiplier` and limit incoming transactions with `minAcceptableFee` value. All settings are in ARKTOSHI per byte. Delegate settings can be found in [delegates.json](https://github.com/ArkEcosystem/core/blob/develop/packages/core/lib/config/devnet/delegates.json#L2-L4)

Example:
```json
  "dynamicFees": {
    "feeMultiplier": 1000,
    "minAcceptableFee": 30000
  }
```

### Formula calculation network parameters
- `feeMultiplier` is C in the formula above. 
- `minAcceptableFee` is the limit for the inclusion of transaction by the delegate. If the incomming transaction has lower fee than delegates `minAcceptableFee` then the transaction is not included in the delegates pool, but it is only broadcasted to other nodes, where other delegates can pick it up, according to the defined rules.

`T - acts as offset` and is defined in the [network.json](https://github.com/ArkEcosystem/core/blob/c7a3bc75ffed5e5b9453d0de38937540fe48bce5/packages/crypto/lib/networks/ark/devnet.json#L39-L48) for each of the supported transaction types. Offsets are defined as following:

```json
   "dynamicOffsets": {
      "transfer": 100,
      "secondSignature": 250,
      "delegateRegistration": 500,
      "vote": 100,
      "multiSignature": 500,
      "ipfs": 250,
      "timelockTransfer": 500,
      "multiPayment": 500,
      "delegateResignation": 500
   }
```
### Enabling or disabling of dynamic fees
By setting the value `dynamic` to `true` and defining block height from which the settings should go into effect - we can enable or disable the dynamic fees on the node level. Settings are in [network.json](https://github.com/ArkEcosystem/core/blob/c7a3bc75ffed5e5b9453d0de38937540fe48bce5/packages/crypto/lib/networks/ark/testnet.json#L52-L55)

For example the settings below will enable dynamic fee acceptance from block height 10 onward.
```json
    "height": 10,
    "fees":{
      "dynamic" : true
    }
```

The example below will disable dynamic fee processing from block 20 onward. All fees will be calculated according to static fees defined in [network.json](https://github.com/ArkEcosystem/core/blob/c7a3bc75ffed5e5b9453d0de38937540fe48bce5/packages/crypto/lib/networks/ark/testnet.json#L27-L38)
```json
    "height": 20,
    "fees":{
      "dynamic" : false
    }
```
