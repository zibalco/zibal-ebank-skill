# Security checklist

`[docs]` comes from Zibal's docs. `[practice]` is engineering practice for this integration and is not a Zibal requirement.

## Credentials

- [ ] `[docs]` Every request carries `Authorization: Bearer {{ACCESS TOKEN}}`.
- [ ] `[docs]` Requests leave from an IP declared in the panel.
- [ ] `[docs]` The token is restricted in the panel to the IPs and API access this app needs.
- [ ] `[practice]` The token lives only in server-side environment variables or a secret manager.
- [ ] `[practice]` The token never reaches a browser, mobile app, client bundle, repository, log or error report.
- [ ] `[practice]` In Next.js, no credential uses the `NEXT_PUBLIC_` prefix.

## Sensitive data

- [ ] `[practice]` Logs never contain a full IBAN, card number, account number or national ID. Mask them.
- [ ] `[practice]` Logs never contain raw Zibal bodies, because they hold names, IBANs and bank descriptions.
- [ ] `[practice]` API responses to your own clients return masked IBANs.
- [ ] `[practice]` Error handlers and validation errors do not echo submitted values.
- [ ] `[practice]` Stored IBANs are protected at rest, with access limited to who needs them.

## Input validation

- [ ] `[docs]` `amount` is an integer in Rial and at least 10,000.
- [ ] `[docs]` `iban` matches `^IR[0-9]{24}$`.
- [ ] `[docs]` `delay` is an integer from -1 to 10080.
- [ ] `[docs]` `reasonCode` is sent for Saman and Pasargad source accounts.
- [ ] `[docs]` `reasonCode` 12 is used only for accounts owned by the same person.
- [ ] `[practice]` Unknown keys are rejected, including `__proto__` and `constructor`.
- [ ] `[practice]` Types are strict. `"10000"`, `10000.5`, `true` and `null` are not valid amounts.
- [ ] `[practice]` Request body size and string lengths are capped.
- [ ] `[practice]` `callbackUrl` comes from server configuration, never from user input.

## Calling Zibal

- [ ] `[docs]` Trailing slashes on paths are kept.
- [ ] `[practice]` Every call has a timeout.
- [ ] `[practice]` Redirects are disabled on money-moving calls.
- [ ] `[practice]` The HTTP client has no automatic retry on POST calls.
- [ ] `[practice]` GET calls bypass every cache, including framework fetch caches.

## Money movement

- [ ] `[docs]` Rial is used everywhere. Do not convert from Toman implicitly.
- [ ] `[docs]` A repeated `uniqueCode` for the same source account is not accepted.
- [ ] `[practice]` `uniqueCode` is sent on every create, although the docs mark it optional.
- [ ] `[docs]` Outcome after a timeout is checked with `checkout/inquire`.
- [ ] `[docs]` Status 3 can become 4, so success is not treated as permanent.
- [ ] `[docs]` Each `checkouts[]` item is evaluated. With `split` there can be several.
- [ ] `[docs]` `null` `refCode` means not processed yet.
- [ ] `[docs]` Cancel responses are read per ID. `result` 1 does not mean all cancelled.
- [ ] `[docs]` Deposit rejection is treated as money-moving, because it triggers a refund.
- [ ] `[practice]` `accountId` is always sent on create, since the uniqueness of `uniqueCode` is per account and cancel needs the ID.
- [ ] `[practice]` `uniqueCode` is globally unique, because inquire has no `accountId` parameter.
- [ ] `[practice]` A row with a unique key is saved **before** the external call.
- [ ] `[practice]` A conditional update claims each operation, so only one worker can send it.
- [ ] `[practice]` State changes use conditional updates so that a stale read cannot undo a newer state.
- [ ] `[practice]` A timeout, 5xx, redirect, unparsable body or unknown result is `unknown`, never `failed`.
- [ ] `[practice]` Money-moving calls are never sent again automatically.
- [ ] `[practice]` Unresolved operations go to `needs_review` for a person.
- [ ] `[practice]` Routes that move money require authentication and authorization in your app.

## Webhooks

- [ ] `[docs]` The receiver accepts POST with JSON and handles one webhook per checkout.
- [ ] `[docs]` The webhook is sent once, so a scheduled job covers lost ones.
- [ ] `[practice]` The body is only a trigger. State comes from `checkout/inquire`.
- [ ] `[practice]` Unknown codes cause no outbound call.
- [ ] `[practice]` The receiver is exempt from CSRF protection, and only the receiver.
- [ ] `[practice]` The endpoint is served over HTTPS and rate limited.
