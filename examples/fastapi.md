# FastAPI

Tested with Python 3.14, FastAPI 0.162.0, Pydantic 2, httpx 0.28.1, SQLAlchemy 2.0 and uvicorn. Pin the same versions in your project, or re-test after changing them.

Handlers are plain `def`, so FastAPI runs them in a thread pool and the blocking `httpx.Client` and SQLAlchemy calls do not stall the event loop.

SQLite keeps the example self-contained. For production, use your own database and keep the unique constraints and the conditional updates.

Routes are the same as in [node-express.md](node-express.md). Every route except the webhook needs the `X-Internal-Key` header. It is a placeholder for your own authentication and authorization. Replace it before production.

Call `POST /internal/reconcile` from a scheduler. The webhook is sent once, so it cannot be the only signal.

## Files

### `requirements.txt`

```text
fastapi==0.162.0
httpx==0.28.1
sqlalchemy==2.0.44
uvicorn==0.38.0
```

### `.env.example`

```bash
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

### `app/__init__.py`

```python
```

### `app/config.py`

```python
import os


def _required(name: str) -> str:
    value = os.environ.get(name)
    if not value:
        raise RuntimeError(f"Missing environment variable {name}")
    return value


TOKEN = _required("ZIBAL_ACCESS_TOKEN")
ACCOUNT_ID = _required("ZIBAL_ACCOUNT_ID")
INTERNAL_KEY = _required("INTERNAL_API_KEY")
BASE_URL = os.environ.get("ZIBAL_BASE_URL") or "https://api.zibal.ir"
CALLBACK_URL = os.environ.get("ZIBAL_CALLBACK_URL") or None
TIMEOUT_S = int(os.environ.get("ZIBAL_TIMEOUT_MS") or 15000) / 1000
REVIEW_AFTER_S = int(os.environ.get("ZIBAL_REVIEW_AFTER_MS") or 900000) / 1000
RETURN_WATCH_S = int(os.environ.get("ZIBAL_RETURN_WATCH_DAYS") or 7) * 86400
DB_PATH = os.environ.get("DB_PATH") or "./ebank.sqlite"
```

### `app/db.py`

```python
import json

from sqlalchemy import BigInteger, Float, Integer, String, Text, UniqueConstraint, create_engine, event
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, sessionmaker

from . import config

engine = create_engine(f"sqlite:///{config.DB_PATH}", connect_args={"check_same_thread": False, "timeout": 5})


@event.listens_for(engine, "connect")
def _sqlite_pragmas(connection, _record):
    connection.execute("PRAGMA journal_mode = WAL")


Session = sessionmaker(engine, expire_on_commit=False)


class Base(DeclarativeBase):
    pass


class Payout(Base):
    __tablename__ = "payouts"

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    request_key: Mapped[str] = mapped_column(String(64), unique=True)
    request_hash: Mapped[str] = mapped_column(String(64))
    unique_code: Mapped[str] = mapped_column(String(36), unique=True)
    account_id: Mapped[str] = mapped_column(String(64))
    amount: Mapped[int] = mapped_column(BigInteger)
    iban: Mapped[str] = mapped_column(String(26))
    reason_code: Mapped[int] = mapped_column(Integer)
    delay: Mapped[int | None] = mapped_column(Integer)
    description: Mapped[str | None] = mapped_column(Text)
    payment_number: Mapped[str | None] = mapped_column(String(64))
    split: Mapped[bool | None]
    state: Mapped[str] = mapped_column(String(16))
    tracker_id: Mapped[str | None] = mapped_column(String(64))
    checkouts: Mapped[str] = mapped_column(Text, default="[]")
    last_result: Mapped[int | None] = mapped_column(Integer)
    last_message: Mapped[str | None] = mapped_column(Text)
    error_group: Mapped[str | None] = mapped_column(String(16))
    cancel_state: Mapped[str | None] = mapped_column(String(16))
    submitted_at: Mapped[float | None] = mapped_column(Float)
    created_at: Mapped[float] = mapped_column(Float)
    updated_at: Mapped[float] = mapped_column(Float)

    def checkout_list(self) -> list:
        return json.loads(self.checkouts)


