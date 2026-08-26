# The gateway RPC

The gateway is the client facing surface of a Quantova node. Every method is an HTTP POST to `/v1/<method>` with a flat JSON body, and the reply is a JSON object. The methods and their shapes below are taken from the gateway source in Quantova-Chain under `crates/qtv-gateway/src`. The field values in the examples are illustrative and only the shapes are normative.

## Transport

A request is a POST to a method path under the version prefix, for example `POST /v1/node_info`. The body is a flat JSON object, and an empty body is read as an empty object so a method with no fields needs no body. The reply carries `Content-Type application/json`. The gateway answers an OPTIONS preflight with 204 and sets permissive CORS headers, and it rejects any verb other than POST with 405. The request head is capped at 16 KiB and the body at 1 MiB, a connection over the cap is refused, each read times out at fifteen seconds, and the whole request head and body must arrive within twenty seconds. A local devnet node serves the gateway on `127.0.0.1:8645`.

An error reply is the object `{"error":"<code>","message":"<text>"}`. The codes a client sees are `bad_request`, `bad_address`, and `storage_too_large` at 400, `not_found` at 404, `unknown_method` at 404, `method_not_allowed` at 405, `too_large` at 413, `head_too_large` at 431, `forbidden` at 403, `timeout` at 408, `too_many`, `rate_limited`, or `banned` at 429, and `busy` or `unavailable` at 503. A body that is not valid UTF-8, or shorter than its declared length, is `bad_request` at 400.

The methods are `node_info`, `head`, `validators`, `chain_params`, `staking_state`, `get_account`, `get_transaction`, `submit_transaction`, `get_block`, `pending`, `supply`, `get_container`, `get_storage`, `get_storage_at`, `get_events`, `get_side_events`, `get_bridged_balance`, `get_bridged_supply`, `get_asset_balance`, `get_asset_supply`, `governance_referenda`, `genesis_accounts`, `finalized_head`, `burn_block`, and `burn_heights_after`.

## node_info

Returns the chain identity, the head height, the asset and its denomination, the current fee, and the node version.

**Path** `POST /v1/node_info`

The request has no fields.

| response field | type | meaning |
| --- | --- | --- |
| chain_id | string | the chain identifier |
| genesis_hash | string | the genesis hash in hex |
| head_height | integer | the height of the finalized head |
| asset | string | the native asset symbol, TQTOV on the testnet |
| denomination | string | the base unit name, Quon |
| fee | object | the current fee, see below |
| version | string | the node version |

The fee object carries `transfer_micro_usd`, `rate_micro_usd_per_qtov`, `quon_per_qtov`, and `transfer_quon`, each a decimal string. The fee is capped in the native asset and targeted in United States dollars, held to a tenth of a cent by the governance set rate.

```
POST /v1/node_info
{}
```
```
{"chain_id":"Q-test-net-1","genesis_hash":"9f2c...","head_height":10428,"asset":"TQTOV","denomination":"Quon","fee":{"transfer_micro_usd":"500","rate_micro_usd_per_qtov":"1000000","quon_per_qtov":"1000000","transfer_quon":"500"},"version":"0.1.0"}
```

## head

Returns the current finalized head.

**Path** `POST /v1/head`

The request has no fields.

| response field | type | meaning |
| --- | --- | --- |
| height | integer | the height of the head |
| block | string or null | the block id as a QBK identifier, null before the first finalized block |
| q_root | string | the Q state root of the head |

```
POST /v1/head
{}
```
```
{"height":10428,"block":"QBK1QW8P...","q_root":"QST1M4RD..."}
```

## validators

Returns the active validator set with each validator's staked weight.

**Path** `POST /v1/validators`

The request has no fields.

| response field | type | meaning |
| --- | --- | --- |
| count | integer | the number of validators |
| validators | array | one entry per validator |

Each validator entry carries `address`, a Q1 string, and `stake`, an integer weight in Quon.

```
POST /v1/validators
{}
```
```
{"count":1,"validators":[{"address":"Q1QW8P...","stake":10000000000}]}
```

## chain_params

Returns the staking and governance parameters the chain runs under.

**Path** `POST /v1/chain_params`

The request has no fields.

