# Slack Error Notifications in This App: How It Works

- **What it does**: Whenever a reportable exception occurs, the app formats detailed information — including the environment, message, file and line, request info, user, and stack trace — then sends it to a Slack channel via an incoming webhook. It also applies rate limiting to prevent spam.

- **The flow**
  - An exception occurs.
  - `App\Exceptions\Handler::report` runs and calls `SlackNotificationService::sendExceptionNotification`.
  - The service checks configuration, ignores specified exception types, enforces rate limits, builds a Slack Block Kit payload, and posts it to Slack.

---

## 1) Configuration (`config/slack.php`)

Key toggles and connection settings:

```php
'enabled' => env('SLACK_NOTIFICATIONS_ENABLED', false),
'webhook_url' => env('SLACK_WEBHOOK_URL'),
'channel' => env('SLACK_CHANNEL', '#errors'),
'username' => env('SLACK_USERNAME', 'Maqraa Error Bot'),
'icon_emoji' => env('SLACK_ICON_EMOJI', ':rotating_light:'),
```

Exceptions to ignore:

```php
'ignored_exceptions' => [
    \Illuminate\Auth\AuthenticationException::class,
    \Illuminate\Validation\ValidationException::class,
    \Symfony\Component\HttpKernel\Exception\NotFoundHttpException::class,
],
```

Built-in rate limiting:

```php
'rate_limit' => [
    'enabled' => env('SLACK_RATE_LIMIT_ENABLED', true),
    'max_notifications' => env('SLACK_RATE_LIMIT_MAX', 10),
    'per_minutes' => env('SLACK_RATE_LIMIT_MINUTES', 5),
],
```

Notes:
- `notify_all_exceptions` determines whether all exceptions (except the ignored list) are notified, or only those listed in `notifiable_exceptions`.
- `colors` and `color_mapping` define the attachment color based on exception type or HTTP status.

---

## 2) The Service (`app/Services/SlackNotificationService.php`)

Entry point: verifies configuration, checks rate limits, builds, and posts the payload.

```php
public function sendExceptionNotification(Throwable $exception, $request = null)
{
    if (!$this->shouldSendNotification($exception)) {
        return false;
    }

    if ($this->isRateLimited()) {
        return false;
    }

    $payload = $this->buildPayload($exception, $request);

    try {
        $response = Http::post($this->config['webhook_url'], $payload);
        
        if ($response->successful()) {
            $this->incrementRateLimit();
            return true;
        }
    } catch (\Exception $e) {
        \Log::error('Failed to send Slack notification: ' . $e->getMessage());
    }

    return false;
}
```

When to send (feature flag, webhook check, ignore/allow lists):

```php
protected function shouldSendNotification(Throwable $exception): bool
{
    if (!$this->config['enabled']) {
        return false;
    }

    if (empty($this->config['webhook_url'])) {
        return false;
    }

    $exceptionClass = get_class($exception);

    if (in_array($exceptionClass, $this->config['ignored_exceptions'])) {
        return false;
    }

    if (!$this->config['notify_all_exceptions']) {
        if (empty($this->config['notifiable_exceptions'])) {
            return false;
        }
        return in_array($exceptionClass, $this->config['notifiable_exceptions']);
    }

    return true;
}
```

Rate limiting (count per period stored in cache):

```php
protected function isRateLimited(): bool
{
    if (!$this->config['rate_limit']['enabled']) {
        return false;
    }

    $key = 'slack_notifications_count';
    $count = Cache::get($key, 0);
    $max = $this->config['rate_limit']['max_notifications'];

    return $count >= $max;
}

protected function incrementRateLimit(): void
{
    if (!$this->config['rate_limit']['enabled']) {
        return;
    }

    $key = 'slack_notifications_count';
    $minutes = $this->config['rate_limit']['per_minutes'];
    
    $count = Cache::get($key, 0);
    Cache::put($key, $count + 1, now()->addMinutes($minutes));
}
```

Payload highlights (Slack Block Kit, color-coded):

