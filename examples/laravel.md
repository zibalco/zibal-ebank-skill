# Laravel

Tested with PHP 8.5 and Laravel 13.40.0 on a fresh `laravel/laravel` skeleton, with SQLite and four PHP server workers. Laravel's `Http` client is the HTTP client.

Add the files below to the skeleton. `bootstrap/app.php` and `routes/api.php` replace the skeleton's versions.

SQLite keeps the example self-contained. For production, use your own database and keep the unique indexes and the conditional updates.

Routes are the same as in [node-express.md](node-express.md). Every route except the webhook needs the `X-Internal-Key` header. It is a placeholder for your own authentication and authorization. Replace it before production.

Run `php artisan zibal:reconcile` from the scheduler, or call `POST /internal/reconcile`. The webhook is sent once, so it cannot be the only signal.

## Files

### `.env.additions`

```bash
# Append to .env
ZIBAL_ACCESS_TOKEN=
ZIBAL_ACCOUNT_ID=
# ZIBAL_BASE_URL=https://api.zibal.ir
# ZIBAL_CALLBACK_URL=https://your-domain.example/webhooks/zibal/checkout
# ZIBAL_TIMEOUT_MS=15000
# ZIBAL_REVIEW_AFTER_MS=900000
# ZIBAL_RETURN_WATCH_DAYS=7
INTERNAL_API_KEY=
```

### `config/zibal.php`

```php
<?php

return [
    'token' => env('ZIBAL_ACCESS_TOKEN'),
    'account_id' => env('ZIBAL_ACCOUNT_ID'),
    'internal_key' => env('INTERNAL_API_KEY'),
    'base_url' => env('ZIBAL_BASE_URL', 'https://api.zibal.ir'),
    'callback_url' => env('ZIBAL_CALLBACK_URL'),
    'timeout_s' => (int) env('ZIBAL_TIMEOUT_MS', 15000) / 1000,
    'review_after_s' => (int) env('ZIBAL_REVIEW_AFTER_MS', 900000) / 1000,
    'return_watch_s' => (int) env('ZIBAL_RETURN_WATCH_DAYS', 7) * 86400,
];
```

### `bootstrap/app.php`

```php
<?php

use App\Http\Middleware\InternalKey;
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        api: __DIR__.'/../routes/api.php',
        // Routes sit at the root, to match the other examples. Remove this line to get /api/....
        apiPrefix: '',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void {
        $middleware->alias(['internal.key' => InternalKey::class]);
    })
    ->withExceptions(function (Exceptions $exceptions): void {
        $exceptions->shouldRenderJsonWhen(fn () => true);
    })
    ->create();
```

### `routes/api.php`

```php
<?php

use App\Http\Controllers\EbankController;
use Illuminate\Support\Facades\Route;

// Routes in api.php are outside the web group, so CSRF does not apply to the webhook.
Route::post('/webhooks/zibal/checkout', [EbankController::class, 'webhook']);

Route::middleware('internal.key')->group(function () {
    Route::post('/payouts', [EbankController::class, 'createPayout']);
    Route::get('/payouts/{id}', [EbankController::class, 'readPayout']);
    Route::post('/payouts/{id}/reconcile', [EbankController::class, 'reconcilePayout']);
    Route::post('/payouts/{id}/cancel', [EbankController::class, 'cancelPayout']);
    Route::post('/internal/reconcile', [EbankController::class, 'reconcileAll']);
    Route::post('/deposits/{transactionId}/decision', [EbankController::class, 'decideDeposit']);
    Route::post('/deposits/{transactionId}/reconcile', [EbankController::class, 'reconcileDeposit']);
    Route::get('/balance', [EbankController::class, 'balance']);
});
```

### `routes/console.php`

```php
<?php

use App\Services\Zibal\DepositService;
use App\Services\Zibal\PayoutService;
use Illuminate\Support\Facades\Artisan;
use Illuminate\Support\Facades\Schedule;

Artisan::command('zibal:reconcile', function (PayoutService $payouts, DepositService $deposits) {
    $this->info('processed '.($payouts->reconcileDue() + $deposits->reconcileDue()));
});

Schedule::command('zibal:reconcile')->everyFiveMinutes()->withoutOverlapping();
```

