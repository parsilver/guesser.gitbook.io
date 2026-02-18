---
description: รวมโค้ดที่เห็นแล้วกุมขมับ — พร้อมวิธีแก้ทีละ step ไม่ต้อง rewrite ทั้งหมด
---

# โค้ดสยองขวัญ

> บทความนี้รวบรวมโค้ดที่เจอบ่อยใน legacy project พร้อมวิธีแก้ ทุกตัวอย่างมาจากประสบการณ์จริง (ปรับให้อ่านง่ายขึ้น)

---

## 1. The God Controller — 1,200 บรรทัดใน method เดียว

### โค้ดสยองขวัญ

```php
class OrderController extends Controller
{
    public function store(Request $request)
    {
        // ---- validation 80 บรรทัด ----
        if (!$request->has('product_id')) {
            return response()->json(['error' => 'product_id is required'], 400);
        }
        if (!$request->has('quantity')) {
            return response()->json(['error' => 'quantity is required'], 400);
        }
        if ($request->quantity < 1) {
            return response()->json(['error' => 'quantity must be positive'], 400);
        }
        if ($request->quantity > 100) {
            return response()->json(['error' => 'max 100 items'], 400);
        }
        $product = DB::table('products')->where('id', $request->product_id)->first();
        if (!$product) {
            return response()->json(['error' => 'product not found'], 404);
        }
        if ($product->stock < $request->quantity) {
            return response()->json(['error' => 'out of stock'], 400);
        }
        if ($product->status !== 'active') {
            return response()->json(['error' => 'product discontinued'], 400);
        }
        $user = DB::table('users')->where('id', $request->user_id)->first();
        if (!$user) {
            return response()->json(['error' => 'user not found'], 404);
        }
        if ($user->is_banned) {
            return response()->json(['error' => 'user banned'], 403);
        }
        // ...validation อีก 50 บรรทัด...

        // ---- คำนวณราคา 60 บรรทัด ----
        $price = $product->price;
        $discount = 0;
        if ($request->has('coupon')) {
            $coupon = DB::table('coupons')->where('code', $request->coupon)->first();
            if ($coupon && $coupon->expires_at > now()) {
                if ($coupon->type === 'percent') {
                    $discount = $price * $request->quantity * ($coupon->value / 100);
                } elseif ($coupon->type === 'fixed') {
                    $discount = $coupon->value;
                } elseif ($coupon->type === 'buy_one_get_one') {
                    if ($request->quantity >= 2) {
                        $discount = $price * floor($request->quantity / 2);
                    }
                }
                if ($coupon->max_discount && $discount > $coupon->max_discount) {
                    $discount = $coupon->max_discount;
                }
                if ($coupon->min_order && ($price * $request->quantity) < $coupon->min_order) {
                    $discount = 0;
                }
            }
        }
        $subtotal = ($price * $request->quantity) - $discount;
        $vat = $subtotal * 0.07;
        $shipping = 0;
        if ($subtotal < 500) {
            $shipping = 50;
        } elseif ($subtotal < 1000) {
            $shipping = 30;
        }
        if ($user->is_premium) {
            $shipping = 0;
        }
        $total = $subtotal + $vat + $shipping;

        // ---- สร้าง order 40 บรรทัด ----
        $orderId = DB::table('orders')->insertGetId([
            'user_id' => $user->id,
            'subtotal' => $subtotal,
            'discount' => $discount,
            'vat' => $vat,
            'shipping' => $shipping,
            'total' => $total,
            'status' => 'pending',
            'created_at' => now(),
        ]);

        DB::table('order_items')->insert([
            'order_id' => $orderId,
            'product_id' => $product->id,
            'quantity' => $request->quantity,
            'price' => $price,
        ]);

        DB::table('products')->where('id', $product->id)->decrement('stock', $request->quantity);

        // ---- ชำระเงิน 50 บรรทัด ----
        if ($request->payment_method === 'credit_card') {
            $ch = curl_init('https://api.payment.com/charge');
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode([
                'amount' => $total * 100,
                'currency' => 'thb',
                'card_token' => $request->card_token,
            ]));
            curl_setopt($ch, CURLOPT_HTTPHEADER, [
                'Authorization: Bearer sk_live_HARDCODED_KEY_12345',
                'Content-Type: application/json',
            ]);
            curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
            $result = curl_exec($ch);
            $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
            curl_close($ch);

            if ($httpCode !== 200) {
                DB::table('orders')->where('id', $orderId)->update(['status' => 'failed']);
                DB::table('products')->where('id', $product->id)->increment('stock', $request->quantity);
                return response()->json(['error' => 'payment failed'], 500);
            }

            $paymentData = json_decode($result);
            DB::table('orders')->where('id', $orderId)->update([
                'status' => 'paid',
                'payment_id' => $paymentData->id,
            ]);
        } elseif ($request->payment_method === 'bank_transfer') {
            // ...อีก 30 บรรทัด...
        } elseif ($request->payment_method === 'promptpay') {
            // ...อีก 40 บรรทัด...
        }

        // ---- ส่ง notification 40 บรรทัด ----
        $emailBody = "<h1>Order #{$orderId}</h1>";
        $emailBody .= "<p>Total: {$total} THB</p>";
        $emailBody .= "<p>Product: {$product->name} x {$request->quantity}</p>";
        mail($user->email, "Order #{$orderId} Confirmed", $emailBody, "Content-Type: text/html");

        // ส่ง Line
        $ch2 = curl_init('https://api.line.me/v2/bot/message/push');
        // ...อีก 15 บรรทัด...

        // ส่ง SMS
        $ch3 = curl_init('https://api.sms.com/send');
        // ...อีก 15 บรรทัด...

        // ---- log + analytics 20 บรรทัด ----
        DB::table('activity_logs')->insert([
            'user_id' => $user->id,
            'action' => 'create_order',
            'data' => json_encode(['order_id' => $orderId]),
            'created_at' => now(),
        ]);

        file_put_contents(
            '/var/log/orders.log',
            date('Y-m-d H:i:s') . " Order #{$orderId} created\n",
            FILE_APPEND
        );

        return response()->json(['order_id' => $orderId, 'total' => $total]);
    }
}
```