The response has a `staking` object and a `governance` object. The staking object carries `native_unit`, `min_stake`, `staking_pool`, `session_emission`, `session_days`, `high_session_tx`, `mainnet_blackout_days`, `bond_lock_days`, `unbonding_days`, and `reward_vest_days`, each an integer. The governance object carries `conviction_max_x10`, an integer, and `tracks`, an array where each track carries `code`, `deposit`, `threshold_bps`, and `period_seconds`. There are seven governance tracks and the example below shows the first.

```
POST /v1/chain_params
{}
```
```
{"staking":{"native_unit":1000000,"min_stake":2000000000,"staking_pool":685714000000,"session_emission":0,"session_days":182,"high_session_tx":50000000000,"mainnet_blackout_days":365,"bond_lock_days":90,"unbonding_days":21,"reward_vest_days":365},"governance":{"conviction_max_x10":25,"tracks":[{"code":1,"deposit":600000,"threshold_bps":4000,"period_seconds":1209600}]}}
```

## staking_state

Returns the live staking pools and the price the reward cap is measured against.

**Path** `POST /v1/staking_state`

The request has no fields.

| response field | type | meaning |
| --- | --- | --- |
| reward_pool | integer | the remaining reward pool in Quon |
| treasury | integer | the treasury balance in Quon |
| price_micro_usd_per_qtov | string | the price in micro dollars per QTOV |
| mainnet_started | boolean | whether the mainnet start point has passed |
| governance_locked | string | the total stake locked in governance |

```
POST /v1/staking_state
{}
```
```
{"reward_pool":685714000000,"treasury":0,"price_micro_usd_per_qtov":"1000000","mainnet_started":false,"governance_locked":"0"}
```

## get_account

Returns the account record at a Q1 address.

**Path** `POST /v1/get_account`

| request field | type | meaning |
| --- | --- | --- |
| address | string | the Q1 address to read |

| response field | type | meaning |
| --- | --- | --- |
| address | string | the address that was read |
| nonce | integer | the next expected nonce |
| balance | string | the balance in Quon as a decimal string |
| scheme | integer | the signature scheme identifier of the account key |
| has_key | boolean | whether the account has a registered public key |

An address that is not a Q1 Bech32m address returns `bad_address` at 400.

```
POST /v1/get_account
{"address":"Q1QW8P..."}
```
```
{"address":"Q1QW8P...","nonce":3,"balance":"8000000000","scheme":1,"has_key":true}
```

## get_transaction

Returns the status and, when known, the fields of a transaction by its id.

**Path** `POST /v1/get_transaction`

| request field | type | meaning |
| --- | --- | --- |
| tx_id | string | the QTX transaction id |

The response always carries `tx_id` and `status`, where status is `finalised`, `pending`, or `unknown`. A finalised transaction also carries `height` and `block`. A finalised or pending transaction also carries the transaction fields `from`, `to`, `kind`, `value`, `fee`, `nonce`, `meter_limit`, `scheme`, `signature`, and `raw`, and a deploy also carries `contract`, the address the deploy creates. The `value`, `fee`, `from`, `to`, `kind`, `signature`, `raw`, and `contract` fields are strings, where `signature` and `raw` are hex, and `nonce`, `meter_limit`, and `scheme` are integers. The `kind` names the call, one of `transfer`, `deploy`, `call`, `bridge_mint`, `bridge_exit`, `bridge_settle`, `bridge_guardian`, `register`, `key_register`, `evidence`, or `governance`.

```
POST /v1/get_transaction
{"tx_id":"QTX1A9F0..."}
```
```
{"tx_id":"QTX1A9F0...","status":"finalised","height":10420,"block":"QBK1QW8P...","from":"Q1QW8P...","to":"Q1M4RD...","kind":"transfer","value":"1000","fee":"500","nonce":2,"meter_limit":100000,"scheme":1,"signature":"a1b2...","raw":"00ff..."}
```

## submit_transaction

Submits a signed transaction to the mempool. The transaction is the canonical wrapper bytes in hex, as produced by QCore.

**Path** `POST /v1/submit_transaction`

| request field | type | meaning |
| --- | --- | --- |
| tx | string | the canonical signed transaction in hex |

