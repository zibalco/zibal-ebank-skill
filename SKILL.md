---
name: zibal-ebank
description: "Integrate the Zibal ebank API (زیبال، سرویس بانکداری شرکتی زیبال، Zibal EBank) into a server-side project in any stack (Node/Express, Next.js, Laravel, Django, FastAPI and others). Use when the user wants to create, inquire, list or cancel fund transfers or payouts (انتقال وجه، ثبت درخواست انتقال وجه، استعلام انتقال وجه، لغو انتقال وجه، تسویه، checkout), read account statements (صورت حساب), balance (موجودی) or the list of connected bank accounts (لیست حساب‌ها), list, inquire, approve or reject identified deposits (واریزی‌ها، شناسه واریز، تایید یا رد واریز، identified-payment), handle the transfer webhook (callbackUrl، وب‌هوک), or map Zibal ebank result and status codes. Covers IBAN (شبا) transfers via Paya, Satna, internal and Pol (پایا، ساتنا، پل), amounts in Rial, uniqueCode duplicate protection, unknown outcomes after timeouts, and security rules for code that moves money. Not for the Zibal payment gateway (IPG, درگاه پرداخت)."
---

# Zibal ebank integration

Docs snapshot: OpenAPI `info.version` 1.0.0, received 2026-11-02. The docs carry no modified date.

This skill covers Zibal's ebank API (بانکداری شرکتی) only. It is not the payment gateway (IPG). Do not reuse IPG endpoints, parameters, codes or flows here.

Everything Zibal-specific comes from the files in `references/`. If something you need is not there, the docs do not define it. Say so to the user instead of guessing.

## Workflow

1. **Detect the stack.** Read the project's manifest (`package.json`, `composer.json`, `requirements.txt`, `pyproject.toml` and so on), framework config and existing HTTP client and database layer. All Zibal calls run on the server.
2. **Identify the service.** Map the request to endpoints in [references/api.md](references/api.md):
   - Pay out or transfer money: `checkout/create`, then `checkout/inquire`. Moves money.
   - Cancel a pending transfer: `checkout/cancel`. Changes state.
   - Report on transfers: `checkout/list`. Read-only.
   - Balance, statement, account list: `balance`, `statement/list`, `account/list`. Read-only.
   - Incoming deposits: `identified-payment/list` and `identified-payment/inquire`, which are read-only, and `identified-payment/change-status`, which moves money when it rejects a deposit.
3. **Load the closest example** from `examples/` and adapt it to the project's conventions:
   - [examples/node-express.md](examples/node-express.md): Express or any Node server
   - [examples/nextjs-app-router.md](examples/nextjs-app-router.md): Next.js route handlers
   - [examples/laravel.md](examples/laravel.md): Laravel or other PHP
   - [examples/django.md](examples/django.md): Django or Flask
   - [examples/fastapi.md](examples/fastapi.md): FastAPI or other async Python

   Each example shares one design: an operations table, a claim before sending, an outcome mapper, a reconcile job, a webhook receiver and input validation. For another stack, keep that design and use the stack's idiomatic HTTP client and database.
4. **Configure the environment.** Use `ZIBAL_ACCESS_TOKEN` and `ZIBAL_ACCOUNT_ID`, plus the optional variables in [references/auth.md](references/auth.md). Add them to the project's env example with empty values. Never ask the user for real credential values, and never write them into files.
5. **Implement the call** exactly as [references/api.md](references/api.md) documents it: base URL `https://api.zibal.ir`, trailing slash on every path, `Authorization: Bearer` header, JSON body, documented field names and types.
6. **Persist request and result.** Save a row with a unique key before calling Zibal. Afterwards, store `trackerId`, the per-checkout `id`, `status` and `refCode`, and `result`.
7. **Handle result and status codes** with [references/result-codes.md](references/result-codes.md). Map every code to a local state. Unknown codes are unknown outcomes.
8. **Protect and schedule.** Replace the examples' `X-Internal-Key` guard with the project's own authentication and authorization. A route that moves money must require an authorized operator or service. Schedule the reconcile job, one run at a time, and give `needs_review` operations a place where a person sees them.
9. **Test** against a local mock built from `references/api.md`. There is no sandbox: any valid token acts on real accounts. Cover success, each failure group, timeout, duplicate request and malformed input. Never call money-moving endpoints against the live API to test.

## Non-negotiable rules

**Credentials**
- Keep `ZIBAL_ACCESS_TOKEN` on the server. Never send it to a browser or mobile client, never put it in a client bundle (no `NEXT_PUBLIC_` or similar), and never log it.
- Zibal answers only from IPs declared in the panel. The server needs a stable outbound IP.

**Sensitive data**
- Never log a full IBAN, card number, account number or national ID. Mask them, for example `IR12******************3456`.
- Do not log raw Zibal responses. They contain names, IBANs and bank descriptions.
- Do not echo submitted values in validation errors.

**Amounts and limits**
- Amounts are integers in **Rial**. Do not convert from Toman unless the user's domain clearly uses Toman, and then convert explicitly in one place.
- `amount` must be at least 10,000 Rial (result 9).
- `iban` must match `^IR[0-9]{24}$`. Use full-string matching, not `\d`, and not a `$` that allows a trailing newline.
- `delay` is from -1 to 10080 minutes, and `1` means one day.
- `reasonCode` is 1 to 19. It is required when the source bank is Saman or Pasargad.
- The docs give no numeric per-transaction cap. With `split` true the default, Zibal may split one request into several checkouts.

