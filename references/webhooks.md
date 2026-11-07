# Webhooks

Zibal sends a webhook only for transfers created with `callbackUrl`. Deposits, balances and statements have no documented webhook.

## What the docs define

- Zibal sends the webhook **once**, as a POST with a JSON body, when the transfer status changes. The docs describe no retry.
- With a split request, Zibal sends a **separate webhook for each checkout**.
- The webhook is called only on these changes:

| From | To |
| :--- | :--- |
| 0 pending, or 20 locked | 3 success |
| 0 pending, or 20 locked | 5 failed or cancelled |
| 3 success | 4 returned |

No webhook is documented for entering status 20, or for moving from -1 to 0.

## What the docs do not define

- No signature, shared secret, or authentication header.
- No source IP list.
- No required response code or body, and no response deadline.
- No URL rules, though result 5 means `callbackUrl` was rejected.

Because of this, anyone who learns the URL can send a fake webhook. Do not add a signature check. None exists.

## Payload

The docs describe the body as similar to the inquire response, but their sample is wrapped in `data`, uses **snake_case**, and has no `result` or `message`. Values below are placeholders.

```json
{
  "data": {
    "total_amount": 10000,
    "destination": "IR000000000000000000000000",
    "destination_type": 1,
    "total_wage": 0,
    "tracker_id": "155",
    "receipt": "https://r.zib.al/ec/XXXXXX",
    "description": "انتقال وجه کاربر",
    "payment_number": "121655478",
    "unique_code": "17",
    "created_at": "2026-04-11T16:37:43.674000",
    "jalali_created_at": "1405/01/22-16:37:43",
    "sub_merchant": {
      "id": "XXXXXX",
      "name": "ذی‌نفع نمونه",
      "iban": "IR000000000000000000000000"
    },
    "checkouts": [
      {
        "id": "000000000000000000000001",
        "amount": 10000,
        "status": 0,
        "wage": 0,
        "is_cancellable": true,
        "fail_reason": null,
        "statement_track_id": "_000000000_",
        "ref_code": null,
        "transaction_type": null,
        "sent_to_bank_at": null,
        "jalali_sent_to_bank_at": null,
        "settled_at": null,
        "jalali_settled_at": null,
        "predicted_settled_at": "2026-04-12T18:45:00.444375",
        "jalali_predicted_settled_at": "1405/01/23-18:45:00"
      }
    ]
  }
}
```

`is_cancellable` and `statement_track_id` appear only in this sample. They are not in the inquire tables.

## How to handle a webhook

Treat the webhook only as a hint that something changed.

1. Parse the body with a size limit. Read `data.unique_code`. Also accept `data.uniqueCode`, since the docs say the structure is similar to inquire, which is camelCase.
2. Look up the code in your own table. If it is unknown, return 200 and do nothing. Make no call to Zibal for unknown codes.
3. Ignore every other field in the body. Never read status or amount from it.
4. Call `checkout/inquire` with your stored `uniqueCode` and apply that answer through the same conditional update as everything else.
5. Return 200. Log the fact, not the body, because it contains IBANs and names.

Expect the same `unique_code` several times with split transfers. The flow above is safe to repeat.

## You still need a scheduled job

The webhook is sent once and can be lost, so it cannot be your only signal. Run a job that re-inquires every transfer in `unknown` or `processing`, and every `succeeded` transfer within `ZIBAL_RETURN_WATCH_DAYS`, because a success can later become status 4.

The docs set no polling interval or rate limit. Pick a modest interval.

## Testing your receiver

The docs provide a curl for testing your own URL with sample values. It sends a POST to your receiver, not to Zibal:

```bash
curl --location 'https://example.com/callback' \
  --header 'Content-Type: application/json' \
  --data '{"data":{"unique_code":"17","tracker_id":"155","checkouts":[{"id":"000000000000000000000001","status":0}]}}'
```

The body above is a shortened form of the payload. An unknown `unique_code` must return 200 and change nothing.
