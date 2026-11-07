# Node.js and Express

Tested with Node.js 22, Express 5.3.0, better-sqlite3 13.0.3 and zod 4.6.7. Node's built-in `fetch` is the HTTP client.

SQLite keeps the example self-contained. For production, use your own database and keep the unique indexes and the conditional updates.

Routes:

| Route | Purpose |
| :--- | :--- |
| `POST /payouts` | Create a transfer |
| `GET /payouts/:id` | Read local state |
| `POST /payouts/:id/reconcile` | Check the outcome with `checkout/inquire` |
| `POST /payouts/:id/cancel` | Cancel a pending transfer |
| `POST /internal/reconcile` | Scheduled job: re-check open transfers |
| `POST /deposits/:transactionId/decision` | Approve (`2`) or reject (`3`) a deposit |
| `POST /deposits/:transactionId/reconcile` | Check a decision's outcome |
| `GET /balance` | Read the balance |
| `POST /webhooks/zibal/checkout` | Webhook receiver |

Every route except the webhook needs the `X-Internal-Key` header. It is a placeholder for your own authentication and authorization. Replace it before production.

Call `POST /internal/reconcile` from a scheduler. The webhook is sent once, so it cannot be the only signal.

## Files

### `package.json`

```json
{
  "name": "zibal-ebank-express",
  "private": true,
  "type": "module",
  "engines": { "node": ">=22" },
  "scripts": { "start": "node src/server.js" },
  "dependencies": {
    "better-sqlite3": "13.0.3",
    "express": "5.3.0",
    "zod": "4.6.7"
  }
}
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
PORT=3000
DB_PATH=./ebank.sqlite
```

### `src/config.js`

```js
function required(name) {
  const value = process.env[name];
  if (!value) throw new Error(`Missing environment variable ${name}`);
  return value;
}

export const config = {
  token: required('ZIBAL_ACCESS_TOKEN'),
  accountId: required('ZIBAL_ACCOUNT_ID'),
  internalKey: required('INTERNAL_API_KEY'),
  baseUrl: process.env.ZIBAL_BASE_URL || 'https://api.zibal.ir',
  callbackUrl: process.env.ZIBAL_CALLBACK_URL || undefined,
  timeoutMs: Number(process.env.ZIBAL_TIMEOUT_MS || 15000),
  reviewAfterMs: Number(process.env.ZIBAL_REVIEW_AFTER_MS || 900000),
  returnWatchMs: Number(process.env.ZIBAL_RETURN_WATCH_DAYS || 7) * 86400000,
  port: Number(process.env.PORT || 3000),
  dbPath: process.env.DB_PATH || './ebank.sqlite',
};
```

### `src/db.js`

```js
import Database from 'better-sqlite3';
import { config } from './config.js';

export const db = new Database(config.dbPath);
db.pragma('journal_mode = WAL');
db.pragma('busy_timeout = 5000');

db.exec(`
  CREATE TABLE IF NOT EXISTS payouts (
    id TEXT PRIMARY KEY,
    request_key TEXT NOT NULL UNIQUE,
    request_hash TEXT NOT NULL,
    unique_code TEXT NOT NULL UNIQUE,
    account_id TEXT NOT NULL,
    amount INTEGER NOT NULL,
    iban TEXT NOT NULL,
    reason_code INTEGER NOT NULL,
    delay INTEGER,
    description TEXT,
    payment_number TEXT,
    split INTEGER,
    state TEXT NOT NULL,
    tracker_id TEXT,
    checkouts TEXT NOT NULL DEFAULT '[]',
    last_result INTEGER,
    last_message TEXT,
    error_group TEXT,
    cancel_state TEXT,
    submitted_at INTEGER,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
  );

  CREATE TABLE IF NOT EXISTS deposit_decisions (
    id TEXT PRIMARY KEY,
    account_id TEXT NOT NULL,
    transaction_id TEXT NOT NULL,
    status INTEGER NOT NULL,
    state TEXT NOT NULL,
    refund_tracker_id TEXT,
    last_result INTEGER,
    last_message TEXT,
    submitted_at INTEGER,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL,
    UNIQUE (account_id, transaction_id)
  );
`);
```

### `src/zibal.js`