```php
if ($this->config['include_stack_trace']) {
    $blocks[] = $this->buildStackTraceBlock($exception);
}

$color = $this->getExceptionColor($exception);

return [
    'channel' => $this->config['channel'],
    'username' => $this->config['username'],
    'icon_emoji' => $this->config['icon_emoji'],
    'attachments' => [
        [
            'color' => $color,
            'blocks' => $blocks,
            'fallback' => class_basename($exception) . ': ' . $exception->getMessage(),
        ]
    ]
];
```

Request summary (safe metadata only, no body):

```php
protected function buildRequestBlock($request): array
{
    $fields = [
        [
            'type' => 'mrkdwn',
            'text' => "*Method:*\n" . $request->method()
        ],
        [
            'type' => 'mrkdwn',
            'text' => "*URL:*\n" . $request->fullUrl()
        ]
    ];
}
```

Stack trace (safely truncated):

```php
protected function buildStackTraceBlock(Throwable $exception): array
{
    $trace = $exception->getTraceAsString();
    $truncatedTrace = substr($trace, 0, 2500);
    
    if (strlen($trace) > 2500) {
        $truncatedTrace .= "\n... (truncated)";
    }

    return [
        'type' => 'section',
        'text' => [
            'type' => 'mrkdwn',
            'text' => "*Stack Trace:*\n```" . $truncatedTrace . "```"
        ]
    ];
}
```

Color selection:
- Uses `color_mapping` for specific exception types.
- Maps HTTP exceptions: 5xx → critical, 4xx → warning, else → default.

---

## 3) Integration (`app/Exceptions/Handler.php`)

When an exception is reported, it dispatches to the Slack service; failures are logged silently:

```php
public function report(Throwable $exception)
{
    if ($this->shouldReport($exception)) {
        try {
            $slackService = app(SlackNotificationService::class);
            $slackService->sendExceptionNotification($exception, request());
        } catch (\Exception $e) {
            \Log::error('Failed to send Slack notification: ' . $e->getMessage());
        }
    }

    parent::report($exception);
}
```

API responses remain JSON-friendly for validation/auth errors (default Laravel behavior):

```php
public function render($request, Throwable $exception)
{
    if ($request->expectsJson() || $request->is('api/*')) {

        if ($exception instanceof ValidationException) {
            return api()->error(
                $exception->getMessage(),
                $exception->validator->errors()->getMessages(),
                422
            );
        }

        if ($exception instanceof AuthenticationException) {
            return api()->unauthenticated();
        }
    }

    return parent::render($request, $exception);
}
```

---

## How to Enable and Use

- **Set environment variables**
  - `SLACK_NOTIFICATIONS_ENABLED=true`
  - `SLACK_WEBHOOK_URL=your_webhook`
  - Optional: `SLACK_CHANNEL=#errors`, `SLACK_USERNAME=Maqraa Error Bot`, `SLACK_ICON_EMOJI=:rotating_light:`
  - Optional: rate limit and include flags (`SLACK_INCLUDE_*`, `SLACK_RATE_LIMIT_*`)
- **Cache**: Ensure your cache store is configured; rate limiting depends on it.
- **Production only**: Keep `enabled=false` in local/testing environments to avoid noise.

---

## Customization Tips

- **Silence noisy exceptions**: Add classes to `ignored_exceptions`.
- **Alert only specific types**: Set `notify_all_exceptions=false` and define `notifiable_exceptions`.
- **Tune volume**: Adjust `max_notifications` and `per_minutes`.
- **Control verbosity**: Toggle `include_request_data`, `include_user_data`, `include_stack_trace`.
- **Customize colors**: Update `colors` and `color_mapping`.

---

## Troubleshooting

- **No messages**: Check `SLACK_NOTIFICATIONS_ENABLED`, `SLACK_WEBHOOK_URL`, network connectivity, and logs for “Failed to send Slack notification”.
- **Too many messages**: Lower `SLACK_RATE_LIMIT_MAX` or expand `ignored_exceptions`.
- **Missing details**: Verify inclusion flags and ensure the user is authenticated (`Auth::check()`).

If desired, you can extend it with level-based filtering (e.g., only 5xx or “critical” exceptions) or enrich the request block with selected input data while redacting sensitive fields.