On acceptance the reply is `verdict` `accepted`, `state` either `fresh` or `known`, and `tx_id`. On rejection the reply is `verdict` `rejected` and `reason`. The reason is one of `malformed`, `unknown_sender`, `unsupported_scheme`, `bad_signature`, `bad_nonce`, `bad_call`, `self_transfer`, `meter_limit_too_low`, `fee_too_low`, `insufficient_funds`, `wrong_chain`, `pool_full`, `sender_queue_full`, or `rate_limited`. A `bad_nonce` rejection also carries `expected` and `got`. A `wrong_chain` rejection means the wrapper carried a chain id that is not this chain, and a client must sign for the chain id in `node_info` before it resubmits.

```
POST /v1/submit_transaction
{"tx":"00ff1234..."}
```
```
{"verdict":"accepted","state":"fresh","tx_id":"QTX1A9F0..."}
```

## get_block

Returns a finalized block by height or by block id. Supply exactly one of the two.

**Path** `POST /v1/get_block`

| request field | type | meaning |
| --- | --- | --- |
| height | integer | the block height, optional |
| block | string | the QBK block id, optional |

| response field | type | meaning |
| --- | --- | --- |
| height | integer | the block height |
| block | string | the block id |
| parent | string | the parent block id as a QBK identifier |
| q_root | string | the Q state root as a QST identifier |
| transaction_root | string | the transaction root in hex |
| event_root | string | the event root in hex |
| proposer | string | the proposer address |
| time | integer | the block time |
| tx_count | integer | the number of transactions |
| extra_data | string | the header extra data in hex |
| tx_ids | array | the transaction ids in the block |

A missing height or block id returns `bad_request`, and a height or id with no finalized block returns `not_found` at 404.

```
POST /v1/get_block
{"height":10420}
```
```
{"height":10420,"block":"QBK1QW8P...","parent":"QBK1H3K2...","q_root":"QST1M4RD...","transaction_root":"7a1b...","event_root":"9c2d...","proposer":"Q1QW8P...","time":1700004200,"tx_count":1,"extra_data":"","tx_ids":["QTX1A9F0..."]}
```

## pending

Returns the transactions currently in the mempool.

**Path** `POST /v1/pending`

The request has no fields.

The reply carries `count`, the total pending, `returned`, the number served, `truncated`, whether the node held back entries, and `transactions`, an array where each entry carries `tx_id` and the same transaction fields as `get_transaction`. The node serves at most one thousand entries and stops within a response byte budget, so a large mempool is truncated.

```
POST /v1/pending
{}
```
```
{"count":1,"returned":1,"truncated":false,"transactions":[{"tx_id":"QTX1A9F0...","from":"Q1QW8P...","to":"Q1M4RD...","kind":"transfer","value":"1000","fee":"500","nonce":2,"meter_limit":100000,"scheme":1,"signature":"a1b2...","raw":"00ff..."}]}
```

## supply

Returns the total native supply in Quon.

**Path** `POST /v1/supply`

The request has no fields.

| response field | type | meaning |
| --- | --- | --- |
| supply_quon | string | the total supply in Quon as a decimal string |

```
POST /v1/supply
{}
```
```
{"supply_quon":"4571429000000"}
```

## get_container

Returns the bytecode container held at a contract address.

**Path** `POST /v1/get_container`

| request field | type | meaning |
| --- | --- | --- |
| address | string | the Q1 contract address |

| response field | type | meaning |
| --- | --- | --- |
| address | string | the address that was read |
| container | string | the container bytecode in hex |
| size | integer | the container length in bytes |

An address that is not a Q1 address returns `bad_address`, and an address that holds no contract returns `not_found` at 404.

```
POST /v1/get_container
{"address":"Q1C0DE..."}
```
```
{"address":"Q1C0DE...","container":"00010203...","size":512}
```

## get_storage

Returns the contract storage slots at a contract address.

**Path** `POST /v1/get_storage`

| request field | type | meaning |
| --- | --- | --- |
| address | string | the Q1 contract address |

The reply carries `address`, `count`, the total slot count, `returned`, the number served, `truncated`, whether the node held back slots, and `slots`, an array where each entry carries `slot`, the slot key in hex, and `value`, the slot value as a decimal string. The node serves at most one thousand slots, so a large map is truncated. An address that is not a Q1 address returns `bad_address`, and a contract whose storage is too large to serve over the RPC returns `storage_too_large` at 400.

```
POST /v1/get_storage
{"address":"Q1C0DE..."}
```
```
{"address":"Q1C0DE...","count":1,"returned":1,"truncated":false,"slots":[{"slot":"0000...0001","value":"42"}]}
```

