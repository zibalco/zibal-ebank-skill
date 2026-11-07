# Django

Tested with Python 3.14, Django 6.1.3, requests 2.34.2 and gunicorn 26.2.0, served by gunicorn with two worker processes so that the duplicate checks ran across processes.

SQLite keeps the example self-contained. For production, use your own database and keep the unique constraints and the conditional updates.

Routes are the same as in [node-express.md](node-express.md). Every route except the webhook needs the `X-Internal-Key` header. It is a placeholder for your own authentication and authorization. Replace it before production.

Call `POST /internal/reconcile` from a scheduler, or call `payouts.reconcile_due()` and `deposits.reconcile_due()` from a management command. The webhook is sent once, so it cannot be the only signal.

## Files

### `requirements.txt`

```text
Django==6.1.3
gunicorn==26.2.0
requests==2.34.2
```

### `.env.example`

```bash
DJANGO_SECRET_KEY=
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
ZIBAL_ACCESS_TOKEN=
ZIBAL_ACCOUNT_ID=
# ZIBAL_BASE_URL=https://api.zibal.ir
# ZIBAL_CALLBACK_URL=https://your-domain.example/webhooks/zibal/checkout
# ZIBAL_TIMEOUT_MS=15000
# ZIBAL_REVIEW_AFTER_MS=900000
# ZIBAL_RETURN_WATCH_DAYS=7
INTERNAL_API_KEY=
DB_PATH=./ebank.sqlite
```

### `manage.py`

```python
#!/usr/bin/env python
import os
import sys

if __name__ == "__main__":
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
    from django.core.management import execute_from_command_line

    execute_from_command_line(sys.argv)
```

### `config/__init__.py`

```python
```

### `config/settings.py`

```python
import os


def _required(name: str) -> str:
    value = os.environ.get(name)
    if not value:
        raise RuntimeError(f"Missing environment variable {name}")
    return value


SECRET_KEY = _required("DJANGO_SECRET_KEY")
DEBUG = False
ALLOWED_HOSTS = os.environ.get("DJANGO_ALLOWED_HOSTS", "localhost,127.0.0.1").split(",")

INSTALLED_APPS = ["ebank"]
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
]
ROOT_URLCONF = "config.urls"
WSGI_APPLICATION = "config.wsgi.application"
USE_TZ = True

# Routes have no trailing slash, so a POST is never redirected to a slashed URL.
APPEND_SLASH = False
# Request bodies above this size are refused with 400.
DATA_UPLOAD_MAX_MEMORY_SIZE = 65536

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": os.environ.get("DB_PATH") or "./ebank.sqlite",
        "OPTIONS": {"timeout": 5, "transaction_mode": "IMMEDIATE", "init_command": "PRAGMA journal_mode=WAL;"},
    }
}
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"

LOGGING = {
    "version": 1,
    "handlers": {"console": {"class": "logging.StreamHandler"}},
    "loggers": {"zibal": {"handlers": ["console"], "level": "INFO"}},
}

ZIBAL = {
    "TOKEN": _required("ZIBAL_ACCESS_TOKEN"),
    "ACCOUNT_ID": _required("ZIBAL_ACCOUNT_ID"),
    "INTERNAL_KEY": _required("INTERNAL_API_KEY"),
    "BASE_URL": os.environ.get("ZIBAL_BASE_URL") or "https://api.zibal.ir",
    "CALLBACK_URL": os.environ.get("ZIBAL_CALLBACK_URL") or None,
    "TIMEOUT_S": int(os.environ.get("ZIBAL_TIMEOUT_MS") or 15000) / 1000,
    "REVIEW_AFTER_S": int(os.environ.get("ZIBAL_REVIEW_AFTER_MS") or 900000) / 1000,
    "RETURN_WATCH_S": int(os.environ.get("ZIBAL_RETURN_WATCH_DAYS") or 7) * 86400,
}
```

### `config/urls.py`