### `database/migrations/2026_01_01_000000_create_ebank_tables.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('payouts', function (Blueprint $table) {
            $table->string('id', 36)->primary();
            $table->string('request_key', 64)->unique();
            $table->string('request_hash', 64);
            $table->string('unique_code', 36)->unique();
            $table->string('account_id', 64);
            $table->bigInteger('amount');
            $table->string('iban', 26);
            $table->integer('reason_code');
            $table->integer('delay')->nullable();
            $table->text('description')->nullable();
            $table->string('payment_number', 64)->nullable();
            $table->boolean('split')->nullable();
            $table->string('state', 16);
            $table->string('tracker_id', 64)->nullable();
            $table->text('checkouts')->default('[]');
            $table->integer('last_result')->nullable();
            $table->text('last_message')->nullable();
            $table->string('error_group', 16)->nullable();
            $table->string('cancel_state', 16)->nullable();
            $table->double('submitted_at')->nullable();
            $table->double('created_at');
            $table->double('updated_at');
        });

        Schema::create('deposit_decisions', function (Blueprint $table) {
            $table->string('id', 36)->primary();
            $table->string('account_id', 64);
            $table->string('transaction_id', 64);
            $table->integer('status');
            $table->string('state', 16);
            $table->string('refund_tracker_id', 64)->nullable();
            $table->integer('last_result')->nullable();
            $table->text('last_message')->nullable();
            $table->double('submitted_at')->nullable();
            $table->double('created_at');
            $table->double('updated_at');
            $table->unique(['account_id', 'transaction_id']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('deposit_decisions');
        Schema::dropIfExists('payouts');
    }
};
```

### `app/Models/Payout.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Payout extends Model
{
    public $incrementing = false;

    public $timestamps = false;

    protected $keyType = 'string';

    protected $guarded = [];

    protected $hidden = ['iban'];

    protected function casts(): array
    {
        return [
            'amount' => 'integer',
            'reason_code' => 'integer',
            'delay' => 'integer',
            'split' => 'boolean',
            'last_result' => 'integer',
            'submitted_at' => 'float',
            'created_at' => 'float',
            'updated_at' => 'float',
        ];
    }

    public function checkoutList(): array
    {
        return json_decode($this->checkouts, true) ?? [];
    }
}
```

### `app/Models/DepositDecision.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class DepositDecision extends Model
{
    public $incrementing = false;

    public $timestamps = false;

    protected $keyType = 'string';

    protected $guarded = [];

    protected function casts(): array
    {
        return [
            'status' => 'integer',
            'last_result' => 'integer',
            'submitted_at' => 'float',
            'created_at' => 'float',
            'updated_at' => 'float',
        ];
    }
}
```

### `app/Services/Zibal/Reply.php`

```php
<?php

namespace App\Services\Zibal;

final class Reply
{
    public function __construct(public readonly ?int $http, public readonly mixed $json)
    {
    }

    public function result(): ?int
    {
        return is_array($this->json) && is_int($this->json['result'] ?? null) ? $this->json['result'] : null;
    }

    public function message(): ?string
    {
        return is_array($this->json) && is_string($this->json['message'] ?? null) ? $this->json['message'] : null;
    }

    public function get(string $key): mixed
    {
        return is_array($this->json) ? ($this->json[$key] ?? null) : null;
    }
}
```

### `app/Services/Zibal/ZibalClient.php`

```php
<?php

namespace App\Services\Zibal;

use GuzzleHttp\Exception\TransferException;
use Illuminate\Http\Client\ConnectionException;
use Illuminate\Support\Facades\Http;

class ZibalClient
{
    // Grouping of the documented result table. See references/result-codes.md.
    private const RESULT_GROUPS = [
        2 => 'auth', 3 => 'auth', 4 => 'auth', 7 => 'auth', 8 => 'auth',
        5 => 'validation', 6 => 'validation', 9 => 'validation', 21 => 'validation',
        44 => 'validation', 48 => 'validation',
        29 => 'funds',
        14 => 'not_found',
        45 => 'provider',
    ];

    private const REJECTION_GROUPS = ['auth', 'validation', 'funds'];

    private const CHECKOUT_STATUSES = [-1, 0, 3, 4, 5, 20];

    public const PENDING_STATUSES = [-1, 0, 20];

    public static function resultGroup(?int $result): string
    {
        return $result !== null && isset(self::RESULT_GROUPS[$result]) ? self::RESULT_GROUPS[$result] : 'unknown';
    }

    /** True only when Zibal clearly refused the request. Anything else is an unknown outcome. */
    public static function isRejected(Reply $reply): bool
    {
        if ($reply->http === null || $reply->http < 200 || $reply->http >= 500) {
            return false;
        }

        return in_array(self::resultGroup($reply->result()), self::REJECTION_GROUPS, true);
    }

    private function call(string $method, string $path, array $data = []): Reply
    {
        $timeout = config('zibal.timeout_s');
        try {
            // No ->retry(): a POST that moves money must never be resent by the client.
            $response = Http::baseUrl(config('zibal.base_url'))
                ->withToken(config('zibal.token'))
                ->acceptJson()
                ->timeout($timeout)
                ->connectTimeout($timeout)
                ->withoutRedirecting()
                ->send($method, $path, $method === 'GET' ? ['query' => $data] : ['json' => $data]);
        } catch (ConnectionException|TransferException) {
            return new Reply(null, null);
        }
        try {
            $json = json_decode($response->body(), true, 64, JSON_BIGINT_AS_STRING | JSON_THROW_ON_ERROR);
        } catch (\JsonException) {
            $json = null;
        }

        return new Reply($response->status(), $json);
    }

    public function createCheckout(array $body): Reply
    {
        return $this->call('POST', '/ebank/v1/account/checkout/create/', $body);
    }

    public function inquireCheckout(string $uniqueCode): Reply
    {
        return $this->call('GET', '/ebank/v1/account/checkout/inquire/', ['uniqueCode' => $uniqueCode]);
    }

    public function cancelCheckout(string $accountId, array $checkoutIds): Reply
    {
        return $this->call('POST', '/ebank/v1/account/checkout/cancel/', ['accountId' => $accountId, 'checkoutIds' => array_values($checkoutIds)]);
    }

    public function balance(string $accountId): Reply
    {
        return $this->call('GET', '/ebank/v1/account/balance/', ['accountId' => $accountId]);
    }

    public function inquireDeposit(string $accountId, string $transactionId): Reply
    {
        return $this->call('GET', '/ebank/v1/account/identified-payment/inquire/', ['accountId' => $accountId, 'transactionId' => $transactionId]);
    }

    public function changeDepositStatus(string $accountId, string $transactionId, int $status): Reply
    {
        return $this->call('POST', '/ebank/v1/account/identified-payment/change-status/', ['accountId' => $accountId, 'transactionId' => $transactionId, 'status' => $status]);
    }

    /** Maps a transfer object to a local state. Every checkout counts, because split creates several. */
    public static function checkoutOutcome(string $uniqueCode, int $amount, mixed $data): string
    {
        if (! is_array($data) || ($data['uniqueCode'] ?? null) !== $uniqueCode || ($data['totalAmount'] ?? null) !== $amount) {
            return 'needs_review';
        }
        $checkouts = $data['checkouts'] ?? null;
        if (! is_array($checkouts) || $checkouts === [] || ! array_is_list($checkouts)) {
            return 'needs_review';
        }
        $statuses = [];
        foreach ($checkouts as $checkout) {
            $status = is_array($checkout) ? ($checkout['status'] ?? null) : null;
            if (! is_int($status) || ! in_array($status, self::CHECKOUT_STATUSES, true)) {
                return 'needs_review';
            }
            $statuses[] = $status;
        }
        if (array_intersect($statuses, self::PENDING_STATUSES) !== []) {
            return 'processing';
        }
        $all = fn (int $value) => count(array_unique($statuses)) === 1 && $statuses[0] === $value;
        if ($all(3)) {
            foreach ($checkouts as $checkout) {
                if (empty($checkout['refCode'])) {
                    return 'needs_review';
                }
            }

            return 'succeeded';
        }
        if ($all(5)) {
            return 'failed';
        }
        if ($all(4)) {
            return 'returned';
        }

        return 'needs_review';
    }

    public static function maskIban(string $iban): string
    {
        return substr($iban, 0, 4).str_repeat('*', 18).substr($iban, -4);
    }
}
```

### `app/Services/Zibal/Conflict.php`

```php
<?php

namespace App\Services\Zibal;

class Conflict extends \RuntimeException
{
}
```

### `app/Services/Zibal/PayoutService.php`

```php
<?php

namespace App\Services\Zibal;

use App\Models\Payout;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Str;

class PayoutService
{
    // A state can only be entered from these states, so a stale read cannot undo a newer one.
    private const ALLOWED_FROM = [
        'unknown' => ['submitting'],
        'processing' => ['submitting', 'unknown', 'processing', 'needs_review'],
        'succeeded' => ['submitting', 'unknown', 'processing', 'succeeded', 'needs_review'],
        'returned' => ['submitting', 'unknown', 'processing', 'succeeded', 'needs_review'],
        'failed' => ['submitting', 'unknown', 'processing', 'needs_review'],
        'needs_review' => ['submitting', 'unknown', 'processing', 'succeeded', 'needs_review'],
    ];

    private const RECONCILABLE = ['submitting', 'unknown', 'processing', 'succeeded', 'needs_review'];

    public function __construct(private ZibalClient $zibal)
    {
    }

    public function find(string $id): ?Payout
    {
        return Payout::find($id);
    }

    public function view(Payout $payout, array $extra = []): array
    {
        return [
            'id' => $payout->id,
            'requestKey' => $payout->request_key,
            'uniqueCode' => $payout->unique_code,
            'state' => $payout->state,
            'amount' => $payout->amount,
            'iban' => ZibalClient::maskIban($payout->iban),
            'trackerId' => $payout->tracker_id,
            'lastResult' => $payout->last_result,
            'errorGroup' => $payout->error_group,
            'checkouts' => $payout->checkoutList(),
            ...$extra,
        ];
    }

    private function transition(string $id, string $state, array $fields = []): void
    {
        Payout::whereKey($id)
            ->whereIn('state', self::ALLOWED_FROM[$state])
            ->update([...$fields, 'state' => $state, 'updated_at' => microtime(true)]);
    }

    private function checkoutSummary(mixed $data): string
    {
        $checkouts = is_array($data) ? ($data['checkouts'] ?? null) : null;
        if (! is_array($checkouts)) {
            return '[]';
        }
        $rows = [];
        foreach ($checkouts as $checkout) {
            if (is_array($checkout)) {
                $rows[] = [
                    'id' => $checkout['id'] ?? null,
                    'amount' => $checkout['amount'] ?? null,
                    'status' => $checkout['status'] ?? null,
                    'refCode' => $checkout['refCode'] ?? null,
                    'failReason' => $checkout['failReason'] ?? null,
                ];
            }
        }

        return json_encode($rows);
    }

    private function submit(string $id): void
    {
        $now = microtime(true);
        $claimed = Payout::whereKey($id)->where('state', 'created')
            ->update(['state' => 'submitting', 'submitted_at' => $now, 'updated_at' => $now]);
        if ($claimed === 0) {
            return;
        }

        $payout = $this->find($id);
        $body = [
            'amount' => $payout->amount,
            'iban' => $payout->iban,
            'accountId' => $payout->account_id,
            'reasonCode' => $payout->reason_code,
            'uniqueCode' => $payout->unique_code,
        ];
        if ($payout->delay !== null) {
            $body['delay'] = $payout->delay;
        }
        if ($payout->description !== null) {
            $body['description'] = $payout->description;
        }
        if ($payout->payment_number !== null) {
            $body['paymentNumber'] = $payout->payment_number;
        }
        if ($payout->split !== null) {
            $body['split'] = $payout->split;
        }
        if (config('zibal.callback_url')) {
            $body['callbackUrl'] = config('zibal.callback_url');
        }

        $reply = $this->zibal->createCheckout($body);
        $base = ['last_result' => $reply->result(), 'last_message' => $reply->message()];
        $data = $reply->get('data');

        if ($reply->http === 200 && $reply->result() === 1 && is_array($data)) {
            $tracker = $data['trackerId'] ?? null;
            $this->transition($id, ZibalClient::checkoutOutcome($payout->unique_code, $payout->amount, $data), [
                ...$base,
                'tracker_id' => is_string($tracker) ? $tracker : null,
                'checkouts' => $this->checkoutSummary($data),
            ]);
        } elseif (ZibalClient::isRejected($reply)) {
            $this->transition($id, 'failed', [...$base, 'error_group' => ZibalClient::resultGroup($reply->result())]);
        } else {
            $this->transition($id, 'unknown', [...$base, 'error_group' => ZibalClient::resultGroup($reply->result())]);
        }
        Log::info('payout_submitted', ['id' => $id, 'iban' => ZibalClient::maskIban($payout->iban), 'http' => $reply->http, 'result' => $reply->result()]);
    }

    /** @return array{0: Payout, 1: bool} */
    public function create(array $input): array
    {
        $hash = hash('sha256', json_encode([
            $input['amount'], $input['iban'], $input['reasonCode'], $input['delay'] ?? null,
            $input['description'] ?? null, $input['paymentNumber'] ?? null, $input['split'] ?? null,
        ]));
        $now = microtime(true);
        $id = (string) Str::uuid();

        $inserted = DB::table('payouts')->insertOrIgnore([
            'id' => $id,
            'request_key' => $input['requestKey'],
            'request_hash' => $hash,
            'unique_code' => (string) Str::uuid(),
            'account_id' => config('zibal.account_id'),
            'amount' => $input['amount'],
            'iban' => $input['iban'],
            'reason_code' => $input['reasonCode'],
            'delay' => $input['delay'] ?? null,
            'description' => $input['description'] ?? null,
            'payment_number' => $input['paymentNumber'] ?? null,
            'split' => $input['split'] ?? null,
            'state' => 'created',
            'created_at' => $now,
            'updated_at' => $now,
        ]);

        if ($inserted === 0) {
            $existing = Payout::where('request_key', $input['requestKey'])->firstOrFail();
            if (! hash_equals($existing->request_hash, $hash)) {
                throw new Conflict('request_key_reused');
            }
            // 'created' means nothing was sent yet, so the claim in submit() is still safe.
            if ($existing->state === 'created') {
                $this->submit($existing->id);
            }

            return [$this->find($existing->id), true];
        }

        $this->submit($id);

        return [$this->find($id), false];
    }

    public function reconcile(string $id): ?Payout
    {
        $payout = $this->find($id);
        if ($payout === null || ! in_array($payout->state, self::RECONCILABLE, true)) {
            return $payout;
        }

        $now = microtime(true);
        if ($payout->state === 'submitting') {
            // A live request is still inside its timeout. Do not race it.
            if ($now - $payout->updated_at < config('zibal.timeout_s') * 2) {
                return $payout;
            }
            $this->transition($id, 'unknown');
        }

        $reply = $this->zibal->inquireCheckout($payout->unique_code);
        $base = ['last_result' => $reply->result(), 'last_message' => $reply->message()];

        if ($reply->http === 200 && $reply->result() === 1) {
            $data = $reply->get('data');
            $tracker = is_array($data) ? ($data['trackerId'] ?? null) : null;
            $this->transition($id, ZibalClient::checkoutOutcome($payout->unique_code, $payout->amount, $data), [
                ...$base,
                'tracker_id' => is_string($tracker) ? $tracker : $payout->tracker_id,
                'checkouts' => $this->checkoutSummary($data),
                'error_group' => null,
            ]);
        } else {
            $open = in_array($payout->state, ['unknown', 'submitting'], true);
            $expired = $now - ($payout->submitted_at ?? $payout->created_at) >= config('zibal.review_after_s');
            // Not found for a transfer Zibal already accepted is never normal.
            $vanished = $reply->result() === 14 && ! $open && $payout->state !== 'needs_review';
            if (($open && $expired) || $vanished) {
                $this->transition($id, 'needs_review', $base);
            }
        }
        Log::info('payout_reconciled', ['id' => $id, 'http' => $reply->http, 'result' => $reply->result()]);

        return $this->find($id);
    }

    public function reconcileDue(): int
    {
        $now = microtime(true);
        $ids = Payout::query()
            ->where(function ($query) use ($now) {
                $query->whereIn('state', ['unknown', 'processing'])
                    ->orWhere(fn ($q) => $q->where('state', 'submitting')->where('updated_at', '<=', $now - config('zibal.timeout_s') * 2))
                    ->orWhere(fn ($q) => $q->where('state', 'succeeded')->where('submitted_at', '>=', $now - config('zibal.return_watch_s')));
            })
            ->orderBy('updated_at')->limit(50)->pluck('id');
        foreach ($ids as $id) {
            $this->reconcile($id);
        }

        return $ids->count();
    }

    public function cancel(Payout $payout): array
    {
        $ids = [];
        foreach ($payout->checkoutList() as $checkout) {
            if (! empty($checkout['id']) && in_array($checkout['status'] ?? null, ZibalClient::PENDING_STATUSES, true)) {
                $ids[] = $checkout['id'];
            }
        }
        if ($ids === []) {
            throw new Conflict('not_cancellable');
        }

        $claimed = Payout::whereKey($payout->id)->whereNull('cancel_state')->where('state', 'processing')
            ->update(['cancel_state' => 'sent', 'updated_at' => microtime(true)]);
        if ($claimed === 0) {
            throw new Conflict('not_cancellable');
        }

        $reply = $this->zibal->cancelCheckout($payout->account_id, $ids);

        if ($reply->http === 200 && $reply->result() === 1) {
            $data = is_array($reply->get('data')) ? $reply->get('data') : [];

            return [
                'http' => 200,
                'payout' => $this->reconcile($payout->id),
                'cancelOutcome' => 'answered',
                'cancelled' => is_array($data['cancelled_checkouts'] ?? null) ? $data['cancelled_checkouts'] : [],
                'uncancellable' => is_array($data['uncancellable_checkouts'] ?? null) ? $data['uncancellable_checkouts'] : [],
            ];
        }
        if (ZibalClient::isRejected($reply)) {
            Payout::whereKey($payout->id)->update(['cancel_state' => null, 'last_result' => $reply->result(), 'updated_at' => microtime(true)]);

            return ['http' => 422, 'payout' => $this->find($payout->id), 'cancelOutcome' => 'rejected', 'cancelled' => [], 'uncancellable' => []];
        }

        // Unknown: keep cancel_state so it cannot be sent twice. Status comes from inquire.
        return ['http' => 202, 'payout' => $this->reconcile($payout->id), 'cancelOutcome' => 'unknown', 'cancelled' => [], 'uncancellable' => []];
    }
}
```

### `app/Services/Zibal/DepositService.php`

```php
<?php

namespace App\Services\Zibal;

use App\Models\DepositDecision;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Str;

class DepositService
{
    public function __construct(private ZibalClient $zibal)
    {
    }

    public function find(string $transactionId): ?DepositDecision
    {
        return DepositDecision::where('account_id', config('zibal.account_id'))->where('transaction_id', $transactionId)->first();
    }

    public function view(DepositDecision $decision, array $extra = []): array
    {
        return [
            'id' => $decision->id,
            'transactionId' => $decision->transaction_id,
            'status' => $decision->status,
            'state' => $decision->state,
            'refundTrackerId' => $decision->refund_tracker_id,
            'lastResult' => $decision->last_result,
            ...$extra,
        ];
    }

    private function setState(string $id, array $from, string $state, array $fields = []): void
    {
        DepositDecision::whereKey($id)->whereIn('state', $from)
            ->update([...$fields, 'state' => $state, 'updated_at' => microtime(true)]);
    }

    /** Returns the deposit, or null when the answer does not settle anything. */
    private function fetchDeposit(string $transactionId): ?array
    {
        $reply = $this->zibal->inquireDeposit(config('zibal.account_id'), $transactionId);
        $data = $reply->get('data');
        if ($reply->http !== 200 || $reply->result() !== 1 || ! is_array($data) || ! array_is_list($data)) {
            return null;
        }
        foreach ($data as $item) {
            if (is_array($item) && ($item['id'] ?? null) === $transactionId) {
                return $item;
            }
        }

        return null;
    }

    /** @return array{0: int, 1: DepositDecision, 2: bool} */
    public function decide(string $transactionId, int $status): array
    {
        $now = microtime(true);
        $inserted = DB::table('deposit_decisions')->insertOrIgnore([
            'id' => (string) Str::uuid(),
            'account_id' => config('zibal.account_id'),
            'transaction_id' => $transactionId,
            'status' => $status,
            'state' => 'created',
            'created_at' => $now,
            'updated_at' => $now,
        ]);

        $decision = $this->find($transactionId);
        if ($decision->status !== $status) {
            throw new Conflict('different_decision_exists');
        }
        $duplicate = $inserted === 0;

        $claimed = DepositDecision::whereKey($decision->id)->where('state', 'created')
            ->update(['state' => 'submitting', 'updated_at' => microtime(true)]);
        if ($claimed === 0) {
            return [200, $this->find($transactionId), true];
        }

        $deposit = $this->fetchDeposit($transactionId);
        if ($deposit === null) {
            // Nothing was sent, so release the claim and let the caller try again.
            $this->setState($decision->id, ['submitting'], 'created');

            return [503, $this->find($transactionId), $duplicate];
        }
        $current = $deposit['verifyStatus'] ?? null;
        if ($current !== 0) {
            if ($current === $status) {
                $this->setState($decision->id, ['submitting'], 'done');

                return [200, $this->find($transactionId), $duplicate];
            }
            if ($current === 2 || $current === 3) {
                $this->setState($decision->id, ['submitting'], 'failed');

                return [409, $this->find($transactionId), $duplicate];
            }
            $this->setState($decision->id, ['submitting'], 'needs_review');

            return [202, $this->find($transactionId), $duplicate];
        }

        DepositDecision::whereKey($decision->id)->update(['submitted_at' => microtime(true)]);
        $reply = $this->zibal->changeDepositStatus(config('zibal.account_id'), $transactionId, $status);
        $base = ['last_result' => $reply->result(), 'last_message' => $reply->message()];

        if ($reply->http === 200 && $reply->get('success') === true) {
            $data = $reply->get('data');
            $tracker = is_array($data) ? ($data['refundRequestTrackerId'] ?? null) : null;
            $this->setState($decision->id, ['submitting'], 'done', [...$base, 'refund_tracker_id' => is_string($tracker) ? $tracker : null]);
            $http = 200;
        } elseif (ZibalClient::isRejected($reply)) {
            $this->setState($decision->id, ['submitting'], 'failed', $base);
            $http = 422;
        } else {
            $this->setState($decision->id, ['submitting'], 'unknown', $base);
            $http = 202;
        }
        Log::info('deposit_decided', ['id' => $decision->id, 'http' => $reply->http, 'result' => $reply->result()]);

        return [$http, $this->find($transactionId), $duplicate];
    }

    public function reconcile(string $transactionId): ?DepositDecision
    {
        $decision = $this->find($transactionId);
        if ($decision === null) {
            return null;
        }
        if ($decision->state === 'submitting' && microtime(true) - $decision->updated_at >= config('zibal.timeout_s') * 3) {
            // The process died after claiming. The change-status call may or may not have been sent.
            $this->setState($decision->id, ['submitting'], 'unknown');
        } elseif ($decision->state !== 'unknown') {
            return $decision;
        }

        $deposit = $this->fetchDeposit($transactionId);
        if ($deposit !== null && ($deposit['verifyStatus'] ?? null) === $decision->status) {
            $this->setState($decision->id, ['unknown'], 'done');
        } else {
            $age = microtime(true) - ($decision->submitted_at ?? $decision->created_at);
            // A changed status that is not ours, or no change after the window, needs a person.
            $changedElsewhere = $deposit !== null && ($deposit['verifyStatus'] ?? null) !== 0;
            if ($changedElsewhere || $age >= config('zibal.review_after_s')) {
                $this->setState($decision->id, ['unknown'], 'needs_review');
            }
        }

        return $this->find($transactionId);
    }

    public function reconcileDue(): int
    {
        $ids = DepositDecision::query()
            ->where(fn ($query) => $query->where('state', 'unknown')
                ->orWhere(fn ($q) => $q->where('state', 'submitting')->where('updated_at', '<=', microtime(true) - config('zibal.timeout_s') * 3)))
            ->orderBy('updated_at')->limit(50)->pluck('transaction_id');
        foreach ($ids as $transactionId) {
            $this->reconcile($transactionId);
        }

        return $ids->count();
    }
}
```

### `app/Http/Middleware/InternalKey.php`

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

/** Placeholder for your own authentication and authorization. */
class InternalKey
{
    public function handle(Request $request, Closure $next): Response
    {
        $given = (string) $request->header('X-Internal-Key', '');
        $expected = (string) config('zibal.internal_key');
        if ($expected === '' || ! hash_equals(hash('sha256', $expected), hash('sha256', $given))) {
            return response()->json(['error' => 'unauthorized'], 401);
        }

        return $next($request);
    }
}
```

### `app/Http/Controllers/EbankController.php`

```php
<?php

namespace App\Http\Controllers;

use App\Models\Payout;
use App\Services\Zibal\Conflict;
use App\Services\Zibal\DepositService;
use App\Services\Zibal\PayoutService;
use App\Services\Zibal\ZibalClient;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class EbankController extends Controller
{
    private const MAX_BODY_BYTES = 16384;

    // The docs allow int64. This app caps at 2^53 - 1 so JavaScript clients read amounts exactly.
    private const MAX_AMOUNT = 9007199254740991;

    private const PAYOUT_FIELDS = ['requestKey', 'amount', 'iban', 'reasonCode', 'delay', 'description', 'paymentNumber', 'split'];

    private const PAYOUT_REQUIRED = ['requestKey', 'amount', 'iban', 'reasonCode'];

    public function __construct(private PayoutService $payouts, private DepositService $deposits, private ZibalClient $zibal)
    {
    }

    /**
     * Reads the raw body. Laravel's TrimStrings and ConvertEmptyStringsToNull middleware
     * rewrite $request->input(), which would quietly accept " IR..." or "".
     */
    private function json(Request $request, int $limit = self::MAX_BODY_BYTES): ?array
    {
        if (! str_starts_with((string) $request->header('Content-Type'), 'application/json')) {
            return null;
        }
        $raw = $request->getContent();
        if (strlen($raw) > $limit) {
            return null;
        }
        try {
            $body = json_decode($raw, true, 32, JSON_BIGINT_AS_STRING | JSON_THROW_ON_ERROR);
        } catch (\JsonException) {
            return null;
        }

        return is_array($body) && ($body === [] || ! array_is_list($body)) ? $body : null;
    }

    private static function intBetween(mixed $value, int $low, int $high): bool
    {
        return is_int($value) && $value >= $low && $value <= $high;
    }

    private static function text(mixed $value, int $max): bool
    {
        return is_string($value) && mb_strlen($value) <= $max && ! preg_match('/[\x00-\x1f\x7f]/', $value);
    }

    /** amount, iban and delay limits come from the docs. Other caps are this app's own choice. */
    private function parsePayout(?array $body): ?array
    {
        if ($body === null || array_diff(array_keys($body), self::PAYOUT_FIELDS) !== [] || array_diff(self::PAYOUT_REQUIRED, array_keys($body)) !== []) {
            return null;
        }
        // \A and \z, because $ also matches before a trailing newline. [0-9] does not match Persian digits.
        $valid = is_string($body['requestKey']) && preg_match('/\A[A-Za-z0-9_-]{1,64}\z/', $body['requestKey'])
            && self::intBetween($body['amount'], 10000, self::MAX_AMOUNT)
            && is_string($body['iban']) && preg_match('/\AIR[0-9]{24}\z/', $body['iban'])
            && self::intBetween($body['reasonCode'], 1, 19)
            && (! array_key_exists('delay', $body) || self::intBetween($body['delay'], -1, 10080))
            && (! array_key_exists('description', $body) || self::text($body['description'], 255))
            && (! array_key_exists('paymentNumber', $body) || self::text($body['paymentNumber'], 64))
            && (! array_key_exists('split', $body) || is_bool($body['split']));

        return $valid ? $body : null;
    }

    private static function httpStatus(string $state): int
    {
        return match ($state) {
            'failed' => 422,
            'unknown', 'needs_review', 'submitting' => 202,
            default => 201,
        };
    }

    private static function error(string $error, int $status): JsonResponse
    {
        return response()->json(['error' => $error], $status);
    }

    public function createPayout(Request $request): JsonResponse
    {
        $input = $this->parsePayout($this->json($request));
        if ($input === null) {
            return self::error('invalid_request', 400);
        }
        try {
            [$payout, $duplicate] = $this->payouts->create($input);
        } catch (Conflict $conflict) {
            return self::error($conflict->getMessage(), 409);
        }

        return response()->json($this->payouts->view($payout, ['duplicate' => $duplicate]), $duplicate ? 200 : self::httpStatus($payout->state));
    }

    public function readPayout(string $id): JsonResponse
    {
        $payout = $this->payouts->find($id);

        return $payout ? response()->json($this->payouts->view($payout)) : self::error('not_found', 404);
    }

    public function reconcilePayout(string $id): JsonResponse
    {
        $payout = $this->payouts->reconcile($id);

        return $payout ? response()->json($this->payouts->view($payout)) : self::error('not_found', 404);
    }

    public function cancelPayout(string $id): JsonResponse
    {
        $payout = $this->payouts->find($id);
        if ($payout === null) {
            return self::error('not_found', 404);
        }
        try {
            $outcome = $this->payouts->cancel($payout);
        } catch (Conflict $conflict) {
            return self::error($conflict->getMessage(), 409);
        }

        return response()->json($this->payouts->view($outcome['payout'], [
            'cancelOutcome' => $outcome['cancelOutcome'],
            'cancelled' => $outcome['cancelled'],
            'uncancellable' => $outcome['uncancellable'],
        ]), $outcome['http']);
    }

    public function reconcileAll(): JsonResponse
    {
        return response()->json(['processed' => $this->payouts->reconcileDue() + $this->deposits->reconcileDue()]);
    }

    public function decideDeposit(Request $request, string $transactionId): JsonResponse
    {
        $body = $this->json($request);
        $valid = preg_match('/\A[A-Za-z0-9_-]{1,64}\z/', $transactionId)
            && $body !== null && array_keys($body) === ['status'] && self::intBetween($body['status'], 2, 3);
        if (! $valid) {
            return self::error('invalid_request', 400);
        }
        try {
            [$http, $decision, $duplicate] = $this->deposits->decide($transactionId, $body['status']);
        } catch (Conflict $conflict) {
            return self::error($conflict->getMessage(), 409);
        }

        return response()->json($this->deposits->view($decision, ['duplicate' => $duplicate]), $http);
    }

    public function reconcileDeposit(string $transactionId): JsonResponse
    {
        $decision = $this->deposits->reconcile($transactionId);

        return $decision ? response()->json($this->deposits->view($decision)) : self::error('not_found', 404);
    }

    public function balance(): JsonResponse
    {
        $reply = $this->zibal->balance(config('zibal.account_id'));
        if ($reply->http === null) {
            return self::error('zibal_unreachable', 504);
        }
        // Balance has no result field on success.
        if ($reply->http === 200 && is_int($reply->get('balance'))) {
            return response()->json(['balance' => $reply->get('balance'), 'balanceUpdatedAt' => $reply->get('balanceUpdatedAt')]);
        }

        return response()->json([
            'error' => 'zibal_error',
            'lastResult' => $reply->result(),
            'errorGroup' => ZibalClient::resultGroup($reply->result()),
        ], 502);
    }

    /** Public on purpose. Only the unique code is read, and the state comes from inquire. */
    public function webhook(Request $request): JsonResponse
    {
        $body = $this->json($request, 65536);
        $data = is_array($body['data'] ?? null) ? $body['data'] : null;
        $code = $data['unique_code'] ?? $data['uniqueCode'] ?? null;
        if (is_string($code) && strlen($code) <= 64) {
            $id = Payout::where('unique_code', $code)->value('id');
            if ($id !== null) {
                $this->payouts->reconcile($id);
            }
        }

        return response()->json(['ok' => true]);
    }
}
```

## Run

```bash
composer create-project laravel/laravel ebank
cd ebank
# add the files above, append .env.additions to .env, then:
php artisan migrate
php artisan serve
```

For SQLite with several workers, set `'busy_timeout' => 5000` and `'journal_mode' => 'wal'` on the `sqlite` connection in `config/database.php`.

Test against a local mock by setting `ZIBAL_BASE_URL`. There is no sandbox, so a real token would act on real accounts.

## Framework notes

- Routes in `routes/api.php` are outside the `web` group, so `ValidateCsrfToken` does not block the webhook. Do not move the webhook into `routes/web.php`.
- `TrimStrings` and `ConvertEmptyStringsToNull` rewrite `$request->input()`. A trimmed IBAN would pass validation without being what the client sent, so the controller reads the raw body.
- Laravel's `integer` validation rule accepts `"10000"`. The controller uses `is_int` instead.
- `json_decode` turns large integers into floats. `JSON_BIGINT_AS_STRING` keeps them as strings, so they fail `is_int`.
- In PHP, `$` matches before a trailing newline, so the patterns use `\A` and `\z`.
- The `Http` client follows redirects by default. `withoutRedirecting()` turns that off. Never add `->retry()` to these calls.
- `php artisan serve` passes only a short allowlist of environment variables to its workers. Keep configuration in `.env` when using it.
- After `php artisan config:cache`, `env()` works only inside config files. All reads here go through `config('zibal.*')`.
- Run the scheduler with `withoutOverlapping()`, as `routes/console.php` does.
