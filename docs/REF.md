# `ripple-rest` API Reference

This is a reference for the [__Object Formats__](#object-formats) and [__Available API Routes__](#available-api-routes).

Note that the data formats used by this API are different in a number of important ways from those used in `rippled`, `ripple-lib`, and the accompanying documentation. See [below](#differences-from-standard-ripple-data-formats) for more information.

See the [__Guide__](GUIDE.md) for a walkthrough of how this API is intended to be used.


----------

## Differences from standard Ripple data formats

#### New data formats

This API uses different submission and retrieval formats for payments and other transactions than other Ripple technologies. These new formats are intended to standardize fields across different transaction types (Payments, Orders, etc). See the [Object Formats](#object-formats) for more information.

#### No XRP "drops"

Both `rippled` and `ripple-lib` use XRP "drops", or millionths (1/1000000) of an XRP, to denote amounts in XRP. This API uses whole XRP units in amount fields and in reporting network transaction fees paid.

#### XRP Amount as an object

Outside of this API, XRP Amounts are usually denoted by a string representing XRP drops. Not only does this API not use XRP drops, for consistency XRP represented here as an `Amount` object like all other currencies. See the [`Amount`](#1-amount) format for more information.

#### UNIX Epoch instead of Ripple Epoch

This API uses the more standard UNIX timestamp instead of the Ripple Epoch Offset to denote times. The UNIX timestamp is the number of milliseconds since January 1st, 1970 (00:00 UTC). In `rippled` timestamps are stored as the number of seconds since the Ripple Epoch, January 1st, 2000 (00:00 UTC).

#### Not compatible with `ripple-lib`

While this API uses [`ripple-lib`](https://github.com/ripple/ripple-lib/), the Javascript library for connecting to the Ripple Network, the formats specified here are not compatible with `ripple-lib`. These formats can only be used with this API.

----------

## Object Formats

1. [Amount](#1-amount)
2. [Payment](#2-payment)
3. [Notification](#3-notification)

----------

#### 1. Amount

```js
{
  "value": "1.0",
  "currency": "USD",
  "issuer": "r..."
}
```
Or for XRP:
```js
{
  "value": "1.0",
  "currency": "XRP",
  "issuer": ""
}
```

All currencies on the Ripple Network have issuers, except for XRP. In the case of XRP, the `"issuer"` field may be omitted or set to `""`. Otherwise, the `"issuer"` must be a valid Ripple address of the gateway that issues the currency.

Note that the `value` can either be specified as a string or a number. Internally this API uses a BigNumber library to retain higher precision if numbers are inputted as strings.

For more information about XRP see [the Ripple Wiki page on XRP](https://ripple.com/wiki/XRP). For more information about using currencies other than XRP on the Ripple Network see [the Ripple Wiki page for gateways](https://ripple.com/wiki/Ripple_for_Gateways).

----------

#### 2. Payment

The `Payment` object is a simplified version of the standard Ripple transaction format. 

This `Payment` format is intended to be straightforward to create and parse, from strongly or loosely typed programming languages. Once a transaction is processed and validated it also includes information about the final details of the payment.

The following fields are the minimum required to submit a `Payment`:
```js
{
  "src_address": "rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz",
  "dst_address": "rNw4ozCG514KEjPs5cDrqEcdsi31Jtfm5r",
  "dst_amount": {
    "value": "0.001",
    "currency": "XRP",
    "issuer": ""
  }
}
```
+ `dst_amount` is an [`Amount` object](#1-amount)

The full set of fields accepted on `Payment` submission is as follows:

```js
{
    /* User Specified */

    "src_address": "rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz",
    "src_tag": "",
    "src_amount": {
        "value": "0.001",
        "currency": "XRP",
        "issuer": ""
    },
    "src_slippage": "0",
    "dst_address": "rNw4ozCG514KEjPs5cDrqEcdsi31Jtfm5r",
    "dst_tag": "",
    "dst_amount": {
        "value": "0.001",
        "currency": "XRP",
        "issuer": ""
    },

    /* Advanced Options */

    "invoice_id": "",
    "paths": "[]",
    "flag_no_direct_ripple": false,
    "flag_partial_payment": false
}
```
+ `src_tag` is an optional unsigned 32 bit integer (0-4294967294, inclusive) that is generally used if the sender is a hosted wallet at a gateway. This should be the same as the `dst_tag` used to identify the hosted wallet when they are receiving a payment.
+ `dst_tag` is an optional unsigned 32 bit integer (0-4294967294, inclusive) that is generally used if the receiver is a hosted wallet at a gateway
+ `src_slippage` can be specified to give the `src_amount` a cushion and increase its chance of being processed successfully. This is helpful if the payment path changes slightly between the time when a payment options quote is given and when the payment is submitted. The `src_address` will never be charged more than `src_slippage` + the `value` specified in `src_amount`
+ `invoice_id` is an optional 256-bit hexadecimal hash field that can be used to link payments to an invoice or bill
+ `paths` is a ["stringified"](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify) version of the Ripple PathSet structure. Most users of this API will want to treat this field as opaque. See the [Ripple Wiki](https://ripple.com/wiki/Payment_paths) for more information about Ripple pathfinding. Advanced users can submit non-stringified objects if desired
+ `flag_no_direct_ripple` is a boolean that can be set to `true` if `paths` are specified and the sender would like the Ripple Network to disregard any direct paths from the `src_address` to the `dst_address`. This may be used to take advantage of an arbitrage opportunity or by gateways wishing to issue balances from a hot wallet to a user who has mistakenly set a trustline directly to the hot wallet. Most users will not need to use this option.
+ `flag_partial_payment` is a boolean that, if set to true, indicates that this payment should go through even if the whole amount cannot be delivered because of a lack of liquidity or funds in the `src_address` account. The vast majority of senders will never need to use this option.

When a payment is confirmed in the Ripple ledger, it will have additional fields added:
```js
{
    /* ... */

    /* Generated After Validation */

    "tx_direction": "outgoing",
    "tx_state": "confirmed",
    "tx_result": "tesSUCCESS",
    "tx_ledger": 4696959,
    "tx_hash": "55BA3440B1AAFFB64E51F497EFDF2022C90EDB171BBD979F04685904E38A89B7",
    "tx_timestamp": 1391025100000,
    "tx_timestamp_human": "2014-01-29T19:51:40.000Z",
    "tx_fee": "0.000012",
    "tx_src_bals_dec": [{
        "value": "-0.001012",
        "currency": "XRP",
        "issuer": ""
    }],
    "tx_dst_bals_inc": [{
        "value": "0.001",
        "currency": "XRP",
        "issuer": ""
    }]
}
```
+ `tx_result` will be `tesSUCCESS` if the transaction was successfully processed and written into the Ripple ledger. If it was unsuccessful but a transaction fee was claimed the code will start with `tec`. More information about transaction errors can be found on the [Ripple Wiki](https://ripple.com/wiki/Transaction_errors).
+ `tx_timestamp` is the UNIX timestamp for when the transaction was validated, or the number of milliseconds since January 1st, 1970 (00:00 UTC)
+ `tx_timestamp_human` is the transaction validation time represented in the format `YYYY-MM-DDTHH:mm:ss.sssZ`. The timezone is always UTC as denoted by the suffix "Z"
+ `tx_fee` is the network transaction fee charged for processing the transaction. For more information on fees, see the [Ripple Wiki](https://ripple.com/wiki/Transaction_fees), but note that the amount here is [NOT expressed in XRP drops](#no-xrp-drops)
+ `tx_src_bals_dec` is an array of [`Amount`](#1-amount) objects representing all of the balance changes of the `src_address` caused by the payment. Note that this includes the `tx_fee`
+ `tx_dst_bals_inc` is an array of [`Amount`](#1-amount) objects representing all of the balance changes of the `dst_address` caused by the payment

----------

#### 3. Notification

Notifications are new type of object not used elsewhere on the Ripple Network but intended to simplify the process of monitoring accounts for new activity.

If there is a new `Notification` for an account it will contain information about the type of transaction that affected the account, as well as a link to the full details of the transaction and a link to get the next notification. 


If there is a new `notification` for an account, it will come in this format:

```js
{
  "address": "rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz",
  "type": "payment",
  "tx_direction": "outgoing",
  "tx_state": "confirmed",
  "tx_result": "tesSUCCESS",
  "tx_ledger": 4696959,
  "tx_hash": "55BA3440B1AAFFB64E51F497EFDF2022C90EDB171BBD979F04685904E38A89B7",
  "tx_timestamp": 1391025100000,
  "tx_timestamp_human": "2014-01-29T19:51:40.000Z",
  "tx_url": "http://ripple-rest.herokuapp.com/api/v1/addresses/rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz/payments/55BA3440B1AAFFB64E51F497EFDF2022C90EDB171BBD979F04685904E38A89B7",
  "next_notification_url": "http://ripple-rest.herokuapp.com/api/v1/addresses/rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz/next_notification/55BA3440B1AAFFB64E51F497EFDF2022C90EDB171BBD979F04685904E38A89B7"
  "confirmation_token": "55BA3440B1AAFFB64E51F497EFDF2022C90EDB171BBD979F04685904E38A89B7"
}
```
+ `tx_timestamp` is the UNIX timestamp for when the transaction was validated, or the number of milliseconds since January 1st, 1970 (00:00 UTC)
+ `tx_timestamp_human` is the transaction validation time represented in the format `YYYY-MM-DDTHH:mm:ss.sssZ`. The timezone is always UTC as denoted by the suffix "Z"
+ `tx_url` is a URL that can be queried to retrieve the full details of the transaction. If it the transaction is a payment it will be returned in the `Payment` object format, otherwise it will be returned in the standard Ripple transaction format
+ `next_notification_url` is a URL that can be queried to get the notification following this one for the given address
+ `confirmation_token` is a temporary string that is returned upon transaction submission and can be matched against account notifications to confirm that the transaction was processed and written into the Ripple ledger


If there are no new notifications, the empty `Notification` object will be returned in this format:
```js
{
  "address": "rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz",
  "type": "none",
  "tx_direction": "",
  "tx_state": "empty",
  "tx_result": "",
  "tx_ledger": "",
  "tx_hash": "",
  "tx_timestamp": "",
  "tx_timestamp_human": "",
  "tx_url": "",
  "next_notification_url": "http://ripple-rest.herokuapp.com/api/v1/addresses/rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz/next_notification/55BA3440B1AAFFB64E51F497EFDF2022C90EDB171BBD979F04685904E38A89B7",
  "confirmation_token": ""
}
```
+ `type` will be `none` if there are no new notifications
+ `tx_state` will be `pending` if there are still transactions waiting to clear and `empty` otherwise
+ `next_notification_url` will be provided whether there are new notifications or not so that that field can always be used to query the API for new notifications.


----------

## Available API Routes

1. [Notifications](#1-notifications)
    + [`GET /api/v1/addresses/:address/next_notification`](#get-apiv1addressesaddressnext_notification)
    + [`GET /api/v1/addresses/:address/next_notification/:prev_tx_hash`](#get-apiv1addressesaddressnext_notificationprev_tx_hash)
2. [Payments](#2-payments)
    + [`GET /api/v1/addresses/:address/payments/:dst_address/:dst_amount`](docs/REF.md#get-apiv1addressesaddresspaymentsdst_addressdst_amount)
    + [`POST /api/v1/addresses/:address/payments`](#post-apiv1addressesaddresspayments)
    + [`GET /api/v1/addresses/:address/payments/:tx_hash`](#get-apiv1addressesaddresspaymentstx_hash)
3. [Standard Ripple Transactions](#3-standard-ripple-transactions)
    + [`GET /api/v1/addresses/:address/txs/:tx_hash`](#get-apiv1addressesaddresstxstx_hash)
4. [Server Info](#4-server-info)
    + [`GET /api/v1/status`](#get-apiv1status)

----------

### 1. Notifications

__________

#### GET /api/v1/addresses/:address/next_notification

Retrieve the most recent notification for a particular account from the connected rippled.

This uses the [`Notification` Object format](#3-notification).

Response:
```js
{
    "success": true,
    "notification": { 
      /* Notification */ 
    }
}
```

Or if there are no new notifications :

```js
{
    "success": true,
    "notification": { 
      /* Notification with "type": "none" and "tx_state" either "empty" or "pending" */
    }
}
```



__NOTE:__ This command relies on the connected `rippled`'s historical database so it may not work properly on a newly started `rippled` server.
__________

#### GET /api/v1/addresses/:address/next_notification/:prev_tx_hash

Retrieve the next notification after the given `:prev_tx_hash` for a particular account from the connected rippled.

This uses the [`Notification` Object format](#3-notification).

Response:
```js
{
    "success": true,
    "notification": { 
      /* Notification */ 
    }
}
```
Or if there were any failed outgoing transactions not yet reported:
```js
{
  "success": true,
  "notification": {
    /* Notification with "tx_state" as "failed" */
  }
}
```
Or if there are no new notifications:

```js
{
    "success": true,
    "notification": { 
      /* Notification with "type": "none" and "tx_state" either "empty" or "pending" */
    }
}
```


__NOTE:__ This command relies on the connected `rippled`'s historical database so it may respond with an error even for a valid transaction hash if run on a newly started `rippled` server without full history.
__________

### 2. Payments

__________

#### `GET /api/v1/addresses/:address/payments/:dst_address/:dst_amount`

Generate possible payments for a given set of parameters. This is a wrapper around the [Ripple path-find command](https://ripple.com/wiki/RPC_API#path_find) that returns an array of [`Payment Objects`](#2-payment), which can be submitted directly to [`POST /api/v1/addresses/:address/payments`](#post-apiv1addressesaddresspayments).

This uses the [`Payment` Object format](#2-payment).

The `:dst_amount` parameter uses `+` to separate the `value`, `currency`, and `issuer` fields. For XRP the format is `0.1+XRP` and for other currencies it is `0.1+USD+r...`, where the `r...` is the Ripple address of the currency's issuer.

__NOTE:__ This command may be quite slow. If the command times out, please try it again.

Response:
```js
{
    "success": true,
    "payments": [{
      /* Payment */
    }, {
      /* Payment */  
    }...]
}
```
Or if no paths can be found:
```js
{
    "success": false,
    "error": "No paths found",
    "message": "Please ensure that the src_address has sufficient funds to exectue the payment. If it does there may be insufficient liquidity in the network to execute this payment right now"
}
```

__________

#### POST /api/v1/addresses/:address/payments

Submit a payment in the [`Payment` Object](#2-payment) format.

__DO NOT SUBMIT YOUR SECRET TO AN UNTRUSTED REST API SERVER__ -- this is the key to your account and your money. If you are using the test server provided, only use test accounts to submit payments.

Request JSON:
```js
{
  "secret": "s...",
  "payment": { /* Payment */ }
}
```

Response:
```js
{
    "success": true,
    "confirmation_token": "55BA3440B1AAFFB64E51F497EFDF2022C90EDB171BBD979F04685904E38A89B7"
}
```
Or if there is a problem with the transaction:
```js
{
  "success": false,
  "error": "tecPATH_DRY", // A full list of error codes can be found at https://ripple.com/wiki/Transaction_errors
  "message": "Path could not send partial amount. Please ensure that the src_address has sufficient funds (in the src_amount currency, if specified) to execute this transaction."
}
```
More information about transaction errors can be found on the [Ripple Wiki](https://ripple.com/wiki/Transaction_errors).

Save the `confirmation_token` to check for transaction confirmation by matching that against new `notification`'s.

Payments cannot be cancelled once they are submitted.

__________

#### GET /api/v1/addresses/:address/payments/:tx_hash

Get a particular payment for a particular account.

This uses the [`Payment` Object format](#2-payment).

Response:
```js
{
    "success": true,
    "payment": {
        /* Payment */
    }
}
```
Or if the payment cannot be found in the connected `rippled`'s historical database:
```js
{
  "success": false,
  "error": "Cannot locate transaction.",
  "message": "This may be the result of an incomplete rippled historical database or that transaction may not exist."
}
```


__NOTE:__ This command relies on the connected `rippled`'s historical database so it may respond with an error even for a valid transaction hash if run on a newly started `rippled` server without full history.

__________

### 3. Generic Ripple Transactions

__________

#### GET /api/v1/addresses/:address/txs/:tx_hash

Gets a particular transaction in the standard Ripple transaction JSON format.

```js
{
  "success": true,
  "tx": {
    "Account": "rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz",
    "Amount": "1000",
    "Destination": "rNw4ozCG514KEjPs5cDrqEcdsi31Jtfm5r",
    "Fee": "12",
    "Flags": 0,
    "Sequence": 8,
    "SigningPubKey": "025B32A54BFA33FB781581F49B235C0E2820C929FF41E677ADA5D3E53CFBA46332",
    "TransactionType": "Payment",
    "TxnSignature": "304602210093232CDAD8EACB4075F80767FEED2126DFFD02E38B31C90F6FC06402454506ED022100D6E71C90CC42F1632761614055608996E8C866094E8933D1AE642528140D63E6",
    "hash": "55BA3440B1AAFFB64E51F497EFDF2022C90EDB171BBD979F04685904E38A89B7",
    "inLedger": 4696959,
    "ledger_index": 4696959,
    "meta": {
      "AffectedNodes": [
        {
          "ModifiedNode": {
            "FinalFields": {
              "Account": "rKXCummUHnenhYudNb9UoJ4mGBR75vFcgz",
              "Balance": "173504506",
              "Flags": 0,
              "OwnerCount": 1,
              "Sequence": 9
            },
            "LedgerEntryType": "AccountRoot",
            "LedgerIndex": "58D2E252AE8842B950C960B6BC7A3319762F1C66E3C35985A3B160479EFEDF23",
            "PreviousFields": {
              "Balance": "173505518",
              "Sequence": 8
            },
            "PreviousTxnID": "E787C5FA1130C13F400271188401DE3D1C28D87A957241B65E69162D2F4CACBC",
            "PreviousTxnLgrSeq": 4696951
          }
        },
        {
          "ModifiedNode": {
            "FinalFields": {
              "Account": "rNw4ozCG514KEjPs5cDrqEcdsi31Jtfm5r",
              "Balance": "26419669",
              "Flags": 0,
              "OwnerCount": 1,
              "Sequence": 86
            },
            "LedgerEntryType": "AccountRoot",
            "LedgerIndex": "FA39C6EC43AA870B5E9ED592EF683CC1134DB60746C54A997B6BEAE366EF04C9",
            "PreviousFields": {
              "Balance": "26418669"
            },
            "PreviousTxnID": "E787C5FA1130C13F400271188401DE3D1C28D87A957241B65E69162D2F4CACBC",
            "PreviousTxnLgrSeq": 4696951
          }
        }
      ],
      "TransactionIndex": 0,
      "TransactionResult": "tesSUCCESS"
    },
    "validated": true
  }
}
```
Or if the transaction cannot be found in the connected `rippled`'s historical database:
```js
{
  "success": false,
  "error": "Cannot locate transaction.",
  "message": "This may be the result of an incomplete rippled historical database or that transaction may not exist."
}
```

__NOTE:__ This command relies on the connected `rippled`'s historical database so it may respond with an error even for a valid transaction hash if run on a newly started `rippled` server without full history.

__________

### 4. Server Info

__________

#### GET /api/v1/status

Response:
```js
{
  "api_server_status": "online",
  "rippled_server_url": "wss://s_west.ripple.com:443",
  "rippled_server_status": {
    "info": {
      "build_version": "0.21.0-rc2",
      "complete_ledgers": "32570-4805506",
      "hostid": "BUSH",
      "last_close": {
        "converge_time_s": 2.011,
        "proposers": 5
      },
      "load_factor": 1,
      "peers": 51,
      "pubkey_node": "n9KNUUntNaDqvMVMKZLPHhGaWZDnx7soeUiHjeQE8ejR45DmHyfx",
      "server_state": "full",
      "validated_ledger": {
        "age": 2,
        "base_fee_xrp": 0.00001,
        "hash": "2B79CECB06A500A2FB92F4FB610D33A20CF8D7FB39F2C2C7C3A6BD0D75A1884A",
        "reserve_base_xrp": 20,
        "reserve_inc_xrp": 5,
        "seq": 4805506
      },
      "validation_quorum": 3
    }
  },
  "api_documentation_url": "https://github.com/ripple/ripple-rest"
}
```
Or if the server is not connected to the Ripple Network:
```js
{
  "success": false,
  "error": "Cannot connect to the Ripple network.",
  "message": "Please check your internet connection and server settings and try again."
}
```

__________