```python
from django.urls import path

from ebank import views

urlpatterns = [
    path("payouts", views.create_payout),
    path("payouts/<str:payout_id>", views.read_payout),
    path("payouts/<str:payout_id>/reconcile", views.reconcile_payout),
    path("payouts/<str:payout_id>/cancel", views.cancel_payout),
    path("internal/reconcile", views.reconcile_all),
    path("deposits/<str:transaction_id>/decision", views.decide_deposit),
    path("deposits/<str:transaction_id>/reconcile", views.reconcile_deposit),
    path("balance", views.read_balance),
    path("webhooks/zibal/checkout", views.webhook),
]
```

### `config/wsgi.py`

```python
import os

from django.core.wsgi import get_wsgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
application = get_wsgi_application()
```

### `ebank/__init__.py`

```python
```

### `ebank/apps.py`

```python
from django.apps import AppConfig


class EbankConfig(AppConfig):
    name = "ebank"
```

### `ebank/models.py`

```python
import json

from django.db import models


class Payout(models.Model):
    id = models.CharField(primary_key=True, max_length=36)
    request_key = models.CharField(max_length=64, unique=True)
    request_hash = models.CharField(max_length=64)
    unique_code = models.CharField(max_length=36, unique=True)
    account_id = models.CharField(max_length=64)
    amount = models.BigIntegerField()
    iban = models.CharField(max_length=26)
    reason_code = models.IntegerField()
    delay = models.IntegerField(null=True)
    description = models.TextField(null=True)
    payment_number = models.CharField(max_length=64, null=True)
    split = models.BooleanField(null=True)
    state = models.CharField(max_length=16)
    tracker_id = models.CharField(max_length=64, null=True)
    checkouts = models.TextField(default="[]")
    last_result = models.IntegerField(null=True)
    last_message = models.TextField(null=True)
    error_group = models.CharField(max_length=16, null=True)
    cancel_state = models.CharField(max_length=16, null=True)
    submitted_at = models.FloatField(null=True)
    created_at = models.FloatField()
    updated_at = models.FloatField()

    def checkout_list(self) -> list:
        return json.loads(self.checkouts)


class DepositDecision(models.Model):
    id = models.CharField(primary_key=True, max_length=36)
    account_id = models.CharField(max_length=64)
    transaction_id = models.CharField(max_length=64)
    status = models.IntegerField()
    state = models.CharField(max_length=16)
    refund_tracker_id = models.CharField(max_length=64, null=True)
    last_result = models.IntegerField(null=True)
    last_message = models.TextField(null=True)
    submitted_at = models.FloatField(null=True)
    created_at = models.FloatField()
    updated_at = models.FloatField()

    class Meta:
        constraints = [models.UniqueConstraint(fields=["account_id", "transaction_id"], name="one_decision_per_deposit")]
```

### `ebank/zibal.py`