ลองนับดูครับ — validation, pricing, database, payment API (พร้อม **hardcode API key**), email, Line, SMS, logging **ทั้งหมดอยู่ใน method เดียว** ประมาณ 300+ บรรทัด และนี่ยังย่อมาแล้ว ของจริงเคยเจอ 1,200 บรรทัด

### ปัญหา

* แก้ logic คิดราคา → อาจพัง payment
* เพิ่ม payment method ใหม่ → ต้องแก้ controller ยาวๆ นี้
* test ไม่ได้เลย — ต้อง mock ทุกอย่างบนโลก
* **API key hardcode อยู่ในโค้ด** 💀
* ไม่มี transaction — ถ้า payment สำเร็จแต่ส่ง email พัง order ค้าง

### วิธีแก้ทีละ step

ไม่ต้อง refactor ทีเดียว ทำทีละชิ้นครับ:

```php
// Step 1: แยก validation ออก
class StoreOrderRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'product_id' => ['required', 'exists:products,id'],
            'quantity' => ['required', 'integer', 'min:1', 'max:100'],
            'user_id' => ['required', 'exists:users,id'],
            'payment_method' => ['required', 'in:credit_card,bank_transfer,promptpay'],
            'coupon' => ['nullable', 'string'],
        ];
    }
}

// Step 2: แยก pricing ออก
class PricingService
{
    public function calculate(Product $product, int $quantity, ?string $couponCode): PriceBreakdown
    {
        $subtotal = $product->price * $quantity;
        $discount = $this->calculateDiscount($couponCode, $subtotal, $product->price, $quantity);
        $vat = ($subtotal - $discount) * 0.07;
        $shipping = $this->calculateShipping($subtotal - $discount);

        return new PriceBreakdown($subtotal, $discount, $vat, $shipping);
    }
}

// Step 3: แยก payment ออก (ใช้ Strategy Pattern)
interface PaymentGateway
{
    public function charge(float $amount, array $options): PaymentResult;
}

// Step 4: controller เหลือแค่ประสาน
class OrderController extends Controller
{
    public function store(
        StoreOrderRequest $request,
        OrderService $orderService,
    ) {
        $order = $orderService->create($request->validated());

        return response()->json($order, 201);
    }
}
```

