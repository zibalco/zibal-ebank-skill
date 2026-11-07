# Zibal ebank API reference

Base URL: `https://api.zibal.ir`

All requests and responses are JSON. Every request needs `Authorization: Bearer {{ACCESS TOKEN}}`, see [auth.md](auth.md). Every path ends with a slash. Keep it.

No endpoint documents rate limits, a maximum `size`, or a maximum date range. Do not assume any.

| Endpoint | Method | Effect |
| :--- | :--- | :--- |
| [`/ebank/v1/account/checkout/create/`](#create-transfer) | POST | **Moves money** |
| [`/ebank/v1/account/checkout/inquire/`](#inquire-transfer) | GET | Read-only |
| [`/ebank/v1/account/checkout/list/`](#list-transfers) | GET | Read-only |
| [`/ebank/v1/account/checkout/cancel/`](#cancel-transfer) | POST | **Changes state** |
| [`/ebank/v1/account/statement/list/`](#statement-list) | GET | Read-only |
| [`/ebank/v1/account/balance/`](#balance) | GET | Read-only |
| [`/ebank/v1/account/list/`](#account-list) | GET | Read-only |
| [`/ebank/v1/account/identified-payment/list/`](#list-deposits) | GET | Read-only |
| [`/ebank/v1/account/identified-payment/change-status/`](#approve-or-reject-deposit) | POST | **Changes state, and moves money on reject** |
| [`/ebank/v1/account/identified-payment/inquire/`](#inquire-deposit) | GET | Read-only |

Error bodies, when HTTP is 400, 403 or 500:

| Field | Type | Description |
| :--- | :--- | :--- |
| `result` | integer | Code from the `result` table |
| `message` | string | Message with the request result |

## Transfers (انتقال وجه)

### Create transfer

ثبت درخواست انتقال وجه. Registers a new transfer from the merchant account on the ebank service.

`POST /ebank/v1/account/checkout/create/`

Request body:

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `amount` | integer (int64) | Yes | Transfer amount in **Rial**. Minimum 10,000 Rial, from result 9. |
| `iban` | string | Yes | Destination IBAN. Pattern `^IR[0-9]{24}$`. |
| `accountId` | string | No | Merchant account ID, from the ebank section of the panel. If omitted, Zibal picks an account by the transfer strategy settings. |
| `reasonCode` | integer | Conditional | Transfer reason, from the [`reasonCode` table](result-codes.md#reasoncode-table). **Required when the source bank is Saman or Pasargad.** For other banks, if omitted, Zibal sends "پرداخت قرض و تادیه دیون" (debt settlement) to the bank. |
| `delay` | integer | No | Delay. Minimum -1, maximum 10080. Default -1. See below. |
| `uniqueCode` | string | No | Your unique ID for this transfer. See below. |
| `description` | string | No | Transfer description. |
| `paymentNumber` | string | No | Deposit ID (شناسه واریز). |
| `callbackUrl` | string | No | Webhook URL, see [webhooks.md](webhooks.md). |
| `split` | boolean | No | Allow splitting into several transactions. Default true. See below. |

`delay` values:

| Value | Meaning |
| :--- | :--- |
| `-1` | Send to the bank immediately. The default. |
| `0` | Nearest possible cycle |
| `1` | One day delay |
| Other value up to `10080` | That many minutes. `60` is one hour, `10080` is the maximum (7 days). |

`delay` is in minutes, except that `1` means one day. A one-minute delay cannot be expressed.

`uniqueCode` rules:

- It is unique **per source account** (`accountId`). If the same code is sent twice, Zibal does not accept the second request. This is the only documented duplicate protection.
- The docs define no result code for that rejection.
- `checkout/inquire` looks up by `uniqueCode` and takes no `accountId`, so a code reused across accounts cannot be told apart. Generate a globally unique value.
- The docs do not state a length limit or character set. Their samples show a short number and a UUID.

`split` rules:

- `true`, the default: if the amount exceeds the per-transaction cap of the source account, Zibal divides it into several transfers, all returned in `checkouts`.
- `false`: the transfer is never divided. If it cannot be done in one transaction, the request is rejected with result 48, no transfer is registered and no fee is deducted.
- The cap depends on the method (Paya, Satna or internal). For accounts with Satna, a higher cap applies during Satna working hours, so the same amount can be accepted or rejected at different hours. The docs give no numeric caps.

Response 200:

| Field | Type | Description |
| :--- | :--- | :--- |
| `result` | integer | Code from the `result` table |
| `message` | string | Message with the request result |
| `data` | object | Transfer object below |

**`result` 1 means the request was accepted. It does not mean the money arrived.** Read `data.checkouts[].status`.

### Transfer object

Returned as `data` by create and inquire, and as each `data[]` item by list.

| Field | Type | Nullable | Description |
| :--- | :--- | :--- | :--- |
| `totalAmount` | integer (int64) | No | Total transfer amount in Rial |
| `destination` | string | No | Destination IBAN |
| `destinationType` | integer | No | `1` IBAN, `2` card, `3` account |
| `totalWage` | integer | Yes | Total Zibal fee for this transfer. Unit not stated. |
| `trackerId` | string | No | Tracking ID assigned to this transfer |
| `receipt` | string | No | Receipt link for viewing transfer details |
| `description` | string | No | Transfer description |
| `paymentNumber` | string | Yes | Deposit ID |
| `uniqueCode` | string | No | Your unique ID |
| `createdAt` | string (date-time) | No | Request time in ISO format |
| `jalaliCreatedAt` | string | No | Request time in the Jalali calendar |
| `subMerchant` | object | No | Beneficiary, see below |
| `checkouts` | array | No | One entry per transaction, see below |

`subMerchant`:

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | string | Beneficiary ID |
| `name` | string | Beneficiary name |
| `iban` | string | Destination IBAN linked to the beneficiary |

`checkouts[]`:

| Field | Type | Description |
| :--- | :--- | :--- |
| `amount` | integer (long) | Transfer amount in Rial |
| `status` | integer | From the [`status` table](result-codes.md#transfer-status-table-datacheckoutsstatus) |
| `wage` | integer | Zibal fee for this transfer. Unit not stated. |
| `refCode` | string | Bank reference number. `null` means the transfer is not processed yet. |
| `transactionType` | string | `paya`, `internal`, `satna` or `pol` |
| `sentToBankAt` | string (ISO date) | Time sent to the bank. `null` means not processed yet. |
| `jalaliSentToBankAt` | string | Jalali time sent to the bank. `null` means not processed yet. |
| `settledAt` | string (ISO date) | Time the amount reached the destination. `null` means not processed yet. |
| `jalaliSettledAt` | string | Jalali time the amount reached the destination. `null` means not processed yet. |
| `predictedSettledAt` | string (ISO date) | Predicted settle time, if not processed yet |
| `jalaliPredictedSettledAt` | string | Jalali predicted settle time, if not processed yet |
| `id` | string | Checkout ID. Use it to cancel. |
| `failReason` | integer | From the [`failReason` table](result-codes.md#failreason-table) |

The docs do not state a timezone for transfer timestamps. The docs' sample of `checkout/list` omits `id` from `checkouts[]`, though the details table lists it. Do not rely on it from list. Read it from create or inquire.

### Inquire transfer

استعلام انتقال وجه. Inquires a transfer by its unique ID. **This is the documented way to check an outcome.**

`GET /ebank/v1/account/checkout/inquire/`

Query parameters:

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `uniqueCode` | string | One of the two | Your unique ID |
| `trackerId` | string | One of the two | Transfer tracking ID |

Response 200: `result`, `message` and a `data` [transfer object](#transfer-object).

Result 14 means the transfer was not found. The docs do not say how soon a new transfer becomes visible, so right after a timeout, 14 does not prove the transfer was never created.

### List transfers

دریافت لیست انتقال وجه.

`GET /ebank/v1/account/checkout/list/`

Query parameters:

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `accountId` | string | Yes | Bank account ID |
| `fromDate` | string (date-time) | Yes | Range start, for example `2024-01-01T00:00:00.000Z` |
| `toDate` | string (date-time) | Yes | Range end, for example `2024-12-31T23:59:59.000Z` |
| `status` | integer | No | Filter by `status` table value. The docs say 6 values. |
| `size` | integer | No | Records to return. Default 10. |
| `page` | integer | No | Page. Default 1. |

Response 200:

| Field | Type | Description |
| :--- | :--- | :--- |
| `result` | integer | Code from the `result` table |
| `data` | array | [Transfer objects](#transfer-object) |
| `hasMore` | boolean | true means more data exists for this report |

### Cancel transfer

لغو انتقال وجه. Cancels transfers by their checkout IDs.

`POST /ebank/v1/account/checkout/cancel/`

Request body:

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `accountId` | string | Yes | Merchant account ID |
| `checkoutIds` | array of string | Yes | The `checkouts[].id` values from create or inquire |

Response 200:

| Field | Type | Description |
| :--- | :--- | :--- |
| `result` | integer | Code from the `result` table |
| `message` | string | Message with the request result |
| `data.cancelled_checkouts` | array of string | Transfers that were cancelled |
| `data.uncancellable_checkouts` | array of string | Transfers that could not be cancelled |

`result` 1 does not mean everything was cancelled. Read both arrays. These fields are snake_case, unlike the rest of the API.

Create does not return `accountId`, so persist the one you sent. Otherwise you cannot cancel.

## Accounts (حساب‌ها)

### Statement list

دریافت لیست صورت حساب.

`GET /ebank/v1/account/statement/list/`

Query parameters:

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `accountId` | string | Yes | Bank account ID |
| `fromDate` | string (date-time) | Yes | Range start |
| `toDate` | string (date-time) | Yes | Range end |
| `update` | boolean | No | true fetches a fresh inquiry from the bank. Default false. |
| `size` | integer | No | Records to return. Default 10. |
| `page` | integer | No | Page. Default 1. |

Response 200: `result`, `hasMore` (as in list transfers) and `data`, an array of:

| Field | Type | Nullable | Description |
| :--- | :--- | :--- | :--- |
| `side` | integer | No | `1` deposit, `0` withdrawal |
| `balance` | integer (int64) | No | Account balance at that moment. Unit not stated. |
| `amount` | integer (int64) | No | Deposit or withdrawal amount. Unit not stated. |
| `date` | string (date-time) | No | Statement date in Iran timezone |
| `type` | integer | No | From the [`type` table](result-codes.md#type-table-statement-records) |
| `jalaliDate` | string | No | Statement date in Iran timezone, Jalali |
| `relatedIban` | string | Yes | IBAN linked to this record |
| `relatedCard` | string | Yes | Card number linked to this record |
| `paymentId` | string | Yes | Deposit ID |
| `rawDescription` | string | No | Description taken directly from the bank |
| `optionalInformation` | string | Yes | Extra description, usually the reference number |
| `optionalInformation1` | string | Yes | Extra description |
| `recordNumber` | string | No | Unique ID of this record |

Use `recordNumber` to deduplicate imported statement lines. `rawDescription` and `optionalInformation1` can contain personal names and account numbers, so treat them as sensitive.

### Balance

دریافت موجودی.

`GET /ebank/v1/account/balance/`

Query parameters:

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `accountId` | string | Yes | Merchant account ID |
| `update` | boolean | No | true fetches a fresh inquiry from the bank. Default false. |

Response 200, with **no** `result` or `data` wrapper:

| Field | Type | Description |
| :--- | :--- | :--- |
| `balance` | integer (int64) | Balance in Rial |
| `balanceUpdatedAt` | string (date-time) | Balance update time in ISO format |
| `jalaliBalanceUpdatedAt` | string | Balance update time, Jalali |

### Account list

دریافت لیست حساب‌های متصل به بانکداری شرکتی.

`GET /ebank/v1/account/list/`

Query parameters:

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `activeAccounts` | boolean | No | true returns active accounts, false returns inactive accounts |

Response 200, with **no** `result` field:

| Field | Type | Description |
| :--- | :--- | :--- |
| `data` | array | Accounts, see below |
| `total` | integer | Total number of accounts |

`data[]`:

| Field | Type | Description |
| :--- | :--- | :--- |
| `accountId` | string | Account ID |
| `accountName` | string | Account name |
| `accountNumber` | string | Account number |
| `accountIban` | string | Account IBAN |
| `status` | integer | `1` created, `2` awaiting activation, `3` rejected, `4` active |

## Deposits (واریزی‌ها)

### List deposits

دریافت لیست واریزی‌ها.

`GET /ebank/v1/account/identified-payment/list/`

Query parameters:

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `accountId` | string | Yes | Merchant account ID |
| `fromDate` | string (date-time) | Yes | Range start |
| `toDate` | string (date-time) | Yes | Range end |
| `status` | integer | No | `0` pending, `2` approved, `3` rejected, `4` rejected without return, `5` returned |
| `size` | integer | No | Records to return. Default 10. |
| `page` | integer | No | Page. Default 1. |

Response 200: `result`, `hasMore` (as in list transfers) and `data`, an array of:

| Field | Type | Nullable | Description |
| :--- | :--- | :--- | :--- |
| `id` | string | No | Tracking ID. Use it to approve, reject and inquire. |
| `verifyStatus` | integer | No | `0` awaiting approval, `2` approved, `3` rejected |
| `amount` | integer (int64) | No | Deposit amount in Rial |
| `paymentId` | string | No | Deposit ID |
| `sourceIban` | string | Yes | Depositor IBAN |
| `sourceCard` | string | Yes | Depositor card number |
| `sourceName` | string | No | Depositor name |
| `refCode` | string | No | Bank reference number |
| `description` | string | No | Deposit description |
| `jalaliCreatedAt` | string | No | Deposit time, Jalali |
| `jalaliRejectedAt` | string | No | Rejection time, Jalali. Empty string in the docs' sample. |
| `jalaliVerifiedAt` | string | No | Approval time, Jalali |
| `createdAt` | string (date-time) | No | Deposit time in ISO format |
| `rejectedAt` | string | No | Rejection time in ISO format. Empty string in the docs' sample. |
| `verifiedAt` | string (date-time) | No | Approval time in ISO format |

### Approve or reject deposit

تایید یا رد واریز.

`POST /ebank/v1/account/identified-payment/change-status/`

Request body:

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `accountId` | string | Yes | Merchant account ID |
| `transactionId` | string | Yes | Transaction ID, the `id` from the deposit list |
| `status` | integer | Yes | `2` approve, `3` reject |

Response 200, with **no** `result` field:

| Field | Type | Description |
| :--- | :--- | :--- |
| `success` | boolean | Request result |
| `message` | string | Message with the request result |
| `data.refundRequestTrackerId` | string | Tracking ID of the return to the source account, on rejection |

`data` is an empty object on approval. **Rejection sends the money back to the source account**, so treat it as a money-moving call. The docs define no idempotency key for this endpoint and no endpoint to check a refund by `refundRequestTrackerId`. The docs do not define what happens if you change a deposit that was already decided.

### Inquire deposit

استعلام واریز. **This is the way to check a decision's outcome, through `verifyStatus`.**

`GET /ebank/v1/account/identified-payment/inquire/`

Query parameters:

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `accountId` | string | Yes | Merchant account ID |
| `transactionId` | string | Yes | Transaction ID |

Response 200:

| Field | Type | Description |
| :--- | :--- | :--- |
| `result` | integer | Code from the `result` table |
| `data` | array | Deposits, one item per the list below. It is an **array**, even for one ID. |
| `hasMore` | boolean | true means more data exists for this report |

`data[]` has the same fields as [list deposits](#list-deposits), except that `sourceCard` is not listed and `sourceIban` is not marked nullable.

The docs define no "not found" result for deposits. Treat an empty `data` or any non-1 `result` as unresolved.