## get_storage_at

Returns the named contract storage slots at a contract address, so a client reads only the keys it asks for.

**Path** `POST /v1/get_storage_at`

| request field | type | meaning |
| --- | --- | --- |
| address | string | the Q1 contract address |
| keys | array | the slot keys to read, each a thirty two byte hex string |

At most sixty four keys are read per request, and a key that is not thirty two bytes of hex returns `bad_request`. The reply carries `address` and `slots`, an array where each entry carries `slot`, the slot key in hex, and `value`, the slot value as a decimal string. Only keys the contract holds appear in the reply. An address that is not a Q1 address returns `bad_address`, and a contract whose storage is too large to serve over the RPC returns `storage_too_large` at 400.

```
POST /v1/get_storage_at
{"address":"Q1C0DE...","keys":["0000...0001"]}
```
```
{"address":"Q1C0DE...","slots":[{"slot":"0000...0001","value":"42"}]}
```

## get_events

Returns the events recorded at a block height.

**Path** `POST /v1/get_events`

| request field | type | meaning |
| --- | --- | --- |
| height | integer | the block height |

The reply carries `height`, `count`, and `events`, an array where each entry carries `contract`, the contract address, `selector`, the event selector in hex, and `data`, the event payload in hex. A missing height returns `bad_request`.

```
POST /v1/get_events
{"height":10420}
```
```
{"height":10420,"count":1,"events":[{"contract":"Q1C0DE...","selector":"1a2b3c4d","data":"00ff..."}]}
```

## get_side_events

Returns the side events recorded at a block height. A side event is a state change the chain records outside the contract event log, such as a stake bond, a governance vote, a bridge move, or a treasury spend.

**Path** `POST /v1/get_side_events`

| request field | type | meaning |
| --- | --- | --- |
| height | integer | the block height |

The reply carries `height`, `count`, and `events`, an array where each entry carries `index`, `kind`, `actor`, `target`, `amount`, `ref`, and `aux`, and further fields that depend on the kind. The `index` is the position in the block, `kind` names the side event, `actor` and `target` are addresses or empty, `amount` is a decimal string, and `ref` and `aux` are integers. A missing height returns `bad_request`.

```
POST /v1/get_side_events
{"height":10420}
```
```
{"height":10420,"count":1,"events":[{"index":0,"kind":"bond","actor":"Q1QW8P...","target":"","amount":"2000000000","ref":0,"aux":0,"fee":"500"}]}
```

## get_bridged_balance

Returns a holder's balance of a bridged asset and that asset's bridged supply.

**Path** `POST /v1/get_bridged_balance`

| request field | type | meaning |
| --- | --- | --- |
| asset_id | string | the bridged asset id, sixteen bytes of hex |
| holder | string | the holder, a Q1 address or thirty two bytes of hex |

The reply carries `asset_id` in hex, `holder` in hex, `holder_address` as a Q1 string, `balance` as a decimal string, and `supply` as a decimal string.

```
POST /v1/get_bridged_balance
{"asset_id":"00112233445566778899aabbccddeeff","holder":"Q1QW8P..."}
```
```
{"asset_id":"00112233445566778899aabbccddeeff","holder":"1111...","holder_address":"Q1QW8P...","balance":"0","supply":"0"}
```

## get_bridged_supply

Returns the bridged supply of an asset, its cap, and whether it is registered.

**Path** `POST /v1/get_bridged_supply`

| request field | type | meaning |
| --- | --- | --- |
| asset_id | string | the bridged asset id, sixteen bytes of hex |

The reply carries `asset_id` in hex, `supply` as a decimal string, `cap` as a decimal string, and `registered`, a boolean.

```
POST /v1/get_bridged_supply
{"asset_id":"00112233445566778899aabbccddeeff"}
```
```
{"asset_id":"00112233445566778899aabbccddeeff","supply":"0","cap":"0","registered":false}
```

## get_asset_balance

Returns a holder's balance of an issuer native asset and that asset's supply.

**Path** `POST /v1/get_asset_balance`

| request field | type | meaning |
| --- | --- | --- |
| issuer | string | the issuer, a Q1 address or thirty two bytes of hex |
| holder | string | the holder, a Q1 address or thirty two bytes of hex |

