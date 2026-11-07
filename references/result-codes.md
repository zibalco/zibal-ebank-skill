# Result and status codes

Every table below is complete. The Persian text is the original wording of the docs. English is a translation.

Do not parse `message`. Branch on the numeric codes only.

## Which indicator each endpoint uses

Not every endpoint returns `result`. Check the right field for each one.

| Endpoint | Success indicator on HTTP 200 |
| :--- | :--- |
| checkout create | `result` = 1 means the request was **accepted**. Money outcome comes from `data.checkouts[].status`. |
| checkout inquire | `result` = 1, then `data.checkouts[].status` |
| checkout list, statement list, identified-payment list, identified-payment inquire | `result` = 1 |
| checkout cancel | `result` = 1, then read `data.cancelled_checkouts` and `data.uncancellable_checkouts` per ID |
| balance | No `result` field documented. `balance` is present. |
| account list | No `result` field documented. `data` is present. |
| identified-payment change-status | No `result` field documented. `success` = true. |

HTTP codes, from the docs: 200 for all successful requests, 400 for invalid requests, 500 for internal Zibal errors. The docs also define a 403 "دسترسی غیرمجاز" (unauthorized access) response with the same `{result, message}` body. No endpoint lists which `result` values go with which HTTP code.

## `result` table

The group column is the error mapping the examples use. It is this skill's grouping, not a docs concept.

| result | Persian (original) | English | Group |
| :--- | :--- | :--- | :--- |
| 1 | موفق | Success | `success` |
| 2 | API Key به درستی ارسال نشده است. | API Key not sent correctly | `auth` |
| 3 | API Key صحیح نیست. | API Key incorrect | `auth` |
| 4 | اجازه دسترسی به این سرویس صادر نشده‌است | Access to this service not granted | `auth` |
| 5 | callbackUrl نامعتبر است | callbackUrl invalid | `validation` |
| 6 | مقدار ورودی نامعتبر است | Invalid input value | `validation` |
| 7 | IP ارسال‌کننده درخواست نامعتبر می‌باشد | Sender IP invalid | `auth` |
| 8 | API Key غیرفعال است | API Key disabled | `auth` |
| 9 | حداقل مبلغ باید ۱۰,۰۰۰ ریال باشد | Minimum amount is 10,000 Rial | `validation` |
| 14 | انتقال وجه یافت نشد | Transfer not found | `not_found` |
| 21 | شماره شبای وارد شده معتبر نیست.( 26 کاراکتر و شروع با IR و بدون خط تیره (-) و فاصله) | Invalid IBAN (26 characters, starts with IR, no hyphen or space) | `validation` |
| 29 | موجودی کیف‌پول کارمزد برای این عملیات کافی نیست | Fee wallet balance insufficient for this operation | `funds` |
| 44 | با ورودی های داده شده شبای مورد نظر یافت نشد | No IBAN found for the given inputs | `validation` |
| 45 | سرویس دهنده ها برای استعلام در دسترس نیستند | Providers for the inquiry are unavailable | `provider` |
| 48 | مبلغ از سقف هر تراکنش حساب بیشتر است و با split=false امکان تقسیم تسویه وجود ندارد | Amount exceeds the per-transaction cap of the account and split=false prevents dividing it | `validation` |

Notes:

- Results 2, 3 and 8 say "API Key". The docs' only authentication method is the `ACCESS TOKEN` bearer header, so read these as token problems. The docs do not state this link outright.
- Only result 48 is documented as having no effect ("no transfer is registered and no fee is deducted"). The docs class the other rejection codes as invalid requests, and the examples treat them as `failed`.
- The docs define no result code for a duplicate `uniqueCode`. They say only that the second request is not accepted. Never infer "duplicate" from a code.
- Results 44 and 45 refer to an inquiry. The docs do not say which ebank endpoints can return them.
- A `result` outside this table is an unknown outcome. Never treat it as failure on a money-moving call.

## Transfer `status` table (`data.checkouts[].status`)

| status | Persian (original) | English | Meaning for the app |
| :--- | :--- | :--- | :--- |
| -1 | ایجاد شده (Created): درخواست ایجاد شده اما هنوز پردازش نشده است. | Created. Not yet processed. | Not final |
| 0 | در انتظار پردازش (Pending): درخواست در صف پردازش است. | Pending. Queued for processing. | Not final |
| 3 | موفق (Success): انتقال وجه با موفقیت انجام شده است. | Success. Transfer completed. | Success, but **not final**: it can later become 4 |
| 4 | برگشت به حساب (Returned): مبلغ انتقال وجه به حساب مبدا بازگشته است. | Returned. Amount went back to the source account. | Final: money came back |
| 5 | ناموفق (Failed):  انتقال وجه لغو شد. | Failed. Transfer cancelled. | Final: failed |
| 20 | در انتظار مغایرت گیری (Locked): درخواست در حالت بررسی قرار دارد. | Locked. Under reconciliation review. | Not final |

The docs attach "final" to none of these. The right-hand column is inferred from the documented webhook transitions (0 or 20 to 3, 0 or 20 to 5, 3 to 4).

## `failReason` table

| code | Persian (original) | English |
| :--- | :--- | :--- |
| -1 | ناشناخته | Unknown |
| 1 | موجودی حساب کافی نیست | Account balance insufficient |
| 2 | محدودیت سقف برداشت از حساب | Account withdrawal cap reached |
| 3 | مشکلات قانونی حساب مبدا | Legal problems on the source account |
| 4 | واریز به حساب مقصد ممکن نیست | Deposit to the destination account not possible |
| 5 | لغو شده | Cancelled |

## `reasonCode` table

Send as `reasonCode` on checkout create.