---

## 2. The Copy-Paste Nightmare — โค้ดเหมือนกัน 15 ที่

### โค้ดสยองขวัญ

```php
class ReportController extends Controller
{
    public function dailySales(Request $request)
    {
        $from = $request->input('from', date('Y-m-d'));
        $to = $request->input('to', date('Y-m-d'));

        $results = DB::select("
            SELECT
                DATE(created_at) as date,
                COUNT(*) as total_orders,
                SUM(total) as revenue,
                AVG(total) as avg_order
            FROM orders
            WHERE status = 'completed'
            AND created_at >= '{$from} 00:00:00'
            AND created_at <= '{$to} 23:59:59'
            GROUP BY DATE(created_at)
            ORDER BY date DESC
        ");

        $data = [];
        foreach ($results as $row) {
            $data[] = [
                'date' => $row->date,
                'total_orders' => number_format($row->total_orders),
                'revenue' => number_format($row->revenue, 2),
                'avg_order' => number_format($row->avg_order, 2),
            ];
        }

        return response()->json(['data' => $data]);
    }

    public function weeklySales(Request $request)
    {
        $from = $request->input('from', date('Y-m-d', strtotime('-7 days')));
        $to = $request->input('to', date('Y-m-d'));

        $results = DB::select("
            SELECT
                YEARWEEK(created_at) as week,
                COUNT(*) as total_orders,
                SUM(total) as revenue,
                AVG(total) as avg_order
            FROM orders
            WHERE status = 'completed'
            AND created_at >= '{$from} 00:00:00'
            AND created_at <= '{$to} 23:59:59'
            GROUP BY YEARWEEK(created_at)
            ORDER BY week DESC
        ");

        $data = [];
        foreach ($results as $row) {
            $data[] = [
                'week' => $row->week,
                'total_orders' => number_format($row->total_orders),
                'revenue' => number_format($row->revenue, 2),
                'avg_order' => number_format($row->avg_order, 2),
            ];
        }

        return response()->json(['data' => $data]);
    }

    public function monthlySales(Request $request)
    {
        // 🙈 เดาออกเลยว่าหน้าตาเป็นยังไง...
        // copy จาก dailySales แล้วเปลี่ยนแค่ GROUP BY
    }

    public function yearlySales(Request $request)
    {
        // 🙈 copy อีกแล้ว...
    }
}
```

สังเกตนะครับ **SQL injection** ด้วย — `$from` กับ `$to` ไม่ได้ bind parameter เลย ใส่ `'; DROP TABLE orders; --` เข้าไปจบเกม

### ปัญหา

* โค้ด **copy-paste 4 ที่** ต่างกันแค่ GROUP BY
* **SQL Injection** — ไม่ได้ใช้ prepared statement
* แก้ format ที → ต้องแก้ 4 method
* เพิ่ม filter ใหม่ (เช่น by category) → ต้องแก้ 4 method

### วิธีแก้