```js
import { config } from './config.js';

// Grouping of the documented result table. See references/result-codes.md.
const RESULT_GROUPS = new Map([
  [2, 'auth'], [3, 'auth'], [4, 'auth'], [7, 'auth'], [8, 'auth'],
  [5, 'validation'], [6, 'validation'], [9, 'validation'], [21, 'validation'],
  [44, 'validation'], [48, 'validation'],
  [29, 'funds'],
  [14, 'not_found'],
  [45, 'provider'],
]);
const REJECTION_GROUPS = new Set(['auth', 'validation', 'funds']);
const CHECKOUT_STATUSES = new Set([-1, 0, 3, 4, 5, 20]);
const PENDING_STATUSES = new Set([-1, 0, 20]);

export function resultGroup(result) {
  return RESULT_GROUPS.get(result) ?? 'unknown';
}

// True only when Zibal clearly refused the request. Anything else is an unknown outcome.
export function isRejected(reply) {
  if (reply.http === null || reply.http < 200 || reply.http >= 500) return false;
  return REJECTION_GROUPS.has(resultGroup(reply.json?.result));
}

async function call(method, path, { query, body } = {}) {
  const url = new URL(path, config.baseUrl);
  for (const [key, value] of Object.entries(query ?? {})) url.searchParams.set(key, String(value));
  try {
    const response = await fetch(url, {
      method,
      redirect: 'manual',
      signal: AbortSignal.timeout(config.timeoutMs),
      headers: {
        Authorization: `Bearer ${config.token}`,
        ...(body ? { 'Content-Type': 'application/json' } : {}),
      },
      body: body ? JSON.stringify(body) : undefined,
    });
    const text = await response.text();
    let json = null;
    try {
      json = JSON.parse(text);
    } catch {
      json = null;
    }
    return { http: response.status, json };
  } catch {
    return { http: null, json: null };
  }
}

export const zibal = {
  createCheckout: (body) => call('POST', '/ebank/v1/account/checkout/create/', { body }),
  inquireCheckout: (uniqueCode) => call('GET', '/ebank/v1/account/checkout/inquire/', { query: { uniqueCode } }),
  cancelCheckout: (accountId, checkoutIds) =>
    call('POST', '/ebank/v1/account/checkout/cancel/', { body: { accountId, checkoutIds } }),
  balance: (accountId) => call('GET', '/ebank/v1/account/balance/', { query: { accountId } }),
  inquireDeposit: (accountId, transactionId) =>
    call('GET', '/ebank/v1/account/identified-payment/inquire/', { query: { accountId, transactionId } }),
  changeDepositStatus: (accountId, transactionId, status) =>
    call('POST', '/ebank/v1/account/identified-payment/change-status/', { body: { accountId, transactionId, status } }),
};

// Maps a transfer object to a local state. Every checkout counts, because split creates several.
export function checkoutOutcome(payout, data) {
  if (data === null || typeof data !== 'object') return 'needs_review';
  if (data.uniqueCode !== payout.unique_code || data.totalAmount !== payout.amount) return 'needs_review';
  const checkouts = data.checkouts;
  if (!Array.isArray(checkouts) || checkouts.length === 0) return 'needs_review';
  const statuses = checkouts.map((checkout) => checkout?.status);
  if (statuses.some((status) => !CHECKOUT_STATUSES.has(status))) return 'needs_review';
  if (statuses.some((status) => PENDING_STATUSES.has(status))) return 'processing';
  if (statuses.every((status) => status === 3)) {
    return checkouts.every((checkout) => checkout.refCode) ? 'succeeded' : 'needs_review';
  }
  if (statuses.every((status) => status === 5)) return 'failed';
  if (statuses.every((status) => status === 4)) return 'returned';
  return 'needs_review';
}

export function maskIban(iban) {
  return `${iban.slice(0, 4)}${'*'.repeat(18)}${iban.slice(-4)}`;
}
```

### `src/payouts.js`