| code | Persian (original) | English |
| :--- | :--- | :--- |
| 1 | تودیع وثیقه | Collateral deposit |
| 2 | عوارض گمرکی | Customs duties |
| 3 | کمک‌های نقدی و خیریه | Cash aid and charity |
| 4 | تادیه دیون | Debt settlement |
| 5 | امور ارزی | Foreign exchange affairs |
| 6 | سایر خدمات دولتی | Other government services |
| 7 | هزینه‌ عمومی و امور روزمره | General and daily expenses |
| 8 | امور درمانی | Medical affairs |
| 9 | معاملات اموال غیرمنقول | Real estate transactions |
| 10 | امور بیمه خدمات | Insurance services |
| 11 | امور سرمایه گذاری و بورس | Investment and stock exchange |
| 12 | مدیریت نقدینگی (فقط برای حساب های متعلق به خود شخص باید استفاده شود) | Liquidity management (use only for accounts owned by the same person) |
| 13 | بازپرداخت تسهیلات | Loan repayment |
| 14 | معاملات اموال منقول | Movable property transactions |
| 15 | واریز حقوق | Salary payment |
| 16 | امور بازنشستگی | Retirement affairs |
| 17 | انتقال وجه مالیاتی | Tax transfer |
| 18 | خرید کالا | Purchase of goods |
| 19 | خرید خدمات | Purchase of services |

## `type` table (statement records)

| code | Persian (original) | English |
| :--- | :--- | :--- |
| 1 | انتقال وجه پایا | Paya transfer |
| 2 | کارمزد پایا | Paya fee |
| 3 | انتقال وجه ساتنا | Satna transfer |
| 4 | کارمزد ساتنا | Satna fee |
| 5 | برگشت انتقال وجه ساتنا | Satna transfer return |
| 6 | انتقال وجه داخلی | Internal transfer |
| 7 | سودبانکی | Bank interest |
| 8 | برگشت انتقال وجه پایا | Paya transfer return |
| 9 | واریزی درگاه پرداختیاری | Payment facilitator gateway deposit |
| 10 | واریزی پایا | Paya deposit |
| 11 | استرداد | Refund |
| 12 | کارمزد استرداد | Refund fee |
| 13 | رفع مغایرت شاپرک(مربوط به درگاه پرداخت مستقیم و پایانه‌های فروش(دستگاه کارتخوان)) | Shaparak discrepancy fix (direct gateway and POS) |
| 14 | ناشناخته | Unknown |
| 15 | عملیات بانکی که معمولا توسط شعبه انجام می‌گیرد. | Bank operation, usually done by a branch |
| 16 | انتقال وجه شاپرکی(مربوط به درگاه پرداخت مستقیم و پایانه‌های فروش(دستگاه کارتخوان)) | Shaparak transfer (direct gateway and POS) |
| 17 | کارمزد پایاهای برگشتی | Returned Paya fee |
| 18 | انتقال وجه پل | Pol transfer |
| 19 | برگشتی واریزی پل | Pol deposit return |
| 20 | کارمزد شاپرک(دستگاه کارتخوان)) | Shaparak fee (POS) |
| 21 | واریز از طریق حساب به حساب | Account-to-account deposit |
| 22 | واریز ساتنا به حساب | Satna deposit to account |
| 23 | مسدودی حساب | Account block |
| 24 | خرید از طریق پایانه های فروش(دستگاه کارتخوان) | POS purchase |
| 25 | واریز از طرق چک به حساب | Cheque deposit to account |
| 26 | واریز پل به حساب | Pol deposit to account |
| 27 | واریز کارت به حساب | Card deposit to account |
| 28 | اصلاحیه تراکنش کارت به کارت | Card-to-card transaction correction |
| 29 | اصلاحیه واریز کارت به کارت | Card-to-card deposit correction |

## Smaller enums

### `destinationType` (checkout responses)

| value | Persian (original) | English |
| :--- | :--- | :--- |
| 1 | شبا | IBAN |
| 2 | کارت | Card |
| 3 | حساب | Account |

### `transactionType` (checkout details)

| value | Persian (original) | English |
| :--- | :--- | :--- |
| `paya` | پایا | Paya |
| `internal` | تراکنش‌های داخلی بانک | Internal bank transaction |
| `satna` | ساتنا | Satna |
| `pol` | انتقال پول پل | Pol transfer |

### Account `status` (account list)

| value | Persian (original) | English |
| :--- | :--- | :--- |
| 1 | ساخته شده | Created |
| 2 | در انتظار فعال‌سازی | Awaiting activation |
| 3 | رد شده | Rejected |
| 4 | فعال | Active |

### Statement `side`

| value | Persian (original) | English |
| :--- | :--- | :--- |
| 1 | واریز | Deposit |
| 0 | برداشت | Withdrawal |

### Deposit `verifyStatus` (deposit responses)

| value | Persian (original) | English |
| :--- | :--- | :--- |
| 0 | در انتظار تایید | Awaiting approval |
| 2 | تایید شده | Approved |
| 3 | رد شده | Rejected |

### Deposit `status` filter (identified-payment list query)

| value | Persian (original) | English |
| :--- | :--- | :--- |
| 0 | در حال انتظار | Pending |
| 2 | تایید شده | Approved |
| 3 | رد شده | Rejected |
| 4 | رد شده - بدون برگشت | Rejected, no return |
| 5 | برگشت خورده | Returned |

The filter has values 4 and 5, but the `verifyStatus` field documents only 0, 2 and 3. If `verifyStatus` is any other value, flag the record for review.

### Deposit decision `status` (change-status body)

| value | Persian (original) | English |
| :--- | :--- | :--- |
| 2 | تایید | Approve |
| 3 | رد | Reject |
