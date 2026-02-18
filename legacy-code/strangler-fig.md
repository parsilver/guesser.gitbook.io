---
description: ค่อยๆ แทนที่ระบบเก่าโดยไม่ต้องปิดระบบ — เหมือนต้นไทรที่ค่อยๆ โอบรัดต้นเดิม
---

# ย้ายระบบใหญ่ด้วย Strangler Fig

### Strangler Fig คืออะไร

ถ้าเคยเห็น **ต้นไทร** ที่ค่อยๆ เจริญเติบโตรอบต้นไม้อื่น — มันไม่ได้โค่นต้นเดิมทิ้ง แต่ค่อยๆ ขยายรากและลำต้นจนปกคลุมต้นเดิมหมด สุดท้ายต้นเดิมก็ค่อยๆ ถูกแทนที่

Martin Fowler เอา concept นี้มาใช้กับ software ครับ — แทนที่จะ rewrite ระบบเก่าทั้งหมด ให้ **สร้างระบบใหม่ค่อยๆ ครอบระบบเก่า** แล้วย้าย feature ทีละตัว

---

### ทำไมไม่ Big Bang Migration

Big Bang = ปิดระบบเก่า เปิดระบบใหม่ วันเดียวจบ ฟังดูเท่แต่...

* **ความเสี่ยงสูงมาก** — ถ้าระบบใหม่พัง ไม่มีแผน B
* **ต้องย้ายทุกอย่างพร้อมกัน** — database, API, frontend, third-party integrations
* **ทีมต้องหยุดพัฒนา feature** ระหว่างย้าย
* **User ต้องปรับตัวทีเดียว** กับทุกอย่างที่เปลี่ยน

Strangler Fig ดีกว่าเพราะ:

* ย้าย **ทีละส่วน** — ความเสี่ยงต่ำ
* ระบบเก่ายังรันอยู่ — ถ้าส่วนใหม่พัง **rollback ได้ทันที**
* ทีมยัง **deliver feature ได้** ระหว่างย้าย
* User **ค่อยๆ ปรับ** ทีละนิด

---

### หลักการ 3 ขั้นตอน

ทุกรอบของ Strangler Fig มี 3 ขั้น:

**1. Transform** — สร้าง feature ใหม่ในระบบใหม่
**2. Coexist** — ระบบเก่ากับใหม่ทำงานคู่กัน ผ่าน routing layer
**3. Eliminate** — เมื่อมั่นใจแล้ว ลบระบบเก่าทิ้ง

ทำซ้ำ 3 ขั้นนี้กับแต่ละ feature จนย้ายครบ

---

### ตัวอย่างจริง — ย้ายจาก Monolith PHP เป็น Laravel ใหม่

สมมติมีระบบ e-commerce เก่าที่เขียนด้วย PHP แบบ plain (ไม่มี framework) อยากย้ายไป Laravel ทีละส่วน

#### สถาปัตยกรรมเดิม

```
ระบบเก่า (Pure PHP)
├── product-list.php
├── product-detail.php
├── cart.php
├── checkout.php
├── admin/
│   ├── orders.php
│   ├── products.php
│   └── reports.php
└── api/
    ├── get-products.php
    └── create-order.php
```

ทุกหน้าเป็นไฟล์ PHP แยกๆ database query ฝังในแต่ละไฟล์ session จัดการเอง ไม่มี routing layer

#### Step 1: วาง Proxy (Routing Layer) ข้างหน้า

ก่อนจะย้ายอะไร สร้าง **proxy layer** ไว้ข้างหน้า ที่จะเป็นตัวตัดสินว่า request ไหนไประบบเก่า request ไหนไประบบใหม่

```nginx
# nginx.conf — ตั้ง reverse proxy
server {
    listen 80;
    server_name shop.example.com;

    # Default: ส่งทุกอย่างไประบบเก่าก่อน
    location / {
        proxy_pass http://legacy-app:8080;
    }

    # เมื่อพร้อม: route เฉพาะส่วนที่ย้ายแล้วไประบบใหม่
    # location /api/v2/products {
    #     proxy_pass http://new-laravel-app:8000;
    # }
}
```

ตอนนี้ทุก request ยังไประบบเก่าหมด แต่เรามี **จุดควบคุม** ที่สามารถเปลี่ยน routing ได้แล้ว

#### Step 2: เลือก Feature แรกที่จะย้าย

เลือก feature ที่:
* **แยกตัว (self-contained)** — ไม่ผูกกับส่วนอื่นมาก
* **มี business value** — ไม่ใช่แค่หน้า static
* **ไม่ซับซ้อนเกินไป** — เป็น pilot ที่ทีมเรียนรู้ process