```js
import { createHash, randomUUID } from 'node:crypto';
import { config } from './config.js';
import { db } from './db.js';
import { checkoutOutcome, isRejected, maskIban, resultGroup, zibal } from './zibal.js';

// A state can only be entered from these states, so a stale read cannot undo a newer one.
const ALLOWED_FROM = {
  unknown: ['submitting'],
  processing: ['submitting', 'unknown', 'processing', 'needs_review'],
  succeeded: ['submitting', 'unknown', 'processing', 'succeeded', 'needs_review'],
  returned: ['submitting', 'unknown', 'processing', 'succeeded', 'needs_review'],
  failed: ['submitting', 'unknown', 'processing', 'needs_review'],
  needs_review: ['submitting', 'unknown', 'processing', 'succeeded', 'needs_review'],
};
const RECONCILABLE = new Set(['submitting', 'unknown', 'processing', 'succeeded', 'needs_review']);
const PENDING_STATUSES = new Set([-1, 0, 20]);

const log = (event, fields) => console.info(JSON.stringify({ event, ...fields }));

export class Conflict extends Error {}

function find(id) {
  return db.prepare('SELECT * FROM payouts WHERE id = ?').get(id);
}

export function getPayout(id) {
  return find(id);
}

export function view(payout, extra = {}) {
  return {
    id: payout.id,
    requestKey: payout.request_key,
    uniqueCode: payout.unique_code,
    state: payout.state,
    amount: payout.amount,
    iban: maskIban(payout.iban),
    trackerId: payout.tracker_id,
    lastResult: payout.last_result,
    errorGroup: payout.error_group,
    checkouts: JSON.parse(payout.checkouts),
    ...extra,
  };
}

function transition(id, state, fields = {}) {
  const from = ALLOWED_FROM[state];
  const columns = { ...fields, state, updated_at: Date.now() };
  const assignments = Object.keys(columns).map((column) => `${column} = @${column}`).join(', ');
  const marks = from.map((_, index) => `@from${index}`).join(', ');
  const params = { ...columns, id };
  from.forEach((value, index) => { params[`from${index}`] = value; });
  db.prepare(`UPDATE payouts SET ${assignments} WHERE id = @id AND state IN (${marks})`).run(params);
}

function checkoutSummary(data) {
  if (!Array.isArray(data?.checkouts)) return '[]';
  return JSON.stringify(data.checkouts.map((checkout) => ({
    id: checkout?.id ?? null,
    amount: checkout?.amount ?? null,
    status: checkout?.status ?? null,
    refCode: checkout?.refCode ?? null,
    failReason: checkout?.failReason ?? null,
  })));
}

async function submit(id) {
  const now = Date.now();
  const claim = db.prepare(`
    UPDATE payouts SET state = 'submitting', submitted_at = ?, updated_at = ?
    WHERE id = ? AND state = 'created'
  `).run(now, now, id);
  if (claim.changes === 0) return;

  const payout = find(id);
  const reply = await zibal.createCheckout({
    amount: payout.amount,
    iban: payout.iban,
    accountId: payout.account_id,
    reasonCode: payout.reason_code,
    uniqueCode: payout.unique_code,
    ...(payout.delay !== null ? { delay: payout.delay } : {}),
    ...(payout.description !== null ? { description: payout.description } : {}),
    ...(payout.payment_number !== null ? { paymentNumber: payout.payment_number } : {}),
    ...(payout.split !== null ? { split: payout.split === 1 } : {}),
    ...(config.callbackUrl ? { callbackUrl: config.callbackUrl } : {}),
  });

  const result = Number.isInteger(reply.json?.result) ? reply.json.result : null;
  const message = typeof reply.json?.message === 'string' ? reply.json.message : null;
  const base = { last_result: result, last_message: message };

  if (reply.http === 200 && result === 1 && reply.json.data !== null && typeof reply.json.data === 'object') {
    const data = reply.json.data;
    transition(id, checkoutOutcome(payout, data), {
      ...base,
      tracker_id: typeof data.trackerId === 'string' ? data.trackerId : null,
      checkouts: checkoutSummary(data),
    });
  } else if (isRejected(reply)) {
    transition(id, 'failed', { ...base, error_group: resultGroup(result) });
  } else {
    transition(id, 'unknown', { ...base, error_group: resultGroup(result) });
  }
  log('payout_submitted', { id, iban: maskIban(payout.iban), http: reply.http, result, state: find(id).state });
}

export async function createPayout(input) {
  const hash = createHash('sha256').update(JSON.stringify([
    input.amount, input.iban, input.reasonCode, input.delay ?? null,
    input.description ?? null, input.paymentNumber ?? null, input.split ?? null,
  ])).digest('hex');
  const now = Date.now();
  const id = randomUUID();

  const inserted = db.prepare(`
    INSERT INTO payouts (
      id, request_key, request_hash, unique_code, account_id, amount, iban, reason_code,
      delay, description, payment_number, split, state, created_at, updated_at
    ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, 'created', ?, ?)
    ON CONFLICT (request_key) DO NOTHING
  `).run(
    id, input.requestKey, hash, randomUUID(), config.accountId, input.amount, input.iban, input.reasonCode,
    input.delay ?? null, input.description ?? null, input.paymentNumber ?? null,
    input.split === undefined ? null : Number(input.split), now, now,
  );

  if (inserted.changes === 0) {
    const existing = db.prepare('SELECT * FROM payouts WHERE request_key = ?').get(input.requestKey);
    if (existing.request_hash !== hash) throw new Conflict('request_key_reused');
    // 'created' means nothing was sent yet, so the claim in submit() is still safe.
    if (existing.state === 'created') await submit(existing.id);
    return { payout: find(existing.id), duplicate: true };
  }

  await submit(id);
  return { payout: find(id), duplicate: false };
}

export async function reconcilePayout(id) {
  const payout = find(id);
  if (!payout || !RECONCILABLE.has(payout.state)) return payout;

  const now = Date.now();
  if (payout.state === 'submitting') {
    // A live request is still inside its timeout. Do not race it.
    if (now - payout.updated_at < config.timeoutMs * 2) return payout;
    transition(id, 'unknown');
  }

  const reply = await zibal.inquireCheckout(payout.unique_code);
  const result = Number.isInteger(reply.json?.result) ? reply.json.result : null;
  const base = {
    last_result: result,
    last_message: typeof reply.json?.message === 'string' ? reply.json.message : null,
  };

  if (reply.http === 200 && result === 1) {
    const data = reply.json.data;
    transition(id, checkoutOutcome(payout, data), {
      ...base,
      tracker_id: typeof data?.trackerId === 'string' ? data.trackerId : payout.tracker_id,
      checkouts: checkoutSummary(data),
      error_group: null,
    });
  } else {
    const open = payout.state === 'unknown' || payout.state === 'submitting';
    const expired = now - (payout.submitted_at ?? payout.created_at) >= config.reviewAfterMs;
    // Not found for a transfer Zibal already accepted is never normal.
    const vanished = result === 14 && !open && payout.state !== 'needs_review';
    if ((open && expired) || vanished) transition(id, 'needs_review', base);
  }
  log('payout_reconciled', { id, http: reply.http, result, state: find(id).state });
  return find(id);
}

export async function reconcileDue() {
  const now = Date.now();
  const rows = db.prepare(`
    SELECT id FROM payouts
    WHERE state IN ('unknown', 'processing')
       OR (state = 'submitting' AND updated_at <= ?)
       OR (state = 'succeeded' AND submitted_at >= ?)
    ORDER BY updated_at LIMIT 50
  `).all(now - config.timeoutMs * 2, now - config.returnWatchMs);
  for (const row of rows) await reconcilePayout(row.id);
  return rows.length;
}

export async function cancelPayout(payout) {
  const ids = JSON.parse(payout.checkouts)
    .filter((checkout) => checkout.id && PENDING_STATUSES.has(checkout.status))
    .map((checkout) => checkout.id);
  if (ids.length === 0) throw new Conflict('not_cancellable');

  const claim = db.prepare(`
    UPDATE payouts SET cancel_state = 'sent', updated_at = ?
    WHERE id = ? AND cancel_state IS NULL AND state = 'processing'
  `).run(Date.now(), payout.id);
  if (claim.changes === 0) throw new Conflict('not_cancellable');

  const reply = await zibal.cancelCheckout(payout.account_id, ids);
  const result = Number.isInteger(reply.json?.result) ? reply.json.result : null;

  if (reply.http === 200 && result === 1) {
    const data = reply.json.data ?? {};
    const updated = await reconcilePayout(payout.id);
    return {
      http: 200,
      payout: updated,
      cancelOutcome: 'answered',
      cancelled: Array.isArray(data.cancelled_checkouts) ? data.cancelled_checkouts : [],
      uncancellable: Array.isArray(data.uncancellable_checkouts) ? data.uncancellable_checkouts : [],
    };
  }
  if (isRejected(reply)) {
    db.prepare('UPDATE payouts SET cancel_state = NULL, last_result = ?, updated_at = ? WHERE id = ?')
      .run(result, Date.now(), payout.id);
    return { http: 422, payout: find(payout.id), cancelOutcome: 'rejected', cancelled: [], uncancellable: [] };
  }
  // Unknown: keep cancel_state so it cannot be sent twice. Status comes from inquire.
  const updated = await reconcilePayout(payout.id);
  return { http: 202, payout: updated, cancelOutcome: 'unknown', cancelled: [], uncancellable: [] };
}
```