```php
class SalesReportService
{
    public function generate(string $groupBy, string $from, string $to): array
    {
        $groupExpression = match ($groupBy) {
            'daily' => 'DATE(created_at)',
            'weekly' => 'YEARWEEK(created_at)',
            'monthly' => "DATE_FORMAT(created_at, '%Y-%m')",
            'yearly' => 'YEAR(created_at)',
            default => throw new \InvalidArgumentException("Invalid group: {$groupBy}"),
        };

        $results = DB::table('orders')
            ->select(
                DB::raw("{$groupExpression} as period"),
                DB::raw('COUNT(*) as total_orders'),
                DB::raw('SUM(total) as revenue'),
                DB::raw('AVG(total) as avg_order'),
            )
            ->where('status', 'completed')
            ->whereBetween('created_at', [
                "{$from} 00:00:00",
                "{$to} 23:59:59",
            ])
            ->groupBy('period')
            ->orderByDesc('period')
            ->get();

        return $results->map(fn ($row) => [
            'period' => $row->period,
            'total_orders' => number_format($row->total_orders),
            'revenue' => number_format($row->revenue, 2),
            'avg_order' => number_format($row->avg_order, 2),
        ])->toArray();
    }
}

// Controller เหลือแค่นี้
class ReportController extends Controller
{
    public function sales(Request $request, SalesReportService $service)
    {
        $data = $service->generate(
            groupBy: $request->input('group_by', 'daily'),
            from: $request->input('from', today()->toDateString()),
            to: $request->input('to', today()->toDateString()),
        );

        return response()->json(['data' => $data]);
    }
}
```

4 methods → 1 method, ไม่มี SQL injection, เพิ่ม filter ง่าย

---

## 3. The Boolean Trap — Parameter แบบ true/false 7 ตัว

### โค้ดสยองขวัญ

```php
class UserService
{
    public function createUser(
        string $name,
        string $email,
        string $password,
        bool $isAdmin,
        bool $sendWelcomeEmail,
        bool $requireEmailVerification,
        bool $addToNewsletter,
        bool $isPremium,
        bool $skipValidation,
        bool $forceActivate
    ) {
        // ...
    }
}

// เรียกใช้
$userService->createUser(
    'สมชาย',
    'somchai@example.com',
    'password123',
    false,  // อันนี้คืออะไร?
    true,   // อันนี้ล่ะ?
    false,  // ???
    true,   // 🤷
    false,  // ...
    false,  // เดาไม่ออกแล้ว
    true    // เดาเอา
);
```

**อ่านตรงที่เรียกใช้สิครับ** — `true, false, false, true, false, false, true` ไม่มีใครรู้ว่าอันไหนคืออะไร ต้องไปเปิด function definition ทุกครั้ง

### วิธีแก้ — ใช้ Value Object

```php
class CreateUserOptions
{
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly string $password,
        public readonly UserRole $role = UserRole::Member,
        public readonly bool $sendWelcomeEmail = true,
        public readonly bool $requireEmailVerification = true,
        public readonly bool $addToNewsletter = false,
        public readonly bool $isPremium = false,
    ) {}
}

enum UserRole: string
{
    case Admin = 'admin';
    case Member = 'member';
    case Moderator = 'moderator';
}

// เรียกใช้ — อ่านรู้เรื่องทันที
$userService->createUser(new CreateUserOptions(
    name: 'สมชาย',
    email: 'somchai@example.com',
    password: 'password123',
    role: UserRole::Member,
    sendWelcomeEmail: true,
    addToNewsletter: true,
));
```

Named arguments (PHP 8.0+) ทำให้อ่านรู้เรื่องทันทีว่า parameter ไหนคืออะไร

---

## 4. The Swallowed Exception — กลืน Error แล้วแกล้งทำเป็นไม่รู้

### โค้ดสยองขวัญ

```php
class PaymentService
{
    public function processPayment(Order $order): bool
    {
        try {
            $gateway = new PaymentGateway();
            $result = $gateway->charge($order->total);

            if ($result->success) {
                $order->status = 'paid';
                $order->save();
                return true;
            }

            return false;
        } catch (\Exception $e) {
            // 🙈🙈🙈
            return false;
        }
    }
}

// เรียกใช้
if (!$paymentService->processPayment($order)) {
    // payment failed... แต่ทำไม?
    // card ถูก decline? network error? gateway ล่ม? จำนวนเงินผิด?
    // ไม่มีทางรู้เลย เพราะ exception ถูกกลืนหมด
    return response()->json(['error' => 'Something went wrong'], 500);
}
```