```python
from dataclasses import dataclass
from typing import Any

import requests
from django.conf import settings

# Grouping of the documented result table. See references/result-codes.md.
RESULT_GROUPS = {
    2: "auth", 3: "auth", 4: "auth", 7: "auth", 8: "auth",
    5: "validation", 6: "validation", 9: "validation", 21: "validation",
    44: "validation", 48: "validation",
    29: "funds",
    14: "not_found",
    45: "provider",
}
REJECTION_GROUPS = {"auth", "validation", "funds"}
CHECKOUT_STATUSES = {-1, 0, 3, 4, 5, 20}
PENDING_STATUSES = {-1, 0, 20}

ZIBAL = settings.ZIBAL
session = requests.Session()
session.headers["Authorization"] = f"Bearer {ZIBAL['TOKEN']}"


@dataclass
class Reply:
    http: int | None
    json: Any

    @property
    def result(self) -> int | None:
        value = self.json.get("result") if isinstance(self.json, dict) else None
        return value if isinstance(value, int) and not isinstance(value, bool) else None

    @property
    def message(self) -> str | None:
        value = self.json.get("message") if isinstance(self.json, dict) else None
        return value if isinstance(value, str) else None


def result_group(result: int | None) -> str:
    return RESULT_GROUPS.get(result, "unknown")


def is_rejected(reply: Reply) -> bool:
    """True only when Zibal clearly refused the request. Anything else is an unknown outcome."""
    if reply.http is None or reply.http < 200 or reply.http >= 500:
        return False
    return result_group(reply.result) in REJECTION_GROUPS


def _call(method: str, path: str, **kwargs) -> Reply:
    # requests has no default timeout, and it follows redirects unless told not to.
    try:
        response = session.request(
            method, ZIBAL["BASE_URL"] + path, timeout=ZIBAL["TIMEOUT_S"], allow_redirects=False, **kwargs
        )
    except requests.RequestException:
        return Reply(None, None)
    try:
        return Reply(response.status_code, response.json())
    except ValueError:
        return Reply(response.status_code, None)


def create_checkout(body: dict) -> Reply:
    return _call("POST", "/ebank/v1/account/checkout/create/", json=body)


def inquire_checkout(unique_code: str) -> Reply:
    return _call("GET", "/ebank/v1/account/checkout/inquire/", params={"uniqueCode": unique_code})


def cancel_checkout(account_id: str, checkout_ids: list[str]) -> Reply:
    return _call("POST", "/ebank/v1/account/checkout/cancel/", json={"accountId": account_id, "checkoutIds": checkout_ids})


def balance(account_id: str) -> Reply:
    return _call("GET", "/ebank/v1/account/balance/", params={"accountId": account_id})


def inquire_deposit(account_id: str, transaction_id: str) -> Reply:
    return _call("GET", "/ebank/v1/account/identified-payment/inquire/", params={"accountId": account_id, "transactionId": transaction_id})


def change_deposit_status(account_id: str, transaction_id: str, status: int) -> Reply:
    return _call("POST", "/ebank/v1/account/identified-payment/change-status/", json={"accountId": account_id, "transactionId": transaction_id, "status": status})


def checkout_outcome(unique_code: str, amount: int, data: Any) -> str:
    """Maps a transfer object to a local state. Every checkout counts, because split creates several."""
    if not isinstance(data, dict):
        return "needs_review"
    if data.get("uniqueCode") != unique_code or data.get("totalAmount") != amount:
        return "needs_review"
    checkouts = data.get("checkouts")
    if not isinstance(checkouts, list) or not checkouts:
        return "needs_review"
    if not all(isinstance(item, dict) for item in checkouts):
        return "needs_review"
    statuses = [item.get("status") for item in checkouts]
    if not all(isinstance(status, int) and status in CHECKOUT_STATUSES for status in statuses):
        return "needs_review"
    if any(status in PENDING_STATUSES for status in statuses):
        return "processing"
    if all(status == 3 for status in statuses):
        return "succeeded" if all(item.get("refCode") for item in checkouts) else "needs_review"
    if all(status == 5 for status in statuses):
        return "failed"
    if all(status == 4 for status in statuses):
        return "returned"
    return "needs_review"


def mask_iban(iban: str) -> str:
    return f"{iban[:4]}{'*' * 18}{iban[-4:]}"
```

### `ebank/payouts.py`

