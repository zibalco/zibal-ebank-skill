# Authentication and credentials

## Credential

Zibal authenticates ebank requests with one credential, the `ACCESS TOKEN`.

Every request must carry this header:

```
Authorization: Bearer {{ACCESS TOKEN}}
```

Create tokens in the Zibal user panel under: حساب کاربری / توسعه دهندگان / API Token ها
(user account, developers, API Tokens).

- One panel can hold unlimited access tokens, each with access to different APIs.
- The panel can restrict a token to chosen IPs.
- The panel can limit a token's access level.

Tokens have no documented expiry, rotation or refresh flow. There is no login or token exchange endpoint, so do not build one.

## Environment variables

The docs name the credential but not an environment variable. Use these names consistently:

| Variable | Required | Purpose |
| :--- | :--- | :--- |
| `ZIBAL_ACCESS_TOKEN` | Yes | The `ACCESS TOKEN` |
| `ZIBAL_ACCOUNT_ID` | Yes for money-moving calls | The `accountId`, shown in the ebank section of the panel or returned by the account list endpoint |
| `ZIBAL_BASE_URL` | No | Defaults to `https://api.zibal.ir`. Change it only to point tests at a local mock. |
| `ZIBAL_CALLBACK_URL` | No | Public HTTPS URL of your webhook receiver |
| `ZIBAL_TIMEOUT_MS` | No | Timeout for each Zibal call. The docs set no value. The examples default to 15000. |
| `ZIBAL_REVIEW_AFTER_MS` | No | How long an unresolved outcome may stay unknown before it goes to `needs_review`. The docs set no value. The examples default to 900000. |
| `ZIBAL_RETURN_WATCH_DAYS` | No | How long a succeeded transfer keeps being re-checked for a return. The docs set no value. The examples default to 7. |

## IP allowlist

Zibal responds only to requests sent from the IPs you declared. Consequences:

- Run calls from a server with a stable outbound IP and declare that IP in the panel.
- Result 7 means the sending IP is not accepted.
- Serverless platforms and any host with a changing outbound IP need a static egress IP. A local machine usually has a different IP from production, so a token restricted to the production IP will not work there.

## Rules

- Keep the token on the server. It must never reach a browser, mobile app, client bundle, repository, log or error report.
- Prefer one token per application, limited to the APIs that application needs and to its IPs.
- Do not put the token in a URL.
- There is no sandbox and no test credential. Every valid token acts on real accounts. Test against a local mock.

## Auth failures

| result | Meaning | Fix |
| :--- | :--- | :--- |
| 2 | Credential not sent correctly | Check the `Authorization: Bearer` header |
| 3 | Credential incorrect | Check the value in `ZIBAL_ACCESS_TOKEN` |
| 4 | No access to this service | Grant this token access to the API in the panel |
| 7 | Sending IP not valid | Declare the server's outbound IP in the panel |
| 8 | Credential disabled | Enable it or create a new token in the panel |

Do not retry these automatically. A person must change configuration.

## Getting help from Zibal

- All your requests are visible in the panel under توسعه‌دهندگان - فراخوانی‌ها (developers, calls). Use this when reviewing an unknown outcome by hand.
- For an error, the docs ask you to check your sent values, then send the error with the sent and received values through a ticket in the user panel. Never put the token in a ticket.