### `src/deposits.js`

```js
import { randomUUID } from 'node:crypto';
import { config } from './config.js';
import { db } from './db.js';
import { Conflict } from './payouts.js';
import { isRejected, zibal } from './zibal.js';

const log = (event, fields) => console.info(JSON.stringify({ event, ...fields }));

function find(transactionId) {
  return db.prepare('SELECT * FROM deposit_decisions WHERE account_id = ? AND transaction_id = ?')
    .get(config.accountId, transactionId);
}

export function view(decision, extra = {}) {
  return {
    id: decision.id,
    transactionId: decision.transaction_id,
    status: decision.status,
    state: decision.state,
    refundTrackerId: decision.refund_tracker_id,
    lastResult: decision.last_result,
    ...extra,
  };
}

function setState(id, from, state, fields = {}) {
  const columns = { ...fields, state, updated_at: Date.now() };
  const assignments = Object.keys(columns).map((column) => `${column} = @${column}`).join(', ');
  const marks = from.map((_, index) => `@from${index}`).join(', ');
  const params = { ...columns, id };
  from.forEach((value, index) => { params[`from${index}`] = value; });
  db.prepare(`UPDATE deposit_decisions SET ${assignments} WHERE id = @id AND state IN (${marks})`).run(params);
}

// Returns the deposit, or null when the answer does not settle anything.
async function fetchDeposit(transactionId) {
  const reply = await zibal.inquireDeposit(config.accountId, transactionId);
  if (reply.http !== 200 || reply.json?.result !== 1 || !Array.isArray(reply.json.data)) return null;
  return reply.json.data.find((item) => item?.id === transactionId) ?? null;
}

export async function decideDeposit(transactionId, status) {
  const now = Date.now();
  const inserted = db.prepare(`
    INSERT INTO deposit_decisions (id, account_id, transaction_id, status, state, created_at, updated_at)
    VALUES (?, ?, ?, ?, 'created', ?, ?)
    ON CONFLICT (account_id, transaction_id) DO NOTHING
  `).run(randomUUID(), config.accountId, transactionId, status, now, now);

  const decision = find(transactionId);
  if (decision.status !== status) throw new Conflict('different_decision_exists');
  const duplicate = inserted.changes === 0;

  const claim = db.prepare(`
    UPDATE deposit_decisions SET state = 'submitting', updated_at = ?
    WHERE id = ? AND state = 'created'
  `).run(Date.now(), decision.id);
  if (claim.changes === 0) return { http: 200, decision: find(transactionId), duplicate: true };

  const deposit = await fetchDeposit(transactionId);
  if (!deposit) {
    // Nothing was sent, so release the claim and let the caller try again.
    setState(decision.id, ['submitting'], 'created');
    return { http: 503, decision: find(transactionId), duplicate };
  }
  if (deposit.verifyStatus !== 0) {
    if (deposit.verifyStatus === status) {
      setState(decision.id, ['submitting'], 'done');
      return { http: 200, decision: find(transactionId), duplicate };
    }
    if (deposit.verifyStatus === 2 || deposit.verifyStatus === 3) {
      setState(decision.id, ['submitting'], 'failed');
      return { http: 409, decision: find(transactionId), duplicate };
    }
    setState(decision.id, ['submitting'], 'needs_review');
    return { http: 202, decision: find(transactionId), duplicate };
  }

  db.prepare('UPDATE deposit_decisions SET submitted_at = ? WHERE id = ?').run(Date.now(), decision.id);
  const reply = await zibal.changeDepositStatus(config.accountId, transactionId, status);
  const result = Number.isInteger(reply.json?.result) ? reply.json.result : null;
  const base = { last_result: result, last_message: typeof reply.json?.message === 'string' ? reply.json.message : null };

  let http;
  if (reply.http === 200 && reply.json?.success === true) {
    const tracker = reply.json.data?.refundRequestTrackerId;
    setState(decision.id, ['submitting'], 'done', { ...base, refund_tracker_id: typeof tracker === 'string' ? tracker : null });
    http = 200;
  } else if (isRejected(reply)) {
    setState(decision.id, ['submitting'], 'failed', base);
    http = 422;
  } else {
    setState(decision.id, ['submitting'], 'unknown', base);
    http = 202;
  }
  log('deposit_decided', { id: decision.id, http: reply.http, result, state: find(transactionId).state });
  return { http, decision: find(transactionId), duplicate };
}

export async function reconcileDeposit(transactionId) {
  const decision = find(transactionId);
  if (!decision) return decision;
  if (decision.state === 'submitting' && Date.now() - decision.updated_at >= config.timeoutMs * 3) {
    // The process died after claiming. The change-status call may or may not have been sent.
    setState(decision.id, ['submitting'], 'unknown');
  } else if (decision.state !== 'unknown') {
    return decision;
  }

  const deposit = await fetchDeposit(transactionId);
  if (deposit?.verifyStatus === decision.status) {
    setState(decision.id, ['unknown'], 'done');
  } else {
    const age = Date.now() - (decision.submitted_at ?? decision.created_at);
    // A changed status that is not ours, or no change after the window, needs a person.
    const changedElsewhere = deposit && deposit.verifyStatus !== 0;
    if (changedElsewhere || age >= config.reviewAfterMs) setState(decision.id, ['unknown'], 'needs_review');
  }
  return find(transactionId);
}

export async function reconcileDue() {
  const rows = db.prepare(`
    SELECT transaction_id FROM deposit_decisions
    WHERE state = 'unknown' OR (state = 'submitting' AND updated_at <= ?)
    ORDER BY updated_at LIMIT 50
  `).all(Date.now() - config.timeoutMs * 3);
  for (const row of rows) await reconcileDeposit(row.transaction_id);
  return rows.length;
}
```