The reply carries `issuer` as a Q1 string, `holder` as a Q1 string, `balance` as a decimal string, and `supply` as a decimal string.

```
POST /v1/get_asset_balance
{"issuer":"Q1ISSUE...","holder":"Q1QW8P..."}
```
```
{"issuer":"Q1ISSUE...","holder":"Q1QW8P...","balance":"0","supply":"0"}
```

## get_asset_supply

Returns the supply of an issuer native asset.

**Path** `POST /v1/get_asset_supply`

| request field | type | meaning |
| --- | --- | --- |
| issuer | string | the issuer, a Q1 address or thirty two bytes of hex |

The reply carries `issuer` as a Q1 string and `supply` as a decimal string.

```
POST /v1/get_asset_supply
{"issuer":"Q1ISSUE..."}
```
```
{"issuer":"Q1ISSUE...","supply":"0"}
```

## governance_referenda

Returns the most recent governance referenda the chain holds.

**Path** `POST /v1/governance_referenda`

The request has no fields.

The reply carries `referenda`, an array where each entry carries `id`, `track`, `proposer`, `deposit`, `submitted_at`, `aye_stake`, `nay_stake`, `status`, and `killed`. The `id`, `track`, and `submitted_at` are integers, `proposer` is a Q1 address, `deposit`, `aye_stake`, and `nay_stake` are decimal strings, `status` is one of `deciding`, `approved`, or `rejected`, and `killed` is a boolean. The node serves at most one thousand entries, the most recent first.

```
POST /v1/governance_referenda
{}
```
```
{"referenda":[{"id":1,"track":1,"proposer":"Q1QW8P...","deposit":"600000","submitted_at":1700004200,"aye_stake":"0","nay_stake":"0","status":"deciding","killed":false}]}
```

## genesis_accounts

Returns the genesis account allocations and the genesis supply baseline.

**Path** `POST /v1/genesis_accounts`

The request has no fields.

The reply carries `count`, the total allocation count, `returned`, the number served, `truncated`, whether the node held back rows, `supply_quon`, the genesis supply as a decimal string, and `accounts`, an array where each entry carries `address`, a Q1 string, `balance`, a decimal string, and `scheme`, an integer. The node serves at most one thousand rows.

```
POST /v1/genesis_accounts
{}
```
```
{"count":2,"returned":2,"truncated":false,"supply_quon":"4571429000000","accounts":[{"address":"Q1QW8P...","balance":"5000","scheme":1}]}
```

## finalized_head

Returns the height of the highest finalized block the node holds. The bridge exit watcher reads this to learn how far the chain has settled before it asks for a burn.

**Path** `POST /v1/finalized_head`

Takes no request fields. The reply carries `head`, the finalized height as an integer.

```
POST /v1/finalized_head
{}
```
```
{"head":10420}
```

## burn_block

Returns the archived material for a finalized bridge-burn block, so the burn can be proven off chain. A block is archived only when it carries a bridge burn, so this answers for burn heights and returns `not_found` at 404 for any other height.

**Path** `POST /v1/burn_block`

| request field | type | meaning |
| --- | --- | --- |
| height | integer | the finalized block height |

The reply carries `height`, `header_bytes`, the block header in hex, `certificate`, the QORUS finality certificate in hex, and `events`, the ordered event leaves in hex that the header event root was computed over. A caller rebuilds the burn inclusion proof from `events` and verifies `certificate` against its own pinned committee, so the reply is untrusted data and nothing is taken on the node's word.

```
POST /v1/burn_block
{"height":10420}
```
```
{"height":10420,"header_bytes":"00a1...","certificate":"5c2f...","events":["7174762f...","..."]}
```

## burn_heights_after

Returns the finalized heights that carry a bridge burn and sit above a cursor, in order, so the exit watcher can step from burn to burn without walking every height.

**Path** `POST /v1/burn_heights_after`

| request field | type | meaning |
| --- | --- | --- |
| cursor | integer | return burn heights strictly greater than this |

The reply carries `cursor`, `count`, and `heights`, the ordered burn heights above the cursor. The node serves a bounded number of heights per call, so a client steps with the last height it saw as the next cursor.

```
POST /v1/burn_heights_after
{"cursor":10000}
```
```
{"cursor":10000,"count":2,"heights":[10420,10930]}
```