ปัญหาสุด classic ครับ — **catch แล้วไม่ทำอะไรเลย** ทำให้ debug เป็นไปไม่ได้ user บอก "ชำระเงินไม่ได้" แต่ log ไม่มีอะไรเลย

### วิธีแก้

```php
class PaymentService
{
    public function __construct(
        private LoggerInterface $logger,
    ) {}

    /**
     * @throws PaymentFailedException
     * @throws PaymentGatewayException
     */
    public function processPayment(Order $order): PaymentResult
    {
        try {
            $result = $this->gateway->charge($order->total);
        } catch (GatewayTimeoutException $e) {
            $this->logger->critical('Payment gateway timeout', [
                'order_id' => $order->id,
                'amount' => $order->total,
                'error' => $e->getMessage(),
            ]);
            throw new PaymentGatewayException(
                "Gateway timeout for order {$order->id}",
                previous: $e,
            );
        } catch (\Exception $e) {
            $this->logger->error('Unexpected payment error', [
                'order_id' => $order->id,
                'exception' => get_class($e),
                'message' => $e->getMessage(),
            ]);
            throw new PaymentFailedException(
                orderId: $order->id,
                reason: $e->getMessage(),
                previous: $e,
            );
        }

        if (!$result->success) {
            $this->logger->warning('Payment declined', [
                'order_id' => $order->id,
                'decline_code' => $result->declineCode,
                'message' => $result->message,
            ]);
            throw new PaymentDeclinedException(
                orderId: $order->id,
                declineCode: $result->declineCode,
            );
        }

        $order->markAsPaid($result->transactionId);

        return $result;
    }
}
```

**กฎ: ห้ามกลืน exception เด็ดขาด** อย่างน้อย log ไว้ แล้วถ้า caller ต้องรู้ ให้ throw ต่อ

---

## 5. The Mega If-Else — เงื่อนไขซ้อนกัน 8 ชั้น

### โค้ดสยองขวัญ

```php
public function getDiscount(User $user, Order $order, ?string $coupon): float
{
    $discount = 0;

    if ($user->is_premium) {
        if ($order->total > 5000) {
            if ($coupon !== null) {
                $c = Coupon::where('code', $coupon)->first();
                if ($c) {
                    if ($c->expires_at > now()) {
                        if ($c->type === 'percent') {
                            $discount = $order->total * ($c->value / 100);
                            if ($discount > $c->max_discount) {
                                $discount = $c->max_discount;
                            }
                        } else {
                            if ($c->type === 'fixed') {
                                $discount = $c->value;
                            } else {
                                $discount = 0;
                            }
                        }
                    } else {
                        $discount = 0;
                    }
                } else {
                    $discount = 0;
                }
            } else {
                $discount = $order->total * 0.10;
            }
        } else {
            if ($coupon !== null) {
                // ...อีก 20 บรรทัด ที่ copy มาจากด้านบน แต่ต่าง %...
            } else {
                $discount = $order->total * 0.05;
            }
        }
    } else {
        if ($order->total > 10000) {
            if ($coupon !== null) {
                // ...copy มาอีก 20 บรรทัด...
            } else {
                $discount = $order->total * 0.03;
            }
        } else {
            if ($coupon !== null) {
                // ...อีกแล้ว...
            }
        }
    }

    return $discount;
}
```

if ซ้อน if ซ้อน if... 8 ชั้น อ่านไม่รู้เรื่อง debug ก็ไม่ได้ test ก็ไม่ครบ case

### วิธีแก้ — Early Return + แยก method