```python
import hashlib
import json
import logging
import time
import uuid

from django.db import IntegrityError, transaction
from django.db.models import Q

from . import zibal
from .models import Payout

log = logging.getLogger("zibal")
ZIBAL = zibal.ZIBAL

# A state can only be entered from these states, so a stale read cannot undo a newer one.
ALLOWED_FROM = {
    "unknown": ["submitting"],
    "processing": ["submitting", "unknown", "processing", "needs_review"],
    "succeeded": ["submitting", "unknown", "processing", "succeeded", "needs_review"],
    "returned": ["submitting", "unknown", "processing", "succeeded", "needs_review"],
    "failed": ["submitting", "unknown", "processing", "needs_review"],
    "needs_review": ["submitting", "unknown", "processing", "succeeded", "needs_review"],
}
RECONCILABLE = {"submitting", "unknown", "processing", "succeeded", "needs_review"}


class Conflict(Exception):
    pass


def get_payout(payout_id: str) -> Payout | None:
    return Payout.objects.filter(pk=payout_id).first()


def view(payout: Payout, **extra) -> dict:
    return {
        "id": payout.id,
        "requestKey": payout.request_key,
        "uniqueCode": payout.unique_code,
        "state": payout.state,
        "amount": payout.amount,
        "iban": zibal.mask_iban(payout.iban),
        "trackerId": payout.tracker_id,
        "lastResult": payout.last_result,
        "errorGroup": payout.error_group,
        "checkouts": payout.checkout_list(),
        **extra,
    }


def _transition(payout_id: str, state: str, **fields) -> None:
    Payout.objects.filter(pk=payout_id, state__in=ALLOWED_FROM[state]).update(state=state, updated_at=time.time(), **fields)


def _checkout_summary(data) -> str:
    checkouts = data.get("checkouts") if isinstance(data, dict) else None
    if not isinstance(checkouts, list):
        return "[]"
    return json.dumps([
        {key: item.get(key) for key in ("id", "amount", "status", "refCode", "failReason")}
        for item in checkouts if isinstance(item, dict)
    ])


def _submit(payout_id: str) -> None:
    now = time.time()
    claimed = Payout.objects.filter(pk=payout_id, state="created").update(state="submitting", submitted_at=now, updated_at=now)
    if claimed == 0:
        return

    payout = get_payout(payout_id)
    body = {
        "amount": payout.amount,
        "iban": payout.iban,
        "accountId": payout.account_id,
        "reasonCode": payout.reason_code,
        "uniqueCode": payout.unique_code,
    }
    if payout.delay is not None:
        body["delay"] = payout.delay
    if payout.description is not None:
        body["description"] = payout.description
    if payout.payment_number is not None:
        body["paymentNumber"] = payout.payment_number
    if payout.split is not None:
        body["split"] = payout.split
    if ZIBAL["CALLBACK_URL"]:
        body["callbackUrl"] = ZIBAL["CALLBACK_URL"]

    reply = zibal.create_checkout(body)
    base = {"last_result": reply.result, "last_message": reply.message}
    data = reply.json.get("data") if isinstance(reply.json, dict) else None

    if reply.http == 200 and reply.result == 1 and isinstance(data, dict):
        tracker = data.get("trackerId")
        _transition(
            payout_id,
            zibal.checkout_outcome(payout.unique_code, payout.amount, data),
            tracker_id=tracker if isinstance(tracker, str) else None,
            checkouts=_checkout_summary(data),
            **base,
        )
    elif zibal.is_rejected(reply):
        _transition(payout_id, "failed", error_group=zibal.result_group(reply.result), **base)
    else:
        _transition(payout_id, "unknown", error_group=zibal.result_group(reply.result), **base)
    log.info("payout_submitted id=%s iban=%s http=%s result=%s", payout_id, zibal.mask_iban(payout.iban), reply.http, reply.result)


def create_payout(values: dict) -> tuple[Payout, bool]:
    fingerprint = json.dumps([
        values["amount"], values["iban"], values["reasonCode"], values.get("delay"),
        values.get("description"), values.get("paymentNumber"), values.get("split"),
    ], ensure_ascii=False)
    request_hash = hashlib.sha256(fingerprint.encode()).hexdigest()
    now = time.time()

    try:
        with transaction.atomic():
            payout = Payout.objects.create(
                id=str(uuid.uuid4()),
                request_key=values["requestKey"],
                request_hash=request_hash,
                unique_code=str(uuid.uuid4()),
                account_id=ZIBAL["ACCOUNT_ID"],
                amount=values["amount"],
                iban=values["iban"],
                reason_code=values["reasonCode"],
                delay=values.get("delay"),
                description=values.get("description"),
                payment_number=values.get("paymentNumber"),
                split=values.get("split"),
                state="created",
                created_at=now,
                updated_at=now,
            )
    except IntegrityError:
        existing = Payout.objects.get(request_key=values["requestKey"])
        if existing.request_hash != request_hash:
            raise Conflict("request_key_reused")
        # 'created' means nothing was sent yet, so the claim in _submit() is still safe.
        if existing.state == "created":
            _submit(existing.id)
        return get_payout(existing.id), True

    _submit(payout.id)
    return get_payout(payout.id), False


def reconcile_payout(payout_id: str) -> Payout | None:
    payout = get_payout(payout_id)
    if payout is None or payout.state not in RECONCILABLE:
        return payout

    now = time.time()
    if payout.state == "submitting":
        # A live request is still inside its timeout. Do not race it.
        if now - payout.updated_at < ZIBAL["TIMEOUT_S"] * 2:
            return payout
        _transition(payout_id, "unknown")

    reply = zibal.inquire_checkout(payout.unique_code)
    base = {"last_result": reply.result, "last_message": reply.message}

    if reply.http == 200 and reply.result == 1:
        data = reply.json.get("data")
        tracker = data.get("trackerId") if isinstance(data, dict) else None
        _transition(
            payout_id,
            zibal.checkout_outcome(payout.unique_code, payout.amount, data),
            tracker_id=tracker if isinstance(tracker, str) else payout.tracker_id,
            checkouts=_checkout_summary(data),
            error_group=None,
            **base,
        )
    else:
        still_open = payout.state in ("unknown", "submitting")
        expired = now - (payout.submitted_at or payout.created_at) >= ZIBAL["REVIEW_AFTER_S"]
        # Not found for a transfer Zibal already accepted is never normal.
        vanished = reply.result == 14 and not still_open and payout.state != "needs_review"
        if (still_open and expired) or vanished:
            _transition(payout_id, "needs_review", **base)
    log.info("payout_reconciled id=%s http=%s result=%s", payout_id, reply.http, reply.result)
    return get_payout(payout_id)


def reconcile_due() -> int:
    now = time.time()
    ids = list(
        Payout.objects.filter(
            Q(state__in=["unknown", "processing"])
            | Q(state="submitting", updated_at__lte=now - ZIBAL["TIMEOUT_S"] * 2)
            | Q(state="succeeded", submitted_at__gte=now - ZIBAL["RETURN_WATCH_S"])
        ).order_by("updated_at").values_list("id", flat=True)[:50]
    )
    for payout_id in ids:
        reconcile_payout(payout_id)
    return len(ids)


def cancel_payout(payout: Payout) -> dict:
    ids = [
        item["id"] for item in payout.checkout_list()
        if item.get("id") and item.get("status") in zibal.PENDING_STATUSES
    ]
    if not ids:
        raise Conflict("not_cancellable")

    claimed = Payout.objects.filter(pk=payout.id, cancel_state__isnull=True, state="processing").update(
        cancel_state="sent", updated_at=time.time()
    )
    if claimed == 0:
        raise Conflict("not_cancellable")

    reply = zibal.cancel_checkout(payout.account_id, ids)

    if reply.http == 200 and reply.result == 1:
        data = reply.json.get("data")
        data = data if isinstance(data, dict) else {}
        cancelled = data.get("cancelled_checkouts")
        uncancellable = data.get("uncancellable_checkouts")
        return {
            "http": 200,
            "payout": reconcile_payout(payout.id),
            "cancelOutcome": "answered",
            "cancelled": cancelled if isinstance(cancelled, list) else [],
            "uncancellable": uncancellable if isinstance(uncancellable, list) else [],
        }
    if zibal.is_rejected(reply):
        Payout.objects.filter(pk=payout.id).update(cancel_state=None, last_result=reply.result, updated_at=time.time())
        return {"http": 422, "payout": get_payout(payout.id), "cancelOutcome": "rejected", "cancelled": [], "uncancellable": []}
    # Unknown: keep cancel_state so it cannot be sent twice. Status comes from inquire.
    return {"http": 202, "payout": reconcile_payout(payout.id), "cancelOutcome": "unknown", "cancelled": [], "uncancellable": []}
```