class DepositDecision(Base):
    __tablename__ = "deposit_decisions"
    __table_args__ = (UniqueConstraint("account_id", "transaction_id"),)

    id: Mapped[str] = mapped_column(String(36), primary_key=True)
    account_id: Mapped[str] = mapped_column(String(64))
    transaction_id: Mapped[str] = mapped_column(String(64))
    status: Mapped[int] = mapped_column(Integer)
    state: Mapped[str] = mapped_column(String(16))
    refund_tracker_id: Mapped[str | None] = mapped_column(String(64))
    last_result: Mapped[int | None] = mapped_column(Integer)
    last_message: Mapped[str | None] = mapped_column(Text)
    submitted_at: Mapped[float | None] = mapped_column(Float)
    created_at: Mapped[float] = mapped_column(Float)
    updated_at: Mapped[float] = mapped_column(Float)


def create_tables() -> None:
    Base.metadata.create_all(engine)
```

### `app/zibal.py`

```python
from dataclasses import dataclass
from typing import Any

import httpx

from . import config

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

# httpx does not follow redirects by default. The timeout is explicit on purpose.
client = httpx.Client(
    base_url=config.BASE_URL,
    timeout=config.TIMEOUT_S,
    follow_redirects=False,
    headers={"Authorization": f"Bearer {config.TOKEN}"},
)


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
    try:
        response = client.request(method, path, **kwargs)
    except httpx.HTTPError:
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

### `app/payouts.py`

```python
import hashlib
import json
import logging
import time
import uuid

from sqlalchemy import or_, select, update
from sqlalchemy.dialects.sqlite import insert

from . import config, zibal
from .db import Payout, Session

log = logging.getLogger("zibal")

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
    with Session() as session:
        return session.get(Payout, payout_id)


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
    with Session.begin() as session:
        session.execute(
            update(Payout)
            .where(Payout.id == payout_id, Payout.state.in_(ALLOWED_FROM[state]))
            .values(state=state, updated_at=time.time(), **fields)
        )


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
    with Session.begin() as session:
        claim = session.execute(
            update(Payout)
            .where(Payout.id == payout_id, Payout.state == "created")
            .values(state="submitting", submitted_at=now, updated_at=now)
        )
    if claim.rowcount == 0:
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
    if config.CALLBACK_URL:
        body["callbackUrl"] = config.CALLBACK_URL

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
    payout_id = str(uuid.uuid4())

    with Session.begin() as session:
        inserted = session.execute(
            insert(Payout)
            .values(
                id=payout_id,
                request_key=values["requestKey"],
                request_hash=request_hash,
                unique_code=str(uuid.uuid4()),
                account_id=config.ACCOUNT_ID,
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
            .on_conflict_do_nothing(index_elements=["request_key"])
        )

    if inserted.rowcount == 0:
        with Session() as session:
            existing = session.scalar(select(Payout).where(Payout.request_key == values["requestKey"]))
        if existing.request_hash != request_hash:
            raise Conflict("request_key_reused")
        # 'created' means nothing was sent yet, so the claim in _submit() is still safe.
        if existing.state == "created":
            _submit(existing.id)
        return get_payout(existing.id), True

    _submit(payout_id)
    return get_payout(payout_id), False


def reconcile_payout(payout_id: str) -> Payout | None:
    payout = get_payout(payout_id)
    if payout is None or payout.state not in RECONCILABLE:
        return payout

    now = time.time()
    if payout.state == "submitting":
        # A live request is still inside its timeout. Do not race it.
        if now - payout.updated_at < config.TIMEOUT_S * 2:
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
        expired = now - (payout.submitted_at or payout.created_at) >= config.REVIEW_AFTER_S
        # Not found for a transfer Zibal already accepted is never normal.
        vanished = reply.result == 14 and not still_open and payout.state != "needs_review"
        if (still_open and expired) or vanished:
            _transition(payout_id, "needs_review", **base)
    log.info("payout_reconciled id=%s http=%s result=%s", payout_id, reply.http, reply.result)
    return get_payout(payout_id)


def reconcile_due() -> int:
    now = time.time()
    with Session() as session:
        ids = session.scalars(
            select(Payout.id)
            .where(or_(
                Payout.state.in_(["unknown", "processing"]),
                (Payout.state == "submitting") & (Payout.updated_at <= now - config.TIMEOUT_S * 2),
                (Payout.state == "succeeded") & (Payout.submitted_at >= now - config.RETURN_WATCH_S),
            ))
            .order_by(Payout.updated_at)
            .limit(50)
        ).all()
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

    with Session.begin() as session:
        claim = session.execute(
            update(Payout)
            .where(Payout.id == payout.id, Payout.cancel_state.is_(None), Payout.state == "processing")
            .values(cancel_state="sent", updated_at=time.time())
        )
    if claim.rowcount == 0:
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
        with Session.begin() as session:
            session.execute(
                update(Payout).where(Payout.id == payout.id)
                .values(cancel_state=None, last_result=reply.result, updated_at=time.time())
            )
        return {"http": 422, "payout": get_payout(payout.id), "cancelOutcome": "rejected", "cancelled": [], "uncancellable": []}
    # Unknown: keep cancel_state so it cannot be sent twice. Status comes from inquire.
    return {"http": 202, "payout": reconcile_payout(payout.id), "cancelOutcome": "unknown", "cancelled": [], "uncancellable": []}
```