### `src/server.js`

```js
import { createHash, timingSafeEqual } from 'node:crypto';
import express from 'express';
import { z } from 'zod';
import { config } from './config.js';
import { db } from './db.js';
import * as deposits from './deposits.js';
import * as payouts from './payouts.js';
import { resultGroup, zibal } from './zibal.js';

// amount, iban and delay limits come from the docs. Other caps are this app's own choice.
const text = (max) => z.string().max(max).regex(/^[^\p{Cc}]*$/u);
const payoutSchema = z.strictObject({
  requestKey: z.string().regex(/^[A-Za-z0-9_-]{1,64}$/),
  amount: z.number().int().min(10000).max(Number.MAX_SAFE_INTEGER),
  iban: z.string().regex(/^IR[0-9]{24}$/),
  reasonCode: z.number().int().min(1).max(19),
  delay: z.number().int().min(-1).max(10080).optional(),
  description: text(255).optional(),
  paymentNumber: text(64).optional(),
  split: z.boolean().optional(),
});
const decisionSchema = z.strictObject({ status: z.union([z.literal(2), z.literal(3)]) });
const transactionIdSchema = z.string().regex(/^[A-Za-z0-9_-]{1,64}$/);

const digest = (value) => createHash('sha256').update(String(value)).digest();

function requireInternalKey(req, res, next) {
  const given = req.get('x-internal-key') ?? '';
  if (!timingSafeEqual(digest(given), digest(config.internalKey))) {
    return res.status(401).json({ error: 'unauthorized' });
  }
  return next();
}

function httpStatus(state) {
  if (state === 'failed') return 422;
  if (state === 'unknown' || state === 'needs_review' || state === 'submitting') return 202;
  return 201;
}

const app = express();
app.disable('x-powered-by');

// Public on purpose. Only the unique code is read, and the state comes from inquire.
app.post('/webhooks/zibal/checkout', express.json({ limit: '64kb' }), async (req, res) => {
  const data = req.body?.data;
  const code = data?.unique_code ?? data?.uniqueCode;
  if (typeof code === 'string' && code.length <= 64) {
    const row = db.prepare('SELECT id FROM payouts WHERE unique_code = ?').get(code);
    if (row) await payouts.reconcilePayout(row.id);
  }
  res.json({ ok: true });
});

app.use(requireInternalKey);
app.use(express.json({ limit: '16kb' }));

app.post('/payouts', async (req, res) => {
  const parsed = payoutSchema.safeParse(req.body);
  if (!parsed.success) return res.status(400).json({ error: 'invalid_request' });
  try {
    const { payout, duplicate } = await payouts.createPayout(parsed.data);
    return res.status(duplicate ? 200 : httpStatus(payout.state)).json(payouts.view(payout, { duplicate }));
  } catch (error) {
    if (error instanceof payouts.Conflict) return res.status(409).json({ error: error.message });
    throw error;
  }
});

app.get('/payouts/:id', (req, res) => {
  const payout = payouts.getPayout(req.params.id);
  if (!payout) return res.status(404).json({ error: 'not_found' });
  return res.json(payouts.view(payout));
});

app.post('/payouts/:id/reconcile', async (req, res) => {
  const payout = await payouts.reconcilePayout(req.params.id);
  if (!payout) return res.status(404).json({ error: 'not_found' });
  return res.json(payouts.view(payout));
});

app.post('/payouts/:id/cancel', async (req, res) => {
  const payout = payouts.getPayout(req.params.id);
  if (!payout) return res.status(404).json({ error: 'not_found' });
  try {
    const outcome = await payouts.cancelPayout(payout);
    return res.status(outcome.http).json(payouts.view(outcome.payout, {
      cancelOutcome: outcome.cancelOutcome,
      cancelled: outcome.cancelled,
      uncancellable: outcome.uncancellable,
    }));
  } catch (error) {
    if (error instanceof payouts.Conflict) return res.status(409).json({ error: error.message });
    throw error;
  }
});

app.post('/internal/reconcile', async (req, res) => {
  res.json({ processed: (await payouts.reconcileDue()) + (await deposits.reconcileDue()) });
});

app.post('/deposits/:transactionId/decision', async (req, res) => {
  const id = transactionIdSchema.safeParse(req.params.transactionId);
  const body = decisionSchema.safeParse(req.body);
  if (!id.success || !body.success) return res.status(400).json({ error: 'invalid_request' });
  try {
    const { http, decision, duplicate } = await deposits.decideDeposit(id.data, body.data.status);
    return res.status(http).json(deposits.view(decision, { duplicate }));
  } catch (error) {
    if (error instanceof payouts.Conflict) return res.status(409).json({ error: error.message });
    throw error;
  }
});

app.post('/deposits/:transactionId/reconcile', async (req, res) => {
  const decision = await deposits.reconcileDeposit(req.params.transactionId);
  if (!decision) return res.status(404).json({ error: 'not_found' });
  return res.json(deposits.view(decision));
});

app.get('/balance', async (req, res) => {
  const reply = await zibal.balance(config.accountId);
  if (reply.http === null) return res.status(504).json({ error: 'zibal_unreachable' });
  // Balance has no result field on success.
  if (reply.http === 200 && Number.isSafeInteger(reply.json?.balance)) {
    return res.json({ balance: reply.json.balance, balanceUpdatedAt: reply.json.balanceUpdatedAt });
  }
  const result = Number.isInteger(reply.json?.result) ? reply.json.result : null;
  return res.status(502).json({ error: 'zibal_error', lastResult: result, errorGroup: resultGroup(result) });
});

// Answers body parser errors too. It never echoes the submitted value.
app.use((error, req, res, next) => {
  if (res.headersSent) return next(error);
  const status = error.status >= 400 && error.status < 500 ? error.status : 500;
  if (status === 500) console.error(JSON.stringify({ event: 'unhandled_error', name: error.name }));
  return res.status(status).json({ error: status === 500 ? 'internal_error' : 'invalid_request' });
});

app.listen(config.port);
```

## Run

```bash
npm install
cp .env.example .env
node --env-file=.env src/server.js
```

Test against a local mock by setting `ZIBAL_BASE_URL`. There is no sandbox, so a real token would act on real accounts.

```bash
curl -X POST http://localhost:3000/payouts \
  -H 'X-Internal-Key: <your key>' -H 'Content-Type: application/json' \
  -d '{"requestKey":"order-1001","amount":10000,"iban":"IR000000000000000000000000","reasonCode":4}'
```

## Framework notes

- `fetch` has no default timeout, so `AbortSignal.timeout` is mandatory.
- `redirect: 'manual'` stops a redirect from being followed. A followed POST can turn into a GET and hide the outcome.
- Express 5 leaves `req.body` undefined when no body was parsed. The zod schema rejects it.
- `z.strictObject` rejects `__proto__` and `constructor`, which `JSON.parse` creates as own keys.
- The default `express.json` limit is 100kb. It is lowered here.
- JavaScript cannot hold every int64 exactly. `Number.MAX_SAFE_INTEGER` is the upper bound here for that reason.
- Express has no CSRF protection by default, so the webhook needs no exemption. If you add cookie-based CSRF middleware, keep it away from this route.
- Run one scheduled reconcile at a time.
