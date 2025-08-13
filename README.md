# Midtrans-API-Documentation  
**A complete Midtrans API integration guide with Laravel 12 and Ngrok** by **Cupcake-Legend** for personal documentation and future projects.

## 1. Prerequisites

* PHP 8.1+
* Laravel 11 +
* Herd
* Composer
* Midtrans Account ([Sign Up](https://dashboard.midtrans.com/register))
* Ngrok account ([Sign Up](https://ngrok.com))
* MySQL or other database
* Postman or `curl` for testing

---

## 2. Install Midtrans PHP SDK

```bash
composer require midtrans/midtrans-php
```

---
## 3. Configure `.env`

In your Laravel `.env` file, add:

```env
MIDTRANS_SERVER_KEY=your-server-key
MIDTRANS_CLIENT_KEY=your-client-key
MIDTRANS_IS_PRODUCTION=false
```

You can find these keys in the [Midtrans Dashboard](https://dashboard.midtrans.com/) under **Settings → Access Keys**.

---

## 4. Publish Midtrans Configuration

Create a configuration file at `config/midtrans.php` with the following content:

```php
<?php

return [
    'server_key'    => env('MIDTRANS_SERVER_KEY', ''),
    'client_key'    => env('MIDTRANS_CLIENT_KEY', ''),
    'is_production' => env('MIDTRANS_IS_PRODUCTION', false),
    'is_sanitized'  => true,
    'is_3ds'        => true,
];
```

This config file will help you easily manage Midtrans settings within your Laravel app.

---

## 5. Create Callback Route

In Laravel 11 and above, `api.php` is not included by default. To enable API routes:

```bash
php artisan install:api
```

This will create the `routes/api.php` file.
Open it and add your Midtrans callback route:

```php
use App\Http\Controllers\MidtransController;

Route::post('/midtrans/callback', [MidtransController::class, 'callback']);
```

---

## 6. Register API Routes in `bootstrap/app.php`

To enable your API routes, register the `api.php` file in `bootstrap/app.php` by adding it to the routing configuration:

```php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__ . '/../routes/web.php',
        api: __DIR__ . '/../routes/api.php',  // add this line
        commands: __DIR__ . '/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void {})
    ->withExceptions(function (Exceptions $exceptions): void {})
    ->create();
```

This makes sure your API routes are loaded and accessible.

---

## 7. Create Controller

Create `app/Http/Controllers/MidtransController.php`:  

Example Logic:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Auth;
use App\Models\Transaction;
use Exception;

class MidtransController extends Controller
{
    public function callback(Request $request)
    {
        try {
            Log::info('Midtrans callback received:', $request->all());

            $serverKey = env('MIDTRANS_SERVER_KEY');
            $hashed = hash(
                "sha512",
                $request->order_id . $request->status_code . $request->gross_amount . $serverKey
            );

            if ($hashed !== $request->signature_key) {
                Log::warning('Invalid signature for order_id: ' . $request->order_id);
                return response()->json(['message' => 'Invalid signature'], 403);
            }

            // Find transaction by order_id
            $transaction = Transaction::where('order_id', $request->order_id)->first();

            if (!$transaction) {
                $transaction = Transaction::create([
                    'order_id' => $request->order_id,
                    'payment_type' => $request->payment_type,
                    'status' => $request->transaction_status,
                    'amount' => (int) $request->gross_amount,
                    'midtrans_payload' => json_encode($request->all())
                ]);
            } else {
                $transaction->update([
                    'status' => $request->transaction_status,
                    'payment_type' => $request->payment_type,
                    'midtrans_payload' => json_encode($request->all())
                ]);
            }

            return response()->json(['message' => 'OK'], 200);
        } catch (\Throwable $e) {
            Log::error('Midtrans callback error: ' . $e->getMessage(), [
                'trace' => $e->getTraceAsString(),
                'payload' => $request->all()
            ]);

            return response()->json([
                'message' => 'Internal Server Error',
                'error' => $e->getMessage()
            ], 500);
        }
    }
}
```

---

## 8. Example Midtrans Notification Payload

When Midtrans sends a payment notification, it sends JSON data like this:

```json
{
  "transaction_time": "2023-11-15 18:45:13",
  "transaction_status": "settlement",
  "transaction_id": "513xxxxx-c9da-474c-9fc9-d5c6436xxxxx",
  "status_message": "midtrans payment notification",
  "status_code": "200",
  "signature_key": "c88f4ce4afxxxx...........",
  "settlement_time": "2023-11-15 22:45:13",
  "payment_type": "gopay",
  "order_id": "payment_notif_test_XXXXXX.....",
  "merchant_id": "GXXXXXXXXX",
  "gross_amount": "105000.00",
  "fraud_status": "accept",
  "currency": "IDR"
}
```

---

## 9. Transaction Migration & Model

Based on the above data, your transactions table should store relevant fields like:

```php
Schema::create('transactions', function (Blueprint $table) {
    $table->id();
    $table->string('order_id')->unique();
    $table->string('payment_type')->nullable();
    $table->enum('status', ['capture', 'settlement', 'pending', 'deny', 'expire', 'cancel'])->default('pending');
    $table->integer('amount')->nullable();
    $table->json('midtrans_payload')->nullable();
    $table->timestamps();
});
```

Model:

```php
class Transaction extends Model
{
    protected $fillable = [
        'order_id',
        'payment_type',
        'status',
        'amount',
        'midtrans_payload',
    ];
}
```
---

## 10. Expose Localhost with Ngrok

If you’re running Laravel using **Herd** with a custom local domain like `http://midtrans-api.test`, you can expose it with ngrok by specifying the full URL and additional options.

Instead of the usual:

```bash
ngrok http 80
```

Use this command with Herd:

```bash
ngrok http 80 --host-header=midtrans-api.test
```

* `--host-header=midtrans-api.test` rewrites the HTTP Host header so ngrok forwards requests correctly to Herd’s custom domain.

After running the command, you’ll get a public URL like:

```
https://fc262999bcb5.ngrok-free.app
```


## 11. Set Midtrans Notification URL

In the [Midtrans Dashboard](https://dashboard.midtrans.com/):

```
Notification URL: https://your-ngrok-id.ngrok-free.app/api/midtrans/callback
```

---

## 12. Testing with cURL

### Linux / macOS (bash, zsh)

You can use the multiline command with backslashes (`\`):

```bash
curl -X POST "https://fc262999bcb5.ngrok-free.app/api/midtrans/callback" \
-H "Content-Type: application/json" \
-d '{"order_id":"ORDER-101","status_code":"200","gross_amount":"10000.00","signature_key":"<your_signature_key>","payment_type":"bank_transfer","transaction_status":"settlement"}'
```

---

### Windows CMD

In Windows Command Prompt, write the entire command in one line without backslashes, and escape inner quotes with backslashes (`\"`):

```cmd
curl -X POST "https://fc262999bcb5.ngrok-free.app/api/midtrans/callback" -H "Content-Type: application/json" -d "{\"order_id\":\"ORDER-101\",\"status_code\":\"200\",\"gross_amount\":\"10000.00\",\"signature_key\":\"<your_signature_key>\",\"payment_type\":\"bank_transfer\",\"transaction_status\":\"settlement\"}"
```

---

### Windows PowerShell

In PowerShell, you can use single quotes outside and double quotes inside, or use backticks for multiline:

```powershell
curl -X POST "https://fc262999bcb5.ngrok-free.app/api/midtrans/callback" `
-H "Content-Type: application/json" `
-d '{"order_id":"ORDER-101","status_code":"200","gross_amount":"10000.00","signature_key":"<your_signature_key>","payment_type":"bank_transfer","transaction_status":"settlement"}'
```

---

This helps avoid common pitfalls depending on your terminal environment.

💡 **Signature key formula**:

```
SHA512(order_id + status_code + gross_amount + server_key)
```

---

## 13. Logs

Check Laravel logs:

```bash
tail -f storage/logs/laravel.log
```

You should see:

```
[INFO] Midtrans callback received: {...}
```

---

## 14. Common Issues

* **Invalid signature** → Make sure you generate `signature_key` exactly as Midtrans specifies.
* **500 Internal Server Error** → Check Laravel logs for exception messages.
* **Ngrok tunnel closed** → Keep ngrok running in a separate terminal while testing.

---