### `app/deposits.py`

```python
import logging
import time
import uuid

from sqlalchemy import or_, select, update
from sqlalchemy.dialects.sqlite import insert

from . import config, zibal
from .db import DepositDecision, Session
from .payouts import Conflict

log = logging.getLogger("zibal")


def _find(transaction_id: str) -> DepositDecision | None:
    with Session() as session:
        return session.scalar(select(DepositDecision).where(
            DepositDecision.account_id == config.ACCOUNT_ID,
            DepositDecision.transaction_id == transaction_id,
        ))


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
    with Session.begin() as session:
        session.execute(
            update(DepositDecision)
            .where(DepositDecision.id == decision_id, DepositDecision.state.in_(allowed_from))
            .values(state=state, updated_at=time.time(), **fields)
        )


def _fetch_deposit(transaction_id: str) -> dict | None:
    """Returns the deposit, or None when the answer does not settle anything."""
    reply = zibal.inquire_deposit(config.ACCOUNT_ID, transaction_id)
    data = reply.json.get("data") if isinstance(reply.json, dict) else None
    if reply.http != 200 or reply.result != 1 or not isinstance(data, list):
        return None
    return next((item for item in data if isinstance(item, dict) and item.get("id") == transaction_id), None)


def decide_deposit(transaction_id: str, status: int) -> tuple[int, DepositDecision, bool]:
    now = time.time()
    with Session.begin() as session:
        inserted = session.execute(
            insert(DepositDecision)
            .values(
                id=str(uuid.uuid4()), account_id=config.ACCOUNT_ID, transaction_id=transaction_id,
                status=status, state="created", created_at=now, updated_at=now,
            )
            .on_conflict_do_nothing(index_elements=["account_id", "transaction_id"])
        )

    decision = _find(transaction_id)
    if decision.status != status:
        raise Conflict("different_decision_exists")
    duplicate = inserted.rowcount == 0

    with Session.begin() as session:
        claim = session.execute(
            update(DepositDecision)
            .where(DepositDecision.id == decision.id, DepositDecision.state == "created")
            .values(state="submitting", updated_at=time.time())
        )
    if claim.rowcount == 0:
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

    with Session.begin() as session:
        session.execute(update(DepositDecision).where(DepositDecision.id == decision.id).values(submitted_at=time.time()))
    reply = zibal.change_deposit_status(config.ACCOUNT_ID, transaction_id, status)
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
    if decision.state == "submitting" and time.time() - decision.updated_at >= config.TIMEOUT_S * 3:
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
        if changed_elsewhere or age >= config.REVIEW_AFTER_S:
            _set_state(decision.id, ["unknown"], "needs_review")
    return _find(transaction_id)


def reconcile_due() -> int:
    with Session() as session:
        ids = session.scalars(
            select(DepositDecision.transaction_id)
            .where(or_(
                DepositDecision.state == "unknown",
                (DepositDecision.state == "submitting") & (DepositDecision.updated_at <= time.time() - config.TIMEOUT_S * 3),
            ))
            .order_by(DepositDecision.updated_at)
            .limit(50)
        ).all()
    for transaction_id in ids:
        reconcile_deposit(transaction_id)
    return len(ids)
```