ในตัวอย่างนี้ เลือก **Product Catalog API** ก่อนครับ — เป็น read-only, ไม่ต้องจัดการ session, ย้ายง่ายสุด

#### Step 3: สร้างใหม่ใน Laravel

```php
// routes/api.php
Route::prefix('v2')->group(function () {
    Route::get('/products', [ProductController::class, 'index']);
    Route::get('/products/{id}', [ProductController::class, 'show']);
});
```

```php
// app/Http/Controllers/Api/ProductController.php
class ProductController extends Controller
{
    public function __construct(
        private ProductRepository $products,
    ) {
        //
    }

    public function index(Request $request): JsonResponse
    {
        $products = $this->products->search(
            keyword: $request->input('q'),
            category: $request->input('category'),
            page: $request->integer('page', 1),
        );

        return response()->json($products);
    }

    public function show(int $id): JsonResponse
    {
        $product = $this->products->findOrFail($id);
        return response()->json($product);
    }
}
```

สำคัญ: **ใช้ database เดียวกัน** กับระบบเก่าได้ครับ ไม่จำเป็นต้องย้าย database ตอนนี้

```php
// config/database.php — ชี้ไป database เดิม
'connections' => [
    'legacy' => [
        'driver' => 'mysql',
        'host' => env('LEGACY_DB_HOST'),
        'database' => env('LEGACY_DB_DATABASE'),
        // ...
    ],
],
```

#### Step 4: ทดสอบเปรียบเทียบ (Shadow Testing)

ก่อนเปิดใช้งานจริง ให้ **เปรียบเทียบ response** ระหว่างระบบเก่ากับใหม่

```php
// Middleware สำหรับ shadow testing
class ShadowTestMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        // เรียกระบบใหม่
        $newResponse = $next($request);

        // เรียกระบบเก่าเปรียบเทียบ (async)
        dispatch(function () use ($request, $newResponse) {
            $legacyResponse = Http::get(
                config('legacy.base_url') . $request->getPathInfo(),
                $request->query()
            );

            if ($this->isDifferent($legacyResponse, $newResponse)) {
                Log::warning('Shadow test mismatch', [
                    'path' => $request->getPathInfo(),
                    'legacy' => $legacyResponse->json(),
                    'new' => $newResponse->getData(),
                ]);
            }
        });

        return $newResponse;
    }
}
```

เมื่อ response ตรงกันจนมั่นใจแล้ว ค่อยเปิด routing

#### Step 5: สลับ Traffic

```nginx
# nginx.conf — เปิด route ใหม่
location /api/v2/products {
    proxy_pass http://new-laravel-app:8000;
}

# ระบบเก่ายังรับ route อื่นอยู่
location / {
    proxy_pass http://legacy-app:8080;
}
```

ถ้าพังก็แค่ comment route ใหม่ออก ระบบเก่ารับทันที — **rollback ภายใน 1 นาที**

#### Step 6: ทำซ้ำกับ Feature ถัดไป

```
รอบที่ 1: Product Catalog API        ✅ ย้ายแล้ว
รอบที่ 2: Cart API                    ✅ ย้ายแล้ว
รอบที่ 3: Checkout + Payment          🔄 กำลังย้าย
รอบที่ 4: Admin - Order Management    ⏳ รอคิว
รอบที่ 5: Admin - Reports             ⏳ รอคิว
รอบที่ 6: ลบระบบเก่าทิ้ง              ⏳ รอทุกอย่างย้ายครบ
```

---

### Pattern ย่อยที่ช่วยในการย้าย

#### Asset Capture — ย้ายหน้า Frontend ทีละหน้า

```nginx
# ย้ายหน้า product ไปใช้ frontend ใหม่
location /products {
    proxy_pass http://new-frontend:3000;
}

# หน้าอื่นยังใช้ของเก่า
location / {
    proxy_pass http://legacy-app:8080;
}
```

#### Branch by Abstraction — ใช้ Feature Flag

```php
class PaymentService
{
    public function process(Order $order): PaymentResult
    {
        // ใช้ feature flag สลับระหว่าง implementation เก่ากับใหม่
        if (Feature::isEnabled('new-payment-gateway')) {
            return $this->newPaymentGateway->process($order);
        }

        return $this->legacyPaymentProcess($order);
    }
}
```

เปิดให้ user บางกลุ่มก่อน (canary release) แล้วค่อยๆ ขยาย:

```php
// เปิดให้ 10% ของ user ก่อน
Feature::define('new-payment-gateway', function (User $user) {
    return $user->id % 10 === 0;
});

// มั่นใจแล้ว เปิดให้ทุกคน
Feature::define('new-payment-gateway', fn () => true);
```

