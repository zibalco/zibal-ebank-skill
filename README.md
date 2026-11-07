# zibal-ebank

[![skills.sh](https://img.shields.io/badge/skills.sh-zibal--ebank-blue)](https://skills.sh/zibalco/zibal-ebank-skill)

An agent skill that helps coding agents such as Cursor and Codex integrate Zibal's ebank API (سرویس بانکداری شرکتی زیبال) correctly in common server stacks.

[فارسی](#فارسی)

## What it does

- Implements fund transfers (انتقال وجه): create, inquire, list and cancel.
- Reads balance, statements and the list of connected accounts.
- Lists, inquires, approves and rejects identified deposits (واریزی‌ها).
- Handles the transfer webhook as a trigger only.
- Maps every documented result and status code, including the original Persian text.
- Enforces safe handling of money: no blind retries after a timeout, duplicate protection, strict input validation, masked logs and server-only credentials.
- Ships complete examples for Express, Next.js App Router, Laravel, Django and FastAPI.

It covers ebank only, not the Zibal payment gateway (IPG).

## Install

```bash
npx skills add zibalco/zibal-ebank-skill
```

## Example prompts

- "Add Zibal ebank payouts to this Express app. Users withdraw their wallet balance to their IBAN."
- "Build a Laravel job that pays sellers through Zibal ebank and reconciles transfers that timed out."
- "Show the Zibal ebank account balance on our Django admin dashboard."
- "Add an endpoint in our FastAPI service to approve or reject incoming Zibal deposits."
- "Handle Zibal's transfer webhook in our Next.js app."

## Layout

| Path | Contents |
| :--- | :--- |
| [SKILL.md](SKILL.md) | Workflow and rules the agent follows |
| [references/api.md](references/api.md) | Every ebank endpoint, parameter and field |
| [references/auth.md](references/auth.md) | Token, headers, IP allowlist, env vars |
| [references/result-codes.md](references/result-codes.md) | All code tables, Persian and English |
| [references/webhooks.md](references/webhooks.md) | Transfer webhook |
| [references/security-checklist.md](references/security-checklist.md) | Checklist tagged `[docs]` or `[practice]` |
| [examples/](examples/) | Runnable examples for five frameworks |

There is no sandbox in Zibal's ebank docs. Test with a local mock. Never test money-moving calls with a real token.

The content was extracted by hand from Zibal's ebank documentation. It does not update itself. Check it against the current docs before relying on it.

## License

[MIT](LICENSE)

---

<div dir="rtl">

## فارسی

یک «اسکیل» برای ایجنت‌های برنامه‌نویسی مثل Cursor و Codex، تا سرویس بانکداری شرکتی زیبال (ebank) را به‌درستی در پروژه پیاده‌سازی کنند.

### چه کاری می‌کند

- انتقال وجه: ثبت، استعلام، لیست و لغو
- دریافت موجودی، صورت حساب و لیست حساب‌های متصل
- لیست، استعلام، تایید و رد واریزی‌ها
- مدیریت وب‌هوک انتقال وجه، فقط به‌عنوان سیگنال
- نگاشت همه‌ی کدهای result و status مستند، همراه با متن اصلی فارسی
- رعایت اصول امن کار با پول: بدون تکرار کورکورانه بعد از timeout، جلوگیری از اجرای تکراری، اعتبارسنجی سخت‌گیرانه‌ی ورودی، ماسک کردن لاگ‌ها و نگه داشتن کلید فقط در سرور
- نمونه‌ی کامل برای Express، Next.js، Laravel، Django و FastAPI

این اسکیل فقط بانکداری شرکتی را پوشش می‌دهد و درگاه پرداخت (IPG) را شامل نمی‌شود.

### نصب

</div>

```bash
npx skills add zibalco/zibal-ebank-skill
```

<div dir="rtl">

### نمونه درخواست‌ها

- «تسویه با بانکداری شرکتی زیبال را به این پروژه‌ی Express اضافه کن. کاربرها موجودی کیف پول را به شبای خودشان برداشت می‌کنند.»
- «در Laravel یک job بنویس که به فروشنده‌ها از طریق زیبال پول واریز کند و انتقال‌های timeout‌شده را پیگیری کند.»
- «موجودی حساب بانکداری شرکتی زیبال را در پنل ادمین Django نشان بده.»
- «در FastAPI یک endpoint برای تایید یا رد واریزی‌های زیبال اضافه کن.»
- «وب‌هوک انتقال وجه زیبال را در Next.js مدیریت کن.»

### نکته

مستندات بانکداری شرکتی زیبال محیط sandbox تعریف نکرده است. برای تست از mock محلی استفاده کنید و هرگز عملیات جابه‌جایی پول را با توکن واقعی تست نکنید.

محتوای این اسکیل به‌صورت دستی از مستندات زیبال استخراج شده و خودکار به‌روز نمی‌شود. پیش از استفاده آن را با مستندات فعلی مقایسه کنید.

</div>