```php
public function getDiscount(User $user, Order $order, ?string $couponCode): float
{
    // coupon discount (ถ้ามี)
    if ($couponCode !== null) {
        return $this->applyCoupon($couponCode, $order);
    }

    // premium member discount
    if ($user->is_premium) {
        return $this->premiumDiscount($order);
    }

    // normal member high-value discount
    if ($order->total > 10_000) {
        return $order->total * 0.03;
    }

    return 0;
}

private function applyCoupon(string $code, Order $order): float
{
    $coupon = Coupon::where('code', $code)->first();

    if ($coupon === null) {
        return 0;
    }

    if ($coupon->isExpired()) {
        return 0;
    }

    $discount = match ($coupon->type) {
        'percent' => $order->total * ($coupon->value / 100),
        'fixed' => $coupon->value,
        default => 0,
    };

    return min($discount, $coupon->max_discount ?? $discount);
}

private function premiumDiscount(Order $order): float
{
    return match (true) {
        $order->total > 5_000 => $order->total * 0.10,
        $order->total > 2_000 => $order->total * 0.05,
        default => 0,
    };
}
```

**กฎ: ถ้า if ซ้อนเกิน 2 ชั้น แปลว่าต้องแยก method**

---

## 6. The Config Graveyard — ค่า hardcode ทั่วทั้ง project

### โค้ดสยองขวัญ

```php
// ไฟล์ที่ 1: OrderService.php
if ($order->total > 10000) { // magic number
    $this->requireApproval($order);
}

// ไฟล์ที่ 2: ShippingService.php
if ($weight > 20) { // 20 อะไร? kg? lb?
    $shipping += 150; // ทำไม 150?
}

// ไฟล์ที่ 3: NotificationService.php
$ch = curl_init('https://hooks.slack.com/services/T00000/B00000/XXXXXXXXXXXX');

// ไฟล์ที่ 4: PaymentController.php
curl_setopt($ch, CURLOPT_HTTPHEADER, [
    'Authorization: Bearer sk_live_xxxxxxxxxxxxxxxxxxxx', // 💀 LIVE API KEY IN CODE
]);

// ไฟล์ที่ 5: UserService.php
if ($loginAttempts > 5) { // ทำไม 5?
    Cache::put("lockout:{$userId}", true, 1800); // 1800 วินาที = 30 นาที (ต้องคำนวณเอง)
}

// ไฟล์ที่ 6: ImageService.php
if ($file->getSize() > 5242880) { // 5MB... ต้องเอา calculator มาคิด
    throw new \Exception('File too large');
}
```

กระจายอยู่ 50+ ไฟล์ เปลี่ยน 1 ค่าต้องไล่หาทั้ง project

### วิธีแก้

```php
// config/business.php — ค่า business logic รวมศูนย์
return [
    'order' => [
        'approval_threshold' => env('ORDER_APPROVAL_THRESHOLD', 10_000),
    ],
    'shipping' => [
        'heavy_weight_kg' => 20,
        'heavy_surcharge' => 150,
    ],
    'security' => [
        'max_login_attempts' => env('MAX_LOGIN_ATTEMPTS', 5),
        'lockout_minutes' => env('LOCKOUT_MINUTES', 30),
    ],
    'upload' => [
        'max_size_mb' => env('UPLOAD_MAX_SIZE_MB', 5),
    ],
];

// ใช้งาน — อ่านรู้เรื่องทันที
if ($order->total > config('business.order.approval_threshold')) {
    $this->requireApproval($order);
}

if ($file->getSize() > config('business.upload.max_size_mb') * 1024 * 1024) {
    throw new FileTooLargeException();
}
```

**ส่วน API key, webhook URL ห้าม hardcode เด็ดขาด** — ใช้ `.env`:

```bash
# .env
PAYMENT_SECRET_KEY=sk_live_xxxxxxxxxxxxxxxxxxxx
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/T00000/B00000/XXXX
```

---

## 7. The Zombie Code — โค้ดตาย แต่ไม่มีใครกล้าลบ

### โค้ดสยองขวัญ