### `ebank/deposits.py`

```python
import logging
import time
import uuid

from django.db import IntegrityError, transaction
from django.db.models import Q

from . import zibal
from .models import DepositDecision
from .payouts import Conflict

log = logging.getLogger("zibal")
ZIBAL = zibal.ZIBAL


def _find(transaction_id: str) -> DepositDecision | None:
    return DepositDecision.objects.filter(account_id=ZIBAL["ACCOUNT_ID"], transaction_id=transaction_id).first()


def view(decision: DepositDecision, **extra) -> dict:
    return {
        "id": decision.id,
        "transactionId": decision.transaction_id,
        "status": decision.status,
        "state": decision.state,
        "refundTrackerId": decision.refund_tracker_id,
        "lastResult": decision.last_result,
        **extra,
    }


def _set_state(decision_id: str, allowed_from: list[str], state: str, **fields) -> None:
    DepositDecision.objects.filter(pk=decision_id, state__in=allowed_from).update(state=state, updated_at=time.time(), **fields)


def _fetch_deposit(transaction_id: str) -> dict | None:
    """Returns the deposit, or None when the answer does not settle anything."""
    reply = zibal.inquire_deposit(ZIBAL["ACCOUNT_ID"], transaction_id)
    data = reply.json.get("data") if isinstance(reply.json, dict) else None
    if reply.http != 200 or reply.result != 1 or not isinstance(data, list):
        return None
    return next((item for item in data if isinstance(item, dict) and item.get("id") == transaction_id), None)


def decide_deposit(transaction_id: str, status: int) -> tuple[int, DepositDecision, bool]:
    now = time.time()
    duplicate = False
    try:
        with transaction.atomic():
            DepositDecision.objects.create(
                id=str(uuid.uuid4()), account_id=ZIBAL["ACCOUNT_ID"], transaction_id=transaction_id,
                status=status, state="created", created_at=now, updated_at=now,
            )
    except IntegrityError:
        duplicate = True

    decision = _find(transaction_id)
    if decision.status != status:
        raise Conflict("different_decision_exists")

    claimed = DepositDecision.objects.filter(pk=decision.id, state="created").update(state="submitting", updated_at=time.time())
    if claimed == 0:
        return 200, _find(transaction_id), True

    deposit = _fetch_deposit(transaction_id)
    if deposit is None:
        # Nothing was sent, so release the claim and let the caller try again.
        _set_state(decision.id, ["submitting"], "created")
        return 503, _find(transaction_id), duplicate

    current = deposit.get("verifyStatus")
    if current != 0:
        if current == status:
            _set_state(decision.id, ["submitting"], "done")
            return 200, _find(transaction_id), duplicate
        if current in (2, 3):
            _set_state(decision.id, ["submitting"], "failed")
            return 409, _find(transaction_id), duplicate
        _set_state(decision.id, ["submitting"], "needs_review")
        return 202, _find(transaction_id), duplicate

    DepositDecision.objects.filter(pk=decision.id).update(submitted_at=time.time())
    reply = zibal.change_deposit_status(ZIBAL["ACCOUNT_ID"], transaction_id, status)
    base = {"last_result": reply.result, "last_message": reply.message}
    payload = reply.json if isinstance(reply.json, dict) else {}

    if reply.http == 200 and payload.get("success") is True:
        data = payload.get("data")
        tracker = data.get("refundRequestTrackerId") if isinstance(data, dict) else None
        _set_state(decision.id, ["submitting"], "done", refund_tracker_id=tracker if isinstance(tracker, str) else None, **base)
        http = 200
    elif zibal.is_rejected(reply):
        _set_state(decision.id, ["submitting"], "failed", **base)
        http = 422
    else:
        _set_state(decision.id, ["submitting"], "unknown", **base)
        http = 202
    log.info("deposit_decided id=%s http=%s result=%s", decision.id, reply.http, reply.result)
    return http, _find(transaction_id), duplicate


def reconcile_deposit(transaction_id: str) -> DepositDecision | None:
    decision = _find(transaction_id)
    if decision is None:
        return None
    if decision.state == "submitting" and time.time() - decision.updated_at >= ZIBAL["TIMEOUT_S"] * 3:
        # The process died after claiming. The change-status call may or may not have been sent.
        _set_state(decision.id, ["submitting"], "unknown")
    elif decision.state != "unknown":
        return decision

    deposit = _fetch_deposit(transaction_id)
    if deposit is not None and deposit.get("verifyStatus") == decision.status:
        _set_state(decision.id, ["unknown"], "done")
    else:
        age = time.time() - (decision.submitted_at or decision.created_at)
        # A changed status that is not ours, or no change after the window, needs a person.
        changed_elsewhere = deposit is not None and deposit.get("verifyStatus") != 0
        if changed_elsewhere or age >= ZIBAL["REVIEW_AFTER_S"]:
            _set_state(decision.id, ["unknown"], "needs_review")
    return _find(transaction_id)


def reconcile_due() -> int:
    ids = list(
        DepositDecision.objects.filter(
            Q(state="unknown") | Q(state="submitting", updated_at__lte=time.time() - ZIBAL["TIMEOUT_S"] * 3)
        ).order_by("updated_at").values_list("transaction_id", flat=True)[:50]
    )
    for transaction_id in ids:
        reconcile_deposit(transaction_id)
    return len(ids)
```