### `app/main.py`

```python
import hashlib
import hmac
import json
import logging
from typing import Annotated, Literal

from fastapi import Depends, FastAPI, Header, HTTPException, Path, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from pydantic import BaseModel, ConfigDict, Field, StrictBool, StrictInt, StrictStr
from sqlalchemy import select
from starlette.types import ASGIApp, Message, Receive, Scope, Send

from . import config, deposits, payouts, zibal
from .db import Payout, Session, create_tables

logging.basicConfig(level=logging.INFO, format="%(message)s")
create_tables()

MAX_BODY_BYTES = 65536


class BodyLimit:
    """FastAPI and uvicorn have no body size limit."""

    def __init__(self, app: ASGIApp):
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] != "http":
            return await self.app(scope, receive, send)
        size = 0
        messages: list[Message] = []
        while True:
            message = await receive()
            size += len(message.get("body", b""))
            if size > MAX_BODY_BYTES:
                response = JSONResponse({"error": "invalid_request"}, status_code=413)
                return await response(scope, receive, send)
            messages.append(message)
            if not message.get("more_body"):
                break

        async def replay() -> Message:
            return messages.pop(0) if messages else await receive()

        await self.app(scope, replay, send)


# The docs UI is off, so the API shape is not published.
app = FastAPI(docs_url=None, redoc_url=None, openapi_url=None)
app.add_middleware(BodyLimit)


# amount, iban and delay limits come from the docs. Other caps are this app's own choice.
# Strict mode stops "10000" or true being coerced into integers.
class PayoutBody(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    requestKey: Annotated[StrictStr, Field(pattern=r"^[A-Za-z0-9_-]{1,64}$")]
    amount: Annotated[StrictInt, Field(ge=10000, le=9007199254740991)]
    iban: Annotated[StrictStr, Field(pattern=r"^IR[0-9]{24}$")]
    reasonCode: Annotated[StrictInt, Field(ge=1, le=19)]
    delay: Annotated[StrictInt, Field(ge=-1, le=10080)] | None = None
    description: Annotated[StrictStr, Field(max_length=255, pattern=r"^[^\x00-\x1f\x7f]*$")] | None = None
    paymentNumber: Annotated[StrictStr, Field(max_length=64, pattern=r"^[^\x00-\x1f\x7f]*$")] | None = None
    split: StrictBool | None = None


class DecisionBody(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    status: Literal[2, 3]


def require_internal_key(x_internal_key: Annotated[str, Header()] = "") -> None:
    if not hmac.compare_digest(hashlib.sha256(x_internal_key.encode()).digest(), hashlib.sha256(config.INTERNAL_KEY.encode()).digest()):
        raise HTTPException(status_code=401, detail="unauthorized")


Protected = Depends(require_internal_key)
TransactionId = Annotated[str, Path(pattern=r"^[A-Za-z0-9_-]{1,64}$")]


# The default 422 body echoes the submitted values, which would expose IBANs.
@app.exception_handler(RequestValidationError)
async def invalid_request(_request: Request, _exc: RequestValidationError):
    return JSONResponse({"error": "invalid_request"}, status_code=400)


@app.exception_handler(HTTPException)
async def http_error(_request: Request, exc: HTTPException):
    return JSONResponse({"error": str(exc.detail)}, status_code=exc.status_code)


@app.exception_handler(payouts.Conflict)
async def conflict(_request: Request, exc: payouts.Conflict):
    return JSONResponse({"error": str(exc)}, status_code=409)


def http_status(state: str) -> int:
    if state == "failed":
        return 422
    if state in ("unknown", "needs_review", "submitting"):
        return 202
    return 201


@app.post("/payouts", dependencies=[Protected])
def create_payout(body: PayoutBody):
    payout, duplicate = payouts.create_payout(body.model_dump())
    code = 200 if duplicate else http_status(payout.state)
    return JSONResponse(payouts.view(payout, duplicate=duplicate), status_code=code)


@app.get("/payouts/{payout_id}", dependencies=[Protected])
def read_payout(payout_id: str):
    payout = payouts.get_payout(payout_id)
    if payout is None:
        raise HTTPException(status_code=404, detail="not_found")
    return payouts.view(payout)


@app.post("/payouts/{payout_id}/reconcile", dependencies=[Protected])
def reconcile_payout(payout_id: str):
    payout = payouts.reconcile_payout(payout_id)
    if payout is None:
        raise HTTPException(status_code=404, detail="not_found")
    return payouts.view(payout)


@app.post("/payouts/{payout_id}/cancel", dependencies=[Protected])
def cancel_payout(payout_id: str):
    payout = payouts.get_payout(payout_id)
    if payout is None:
        raise HTTPException(status_code=404, detail="not_found")
    outcome = payouts.cancel_payout(payout)
    body = payouts.view(
        outcome["payout"],
        cancelOutcome=outcome["cancelOutcome"],
        cancelled=outcome["cancelled"],
        uncancellable=outcome["uncancellable"],
    )
    return JSONResponse(body, status_code=outcome["http"])


@app.post("/internal/reconcile", dependencies=[Protected])
def reconcile_all():
    return {"processed": payouts.reconcile_due() + deposits.reconcile_due()}


@app.post("/deposits/{transaction_id}/decision", dependencies=[Protected])
def decide_deposit(transaction_id: TransactionId, body: DecisionBody):
    code, decision, duplicate = deposits.decide_deposit(transaction_id, body.status)
    return JSONResponse(deposits.view(decision, duplicate=duplicate), status_code=code)


@app.post("/deposits/{transaction_id}/reconcile", dependencies=[Protected])
def reconcile_deposit(transaction_id: TransactionId):
    decision = deposits.reconcile_deposit(transaction_id)
    if decision is None:
        raise HTTPException(status_code=404, detail="not_found")
    return deposits.view(decision)


@app.get("/balance", dependencies=[Protected])
def read_balance():
    reply = zibal.balance(config.ACCOUNT_ID)
    if reply.http is None:
        return JSONResponse({"error": "zibal_unreachable"}, status_code=504)
    # Balance has no result field on success.
    value = reply.json.get("balance") if isinstance(reply.json, dict) else None
    if reply.http == 200 and isinstance(value, int) and not isinstance(value, bool):
        return {"balance": value, "balanceUpdatedAt": reply.json.get("balanceUpdatedAt")}
    return JSONResponse(
        {"error": "zibal_error", "lastResult": reply.result, "errorGroup": zibal.result_group(reply.result)},
        status_code=502,
    )


async def _webhook_body(request: Request):
    try:
        return json.loads(await request.body())
    except (ValueError, RecursionError):
        return None


# Public on purpose. Only the unique code is read, and the state comes from inquire.
@app.post("/webhooks/zibal/checkout")
def webhook_receiver(body=Depends(_webhook_body)):
    data = body.get("data") if isinstance(body, dict) else None
    code = None
    if isinstance(data, dict):
        code = data.get("unique_code") or data.get("uniqueCode")
    if isinstance(code, str) and len(code) <= 64:
        with Session() as session:
            payout_id = session.scalar(select(Payout.id).where(Payout.unique_code == code))
        if payout_id:
            payouts.reconcile_payout(payout_id)
    return {"ok": True}
```

## Run

```bash
python -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
uvicorn app.main:app --env-file .env --port 8000
```

Test against a local mock by setting `ZIBAL_BASE_URL`. There is no sandbox, so a real token would act on real accounts.

## Framework notes

- Pydantic coerces in lax mode, so `"10000"` would become `10000`. `strict=True` and `StrictInt` prevent that.
- Pydantic patterns run on a Rust regex engine, where `$` is a strict end of string and `\Z` is a syntax error. In Python's `re`, `$` also matches before a trailing newline, so use `fullmatch` if you validate there. `\d` can match Persian digits, so the patterns use `[0-9]`.
- `extra="forbid"` rejects `__proto__`, `constructor` and every other unknown key.
- The default 422 response echoes input. The handler above replaces it, which keeps IBANs out of responses.
- FastAPI and uvicorn set no body limit. `BodyLimit` applies one.
- FastAPI has no CSRF protection by default, so the webhook needs no exemption.
- `httpx` does not follow redirects by default, and its default timeout is 5 seconds. Both are set explicitly here.
- `docs_url=None` hides the interactive docs. Do not expose them on a service that moves money.
- Run one scheduled reconcile at a time.