```php
class ProductService
{
    public function updatePrice(Product $product, float $newPrice): void
    {
        // OLD: ใช้ตอน v1
        // if ($product->category === 'electronics') {
        //     $newPrice = $newPrice * 1.07; // + VAT
        // }

        // TODO: เอาไว้ทำทีหลัง (comment นี้อายุ 3 ปีแล้ว)
        // $this->notifyPriceWatch($product, $newPrice);

        // FIXME: @somchai - ถามพี่ต้อมก่อนว่าลบได้ไหม
        // (พี่ต้อมลาออกไปปีที่แล้ว)
        // $this->syncToLegacySystem($product, $newPrice);

        // HACK: แก้ bug #1234 ชั่วคราว
        // (bug #1234 ถูก fix ไปตั้งแต่ปี 2021)
        // if ($newPrice < 0) {
        //     $newPrice = 0;
        // }

        $product->price = $newPrice;
        $product->save();

        // NOTE: ด้านล่างนี้ไม่ได้ใช้แล้ว แต่เก็บไว้เผื่อต้องย้อนกลับ
        // $history = new PriceHistory();
        // $history->product_id = $product->id;
        // $history->old_price = $product->getOriginal('price');
        // $history->new_price = $newPrice;
        // $history->changed_by = auth()->id();
        // $history->save();
        //
        // event(new PriceChanged($product, $newPrice));
        //
        // if ($product->watchers->count() > 0) {
        //     foreach ($product->watchers as $watcher) {
        //         Mail::to($watcher)->send(new PriceAlert($product));
        //     }
        // }
    }

    // ไม่มีใครเรียกใช้แล้ว แต่ไม่กล้าลบ
    public function updatePriceV1($id, $price) { /* ... */ }
    public function updatePriceOld($id, $price) { /* ... */ }
    public function updatePrice_backup($id, $price) { /* ... */ }
    public function updatePriceTemp($id, $price) { /* ... */ }
}
```

**comment มากกว่าโค้ดจริง** มี method ที่ไม่ได้ใช้ 4 ตัว ไม่มีใครกล้าลบเพราะ "เผื่อต้องใช้"

### วิธีแก้

```bash
# ตรวจว่า method ไหนไม่ได้ถูกเรียกใช้จริง
grep -rn "updatePriceV1\|updatePriceOld\|updatePrice_backup\|updatePriceTemp" app/ routes/

# ไม่เจอ? ลบได้เลย — git เก็บ history ไว้ให้แล้ว
```

**กฎ:**
* **ลบ commented-out code ทิ้ง** — git มี history อยู่แล้ว ไม่ต้องเก็บไว้ใน source
* **ลบ TODO ที่อายุ > 3 เดือน** — ถ้ายังไม่ได้ทำ แปลว่าไม่สำคัญพอ สร้าง issue ใน tracker แทน
* **ลบ method ที่ไม่ได้เรียกใช้** — grep ก่อน ถ้าไม่เจอ ลบ
* **ลบ comment ที่ reference คนที่ลาออกแล้ว** — ไม่มีประโยชน์อีกแล้ว

---

## สรุป — Red Flags ที่เจอแล้วต้องรีบแก้

| สัญญาณ | ระดับ | สิ่งที่ต้องทำ |
|---------|-------|-------------|
| **API key hardcode ในโค้ด** | 🔴 วิกฤต | ย้ายเข้า `.env` + rotate key ทันที |
| **SQL injection** | 🔴 วิกฤต | เปลี่ยนเป็น prepared statement / query builder |
| **กลืน exception** ไม่ log | 🟠 สูง | เพิ่ม logging อย่างน้อย |
| **method 300+ บรรทัด** | 🟠 สูง | แยก service class |
| **if ซ้อน 4+ ชั้น** | 🟡 กลาง | early return + แยก method |
| **copy-paste 3+ ที่** | 🟡 กลาง | extract เป็น shared method |
| **magic numbers** | 🟡 กลาง | ดึงเป็น config / constant |
| **commented-out code** | 🔵 ต่ำ | ลบทิ้ง ใช้ git history แทน |

ถ้าอยากรู้วิธี refactor ทีละขั้นอย่างปลอดภัย อ่านต่อที่ [Refactoring ทีละขั้น](refactoring-guide.md) ครับ