**Validation**
- Validate every input on the server with strict types. Reject strings for numbers, floats, booleans, `null`, unknown keys (including `__proto__` and `constructor`), oversized bodies and overlong strings.
- Never take `accountId`, `uniqueCode` or `callbackUrl` from the client. They come from server configuration or the server.

**Calls**
- Set a timeout on every external call. Disable redirects on money-moving calls. Never enable automatic HTTP-client retries for POST.
- Bypass caches for every GET, including framework fetch caches.

**Money movement**
- **Never blindly retry a money-moving call after a timeout or an unknown response.** A timeout does not mean failure. Treat as `unknown`: timeouts, connection errors, HTTP 5xx, 3xx, unparsable bodies and any `result` not in the table.
- Resolve `unknown` the way the docs define: `checkout/inquire` by `uniqueCode` for transfers, and `identified-payment/inquire` by `verifyStatus` for deposit decisions. If inquire still cannot settle it within your review window, mark the operation `needs_review` for a human. Do not send it again automatically.
- Prevent duplicate execution with a persisted unique key and a conditional update. Insert the row with a unique constraint, then claim it with `UPDATE ... WHERE state = 'created'`. Only the request that changed one row may call Zibal.
- Send a fresh, globally unique `uniqueCode`, such as a UUID, on every create. It is the documented duplicate protection, and it is scoped per source account.
- Always send `accountId` on create and store it. Cancel needs it, and the response does not return it.
- `identified-payment/change-status` has no documented idempotency key. Check `verifyStatus` is 0 before sending, send once, and resolve unknown outcomes through inquire.
- Apply state changes with conditional updates that allow only forward transitions, so a stale inquire cannot undo a newer state.

**Success criteria**
- `result` 1 on create means **accepted**, not paid.
- A transfer succeeded only when **every** `data.checkouts[]` item has `status` 3 and a non-null `refCode`, and `uniqueCode` and `totalAmount` match what you sent.
- Statuses -1, 0 and 20 are pending. 5 is failed. 4 is returned. Mixed final statuses need review.
- Status 3 can later become 4. Keep re-checking succeeded transfers for a while.
- Cancel: `result` 1 does not mean everything was cancelled. Read `cancelled_checkouts` and `uncancellable_checkouts`, then confirm by inquire.
- Balance, account list and change-status return no `result`. Check `balance`, `data` and `success === true` instead.

**Webhooks**
- The webhook has no signature, secret or IP list, and is sent **once** per change, one per checkout when split. Treat it only as a trigger: read the unique code, ignore every other field, and call `checkout/inquire`.
- The webhook payload is snake_case (`unique_code`), unlike inquire's camelCase.
- Because the webhook can be lost, also run a scheduled reconcile.
- Exempt only the webhook route from CSRF protection.

## References: when to read each

| File | Read when |
| :--- | :--- |
| [references/api.md](references/api.md) | Writing any call. Every endpoint, parameter and response field. |
| [references/auth.md](references/auth.md) | Setting up credentials, env vars, IP allowlist, auth failures |
| [references/result-codes.md](references/result-codes.md) | Mapping `result`, `status`, `failReason`, `reasonCode`, statement `type` and smaller enums. Includes the original Persian text. |
| [references/webhooks.md](references/webhooks.md) | Using `callbackUrl` or building the receiver |
| [references/security-checklist.md](references/security-checklist.md) | Before you finish. Go through every item. |

## Common problems

| Symptom | Cause |
| :--- | :--- |
| result 7 | The server's outbound IP is not declared in the panel. Common on serverless hosts and local machines. |
| result 4 | The token lacks access to that API. Grant it in the panel. |
| result 2, 3 or 8 | The `Authorization: Bearer` header is missing or wrong, or the token is disabled. |
| result 9 | Amount below 10,000 Rial, often a Toman value sent as Rial. |
| result 21 | IBAN is not 26 characters starting with `IR`, or contains spaces or hyphens. |
| result 48 | `split` was false and the amount is over the per-transaction cap. The cap differs during Satna hours. |
| Saman or Pasargad transfers rejected | `reasonCode` is missing. It is required for those source banks. |
| Transfer marked paid but never arrived | `result` 1 was treated as success. Check `checkouts[].status`. |
| Paid transfer later bounced | Status moved from 3 to 4. Keep re-checking succeeded transfers. |
| Delay of one day instead of one minute | `delay: 1` means one day. |
| Webhook handler finds no code | It read camelCase. The sample payload uses `unique_code`. |
| Inquire returns the wrong transfer | A `uniqueCode` was reused across accounts. Inquire has no `accountId`. |
| Deposit inquire breaks the parser | `data` is an array even for one transaction ID. |
| Django returns 403 to Zibal's webhook | `CsrfViewMiddleware`. Exempt the webhook view. Found in testing. |
| FastAPI error responses contain the IBAN | The default 422 handler echoes input. Replace it. Found in testing. |
| `next build` fails on missing env, or inquire shows stale data | Read config lazily and use `cache: 'no-store'`. Found in testing. |
| Env vars missing under `php artisan serve` | It passes only an allowlist of variables to workers. Use `.env`. Found in testing. |