### `ebank/views.py`

```python
import functools
import hashlib
import hmac
import json
import re

from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_http_methods

from . import deposits, payouts, zibal
from .models import Payout

ZIBAL = zibal.ZIBAL
MAX_BODY_BYTES = 16384

# fullmatch, because `$` in Python also matches before a trailing newline.
# [0-9] rather than \d, because \d matches Persian digits.
REQUEST_KEY = re.compile(r"[A-Za-z0-9_-]{1,64}")
IBAN = re.compile(r"IR[0-9]{24}")
CONTROL = re.compile(r"[\x00-\x1f\x7f]")
PAYOUT_FIELDS = {"requestKey", "amount", "iban", "reasonCode", "delay", "description", "paymentNumber", "split"}
PAYOUT_REQUIRED = {"requestKey", "amount", "iban", "reasonCode"}


def _int_between(value, low: int, high: int) -> bool:
    return isinstance(value, int) and not isinstance(value, bool) and low <= value <= high


def _text(value, max_length: int) -> bool:
    return isinstance(value, str) and len(value) <= max_length and not CONTROL.search(value)


def _parse_payout(body) -> dict | None:
    """amount, iban and delay limits come from the docs. Other caps are this app's own choice."""
    if not isinstance(body, dict) or not set(body) <= PAYOUT_FIELDS or not PAYOUT_REQUIRED <= set(body):
        return None
    checks = [
        isinstance(body["requestKey"], str) and REQUEST_KEY.fullmatch(body["requestKey"]),
        _int_between(body["amount"], 10000, 9007199254740991),
        isinstance(body["iban"], str) and IBAN.fullmatch(body["iban"]),
        _int_between(body["reasonCode"], 1, 19),
        "delay" not in body or _int_between(body["delay"], -1, 10080),
        "description" not in body or _text(body["description"], 255),
        "paymentNumber" not in body or _text(body["paymentNumber"], 64),
        "split" not in body or isinstance(body["split"], bool),
    ]
    return body if all(checks) else None


def _json_body(request, limit: int = MAX_BODY_BYTES):
    if request.content_type != "application/json" or len(request.body) > limit:
        return None
    try:
        return json.loads(request.body)
    except (ValueError, RecursionError):
        return None


def _error(error: str, status: int) -> JsonResponse:
    return JsonResponse({"error": error}, status=status)


def internal(methods: list[str]):
    """Header-based key, not cookies, so CSRF does not apply. Replace with your own auth."""
    def decorator(view):
        @csrf_exempt
        @require_http_methods(methods)
        @functools.wraps(view)
        def wrapped(request, *args, **kwargs):
            given = request.headers.get("X-Internal-Key", "")
            expected = ZIBAL["INTERNAL_KEY"]
            if not hmac.compare_digest(hashlib.sha256(given.encode()).digest(), hashlib.sha256(expected.encode()).digest()):
                return _error("unauthorized", 401)
            try:
                return view(request, *args, **kwargs)
            except payouts.Conflict as conflict:
                return _error(str(conflict), 409)
        return wrapped
    return decorator


def _http_status(state: str) -> int:
    if state == "failed":
        return 422
    if state in ("unknown", "needs_review", "submitting"):
        return 202
    return 201


@internal(["POST"])
def create_payout(request):
    values = _parse_payout(_json_body(request))
    if values is None:
        return _error("invalid_request", 400)
    payout, duplicate = payouts.create_payout(values)
    return JsonResponse(payouts.view(payout, duplicate=duplicate), status=200 if duplicate else _http_status(payout.state))


@internal(["GET"])
def read_payout(request, payout_id: str):
    payout = payouts.get_payout(payout_id)
    return JsonResponse(payouts.view(payout)) if payout else _error("not_found", 404)


@internal(["POST"])
def reconcile_payout(request, payout_id: str):
    payout = payouts.reconcile_payout(payout_id)
    return JsonResponse(payouts.view(payout)) if payout else _error("not_found", 404)


@internal(["POST"])
def cancel_payout(request, payout_id: str):
    payout = payouts.get_payout(payout_id)
    if payout is None:
        return _error("not_found", 404)
    outcome = payouts.cancel_payout(payout)
    body = payouts.view(
        outcome["payout"],
        cancelOutcome=outcome["cancelOutcome"],
        cancelled=outcome["cancelled"],
        uncancellable=outcome["uncancellable"],
    )
    return JsonResponse(body, status=outcome["http"])


@internal(["POST"])
def reconcile_all(request):
    return JsonResponse({"processed": payouts.reconcile_due() + deposits.reconcile_due()})


@internal(["POST"])
def decide_deposit(request, transaction_id: str):
    body = _json_body(request)
    valid_id = REQUEST_KEY.fullmatch(transaction_id)
    if not valid_id or not isinstance(body, dict) or set(body) != {"status"} or not _int_between(body["status"], 2, 3):
        return _error("invalid_request", 400)
    code, decision, duplicate = deposits.decide_deposit(transaction_id, body["status"])
    return JsonResponse(deposits.view(decision, duplicate=duplicate), status=code)


@internal(["POST"])
def reconcile_deposit(request, transaction_id: str):
    decision = deposits.reconcile_deposit(transaction_id)
    return JsonResponse(deposits.view(decision)) if decision else _error("not_found", 404)


@internal(["GET"])
def read_balance(request):
    reply = zibal.balance(ZIBAL["ACCOUNT_ID"])
    if reply.http is None:
        return _error("zibal_unreachable", 504)
    # Balance has no result field on success.
    value = reply.json.get("balance") if isinstance(reply.json, dict) else None
    if reply.http == 200 and isinstance(value, int) and not isinstance(value, bool):
        return JsonResponse({"balance": value, "balanceUpdatedAt": reply.json.get("balanceUpdatedAt")})
    return JsonResponse(
        {"error": "zibal_error", "lastResult": reply.result, "errorGroup": zibal.result_group(reply.result)},
        status=502,
    )


# Public on purpose, so it is exempt from CSRF. Only the unique code is read, and the state comes from inquire.
@csrf_exempt
@require_http_methods(["POST"])
def webhook(request):
    body = _json_body(request, limit=65536)
    data = body.get("data") if isinstance(body, dict) else None
    code = (data.get("unique_code") or data.get("uniqueCode")) if isinstance(data, dict) else None
    if isinstance(code, str) and len(code) <= 64:
        payout_id = Payout.objects.filter(unique_code=code).values_list("id", flat=True).first()
        if payout_id:
            payouts.reconcile_payout(payout_id)
    return JsonResponse({"ok": True})
```

## Run

```bash
python -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
set -a; . ./.env; set +a
python manage.py makemigrations ebank
python manage.py migrate
gunicorn config.wsgi --workers 2 --threads 4 --bind 127.0.0.1:8000 --timeout 60
```

Test against a local mock by setting `ZIBAL_BASE_URL`. There is no sandbox, so a real token would act on real accounts.

## Framework notes

- `CsrfViewMiddleware` would refuse Zibal's webhook with 403. Only the webhook is meant to be public. The internal routes use a header key, not cookies, so they are exempt as well. If you switch them to session auth, remove `csrf_exempt` from them.
- `requests` has no default timeout and follows redirects. Both are set on every call.
- Python's `json` accepts `NaN`, `Infinity` and very long integers. The type checks reject them, and parse errors, including very deep nesting, become 400.
- With `APPEND_SLASH = True` and slashed routes, a POST without the slash fails. These routes have no slash, so it is off.
- `DATA_UPLOAD_MAX_MEMORY_SIZE` caps request bodies.
- Keep the gunicorn `--timeout` above `ZIBAL_TIMEOUT_MS`, so workers are not killed mid-call. The `submitting` state covers the case where they are.
- Run one scheduled reconcile at a time.
