# The gateway RPC

The gateway is the client facing surface of a Quantova node. Every method is an HTTP POST to `/v1/<method>` with a flat JSON body, and the reply is a JSON object. The method shapes are taken from the gateway source in Quantova-Chain under `crates/qtv-gateway/src`, and the field values in the examples are illustrative so only the shapes are normative.

## Transport

A request is a POST to a method path under the version prefix, for example `POST /v1/node_info`. The body is a flat JSON object, and an empty body is read as an empty object so a method with no fields needs no body. The reply carries `Content-Type application/json`. The gateway answers an OPTIONS preflight with 204 and sets permissive CORS headers, and it rejects any verb other than POST with 405. The request head is capped at 16 KiB and the body at 2 MiB, a connection over the cap is refused, and a slow request times out at fifteen seconds. A local devnet node serves the gateway on `127.0.0.1:8645`.

An error reply is the object `{"error":"<code>","message":"<text>"}`. The codes a client sees are `bad_request` and `bad_address` at 400, `not_found` at 404, `unknown_method` at 404, `method_not_allowed` at 405, `too_large` at 413, `head_too_large` at 431, and `busy` or `unavailable` at 503.

## Methods

The methods are `node_info`, `head`, `validators`, `chain_params`, `staking_state`, `get_account`, `get_transaction`, `submit_transaction`, `get_block`, `pending`, `supply`, `get_container`, `get_storage`, `get_events`, `finalized_head`, `burn_block`, and `burn_heights_after`.

Every method's request and response, field by field with an example, is in [rpc.md](rpc.md).