#### Event Interception — ดัก Event จากระบบเก่า

บางทีระบบเก่าไม่มี event system แต่เราสามารถดักจาก database ได้:

```php
// ระบบใหม่ poll database ของระบบเก่า
// เพื่อจับ order ใหม่แล้วส่ง event
class LegacyOrderPoller
{
    public function poll(): void
    {
        $newOrders = DB::connection('legacy')
            ->table('orders')
            ->where('created_at', '>', $this->lastPollTime)
            ->get();

        foreach ($newOrders as $order) {
            event(new LegacyOrderCreated($order));
        }

        $this->lastPollTime = now();
    }
}
```

หรือใช้ **database trigger** ส่ง event ผ่าน queue:

```sql
-- MySQL trigger ที่ส่ง message เมื่อมี order ใหม่
CREATE TRIGGER after_order_insert
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    INSERT INTO event_queue (event_type, payload, created_at)
    VALUES ('order_created', JSON_OBJECT('order_id', NEW.id), NOW());
END;
```

---

### Data Migration Strategy

เรื่อง database เป็นส่วนที่ยากที่สุดครับ มีหลายวิธี:

#### 1. Shared Database (ง่ายสุด เริ่มจากตรงนี้)

```
ระบบเก่า ──┐
            ├──► Database เดียวกัน
ระบบใหม่ ──┘
```

ทั้งสองระบบใช้ database เดียวกัน ข้อเสียคือ schema ต้องเข้ากันได้ทั้งสองฝั่ง

#### 2. Database View (กลางๆ)

```php
// ระบบใหม่อ่านจาก view ที่ map schema เก่าเป็นรูปแบบใหม่
// database/migrations/create_product_view.php
DB::statement("
    CREATE VIEW products_v2 AS
    SELECT
        p.prod_id AS id,
        p.prod_name AS name,
        p.prod_desc AS description,
        p.prod_price / 100 AS price,  -- ระบบเก่าเก็บเป็นสตางค์
        c.cat_name AS category_name
    FROM tbl_products p
    LEFT JOIN tbl_categories c ON c.cat_id = p.cat_id
    WHERE p.is_deleted = 0
");
```

ระบบใหม่ไม่ต้องรู้ schema เก่า แค่อ่านจาก view ที่แปลง format ให้แล้ว

#### 3. Dual Write (สำหรับ schema ใหม่)

เมื่อพร้อมจะแยก database ให้เขียนทั้งสองที่:

```php
class OrderService
{
    public function create(array $data): Order
    {
        // เขียนลง database ใหม่ (primary)
        $order = Order::create($data);

        // เขียนลง database เก่าด้วย (จนกว่าจะย้ายครบ)
        DB::connection('legacy')->table('orders')->insert([
            'order_num' => $order->id,
            'cust_id' => $order->customer_id,
            'total_amt' => $order->total * 100, // แปลงกลับเป็นสตางค์
        ]);

        return $order;
    }
}
```

เมื่อมั่นใจว่าระบบเก่าไม่มีใครใช้แล้ว ก็ลบ dual write ออก

---

### เมื่อไหร่ควรใช้ Strangler Fig

✅ **ใช้เมื่อ:**
* ระบบเก่า **ใหญ่เกินกว่าจะ rewrite ทีเดียว**
* ต้อง **ไม่หยุด business** ระหว่างย้าย
* อยาก **ลดความเสี่ยง** ย้ายทีละส่วน
* ทีม **เรียนรู้ tech ใหม่** ไปพร้อมกัน

❌ **ไม่ต้องใช้เมื่อ:**
* ระบบเล็ก สามารถ rewrite ใน **1-2 sprint** ได้
* ระบบเก่า **ไม่มีใครใช้แล้ว** แค่ migrate data ก็พอ
* **ไม่มี routing layer** ที่สามารถแทรกได้ (ถ้าเป็น embedded system, desktop app)

---

### สรุป

* **อย่า Big Bang** — ย้ายทีละส่วน ลดความเสี่ยง
* **วาง Proxy/Router** ข้างหน้าเป็นอย่างแรก — ควบคุมว่า traffic ไปไหน
* **ใช้ database เดียวกันก่อน** — ค่อยแยกทีหลัง
* **Shadow test** เปรียบเทียบ response ก่อนสลับ
* **Feature flag** ช่วยให้ค่อยๆ เปิดให้ user ทีละกลุ่ม
* **Rollback ได้เสมอ** — comment route แล้ว traffic กลับไประบบเก่าทันที
* ทำซ้ำ **Transform → Coexist → Eliminate** จนย้ายครบทุก feature
