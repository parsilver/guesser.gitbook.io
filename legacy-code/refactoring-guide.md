---
description: ขั้นตอน refactor legacy code อย่างปลอดภัย — ไม่ต้อง rewrite ทั้งหมด แค่ค่อยๆ ทำให้ดีขึ้น
---

# Refactoring ทีละขั้น

### ทำไมถึงไม่ rewrite ทั้งหมด

เคยมีคนบอกไหมครับว่า "ทิ้งหมดเลย เขียนใหม่ดีกว่า" — ฟังดูเท่ แต่ **90% ไม่เสร็จ** เพราะ:

* ระบบเก่ามี **business logic ซ่อนอยู่เป็นร้อยจุด** ที่คนเขียนใหม่ไม่รู้
* ระหว่าง rewrite ระบบเก่ายังต้อง **รัน production อยู่** ต้องดูแลสองระบบพร้อมกัน
* ใช้เวลา **6 เดือน - 1 ปี** กว่าจะเสร็จ — ระหว่างนั้น business ก็เพิ่ม requirement มาเรื่อยๆ

ทางที่ดีกว่าคือ **refactor ทีละขั้น** ครับ ค่อยๆ ปรับปรุงโค้ดเดิม ให้ระบบยังทำงานได้ตลอด

---

### ก่อนจะเริ่ม — กฎเหล็ก 3 ข้อ

1. **อย่า refactor โดยไม่มี test** — ถ้าไม่มี test แล้วแก้โค้ด จะไม่รู้ว่าพังตรงไหน
2. **อย่า refactor กับ feature ใน PR เดียวกัน** — แยก PR refactor กับ PR feature ออกจากกัน ถ้ามีปัญหาจะ revert ได้ง่าย
3. **อย่า refactor ทุกอย่างพร้อมกัน** — เลือกจุดที่ปวดที่สุดก่อน ทำทีละจุด

---

### Step 1: ล้อมรั้วด้วย Test ก่อน

ก่อนจะแตะอะไร **เขียน test ครอบโค้ดเดิมให้ได้ก่อน** — เรียกว่า Characterization Test ครับ

Characterization Test คือ test ที่บันทึก **behavior ปัจจุบัน** ของโค้ด ไม่ว่ามันจะถูกหรือผิดก็ตาม เพราะเป้าหมายคือ "ถ้าเรา refactor แล้ว behavior เปลี่ยน ต้องรู้ทันที"

```php
// โค้ดเดิม — ไม่รู้ว่า logic ถูกหรือผิด แต่มันทำงานมา 3 ปีแล้ว
class PriceCalculator
{
    public function calculate(float $price, string $customerType, bool $hasDiscount): float
    {
        if ($customerType === 'vip') {
            $price = $price * 0.8;
        }
        if ($hasDiscount) {
            $price = $price - 100;
        }
        if ($price < 0) {
            $price = 0;
        }
        return $price;
    }
}
```

```php
// Characterization Test — บันทึก behavior ปัจจุบัน
class PriceCalculatorTest extends TestCase
{
    private PriceCalculator $calculator;

    protected function setUp(): void
    {
        $this->calculator = new PriceCalculator();
    }

    public function test_normal_customer_no_discount(): void
    {
        $this->assertEquals(1000.0, $this->calculator->calculate(1000, 'normal', false));
    }

    public function test_vip_gets_20_percent_off(): void
    {
        // VIP ลด 20% → 1000 * 0.8 = 800
        $this->assertEquals(800.0, $this->calculator->calculate(1000, 'vip', false));
    }

    public function test_discount_subtracts_100(): void
    {
        // ลด 100 บาท → 1000 - 100 = 900
        $this->assertEquals(900.0, $this->calculator->calculate(1000, 'normal', true));
    }

    public function test_vip_with_discount_applies_both(): void
    {
        // VIP 20% ก่อน → 1000 * 0.8 = 800 แล้วลดอีก 100 → 700
        $this->assertEquals(700.0, $this->calculator->calculate(1000, 'vip', true));
    }

    public function test_price_never_goes_below_zero(): void
    {
        // ของราคา 50 บาท ลด 100 → ต้องได้ 0 ไม่ใช่ -50
        $this->assertEquals(0.0, $this->calculator->calculate(50, 'normal', true));
    }
}
```

💡 **เทคนิค:** ถ้าไม่รู้ว่าโค้ดจะ return อะไร ให้เขียน test ด้วยค่าอะไรก็ได้ก่อน แล้วรัน — test จะ fail แล้วบอกค่าจริงให้เรา เอาค่าจริงมาใส่ test ซะ

```php
// ยิง assert มั่วๆ ก่อน
$this->assertEquals(999, $this->calculator->calculate(1000, 'vip', true));
// Failed: Expected 999, Actual 700.0
// เอา! ก็ใส่ 700.0 ลงไปแทน
```

---

### Step 2: หา Seams (จุดที่แยก dependency ได้)

Seam คือ **จุดที่เราสามารถเปลี่ยน behavior ของโค้ดได้โดยไม่ต้องแก้โค้ดตรงนั้น** — concept จาก Michael Feathers ในหนังสือ "Working Effectively with Legacy Code"

พูดง่ายๆ คือ หาจุดที่สามารถ "สอด" code ใหม่เข้าไปแทนได้

```php
// ❌ ก่อน — hardcode dependency ตรงๆ
class ReportGenerator
{
    public function generate(): string
    {
        // dependency ถูกสร้างใน method เลย ไม่สามารถ mock ได้
        $db = new MySQLConnection('localhost', 'root', 'password');
        $data = $db->query('SELECT * FROM sales');

        $pdf = new PDFWriter('/tmp/report.pdf');
        $pdf->write($data);

        mail('boss@company.com', 'Report', 'See attachment');

        return '/tmp/report.pdf';
    }
}
```

```php
// ✅ หลัง — ย้าย dependency ออกมาเป็น parameter (Dependency Injection)
class ReportGenerator
{
    public function __construct(
        private DatabaseConnection $db,
        private ReportWriter $writer,
        private Mailer $mailer,
    ) {
        //
    }

    public function generate(): string
    {
        $data = $this->db->query('SELECT * FROM sales');

        $path = $this->writer->write($data);

        $this->mailer->send('boss@company.com', 'Report', 'See attachment');

        return $path;
    }
}
```

ตอนนี้เราสามารถ **mock ทุก dependency** ใน test ได้แล้ว และ method `generate()` ก็ไม่ผูกกับ MySQL, PDF, หรือ mail server จริงอีกต่อไป

💡 **เทคนิค Sprout Method:** ถ้าไม่กล้าแก้ method เดิม ให้ "แตกหน่อ" ออกมาเป็น method ใหม่

```php
// แทนที่จะแก้ method เดิมที่ยาว 200 บรรทัด
// สร้าง method ใหม่แล้วเรียกจากจุดเดิม
class OrderService
{
    public function processOrder(array $data): void
    {
        // ... โค้ดเดิม 200 บรรทัดที่ไม่กล้าแตะ ...

        // แทรก method ใหม่เข้าไป
        $this->applyLoyaltyPoints($data['user_id'], $data['total']);

        // ... โค้ดเดิมต่อ ...
    }

    // method ใหม่ที่เขียนอย่างสะอาด + มี test
    private function applyLoyaltyPoints(int $userId, float $total): void
    {
        $points = (int) floor($total / 100);
        $this->loyaltyService->addPoints($userId, $points);
    }
}
```

---

### Step 3: แยก Layer ออกจากกัน

โค้ดเก่ามักจะ **ปนกันหมด** — database query, business logic, HTML อยู่ใน method เดียวกัน

```php
// ❌ ก่อน — ทุกอย่างอยู่ใน Controller
class ProductController
{
    public function search(Request $request)
    {
        $keyword = $request->input('q');
        $category = $request->input('category');

        // SQL query ใน controller
        $products = DB::select("
            SELECT p.*, c.name as category_name
            FROM products p
            JOIN categories c ON c.id = p.category_id
            WHERE p.name LIKE ?
            AND (? IS NULL OR p.category_id = ?)
            AND p.is_active = 1
            ORDER BY p.created_at DESC
        ", ["%{$keyword}%", $category, $category]);

        // Business logic ใน controller
        foreach ($products as &$product) {
            if ($product->stock <= 0) {
                $product->badge = 'สินค้าหมด';
            } elseif ($product->stock <= 5) {
                $product->badge = 'เหลือน้อย';
            }

            if ($product->discount_percent > 0) {
                $product->final_price = $product->price * (1 - $product->discount_percent / 100);
            } else {
                $product->final_price = $product->price;
            }
        }

        return view('products.search', compact('products'));
    }
}
```

แยกออกมาทีละ layer ครับ:

```php
// ✅ Step 3a: ย้าย query ไป Repository
class ProductRepository
{
    public function search(string $keyword, ?int $categoryId): Collection
    {
        return Product::query()
            ->with('category')
            ->where('name', 'like', "%{$keyword}%")
            ->when($categoryId, fn ($q) => $q->where('category_id', $categoryId))
            ->where('is_active', true)
            ->latest()
            ->get();
    }
}
```

```php
// ✅ Step 3b: ย้าย business logic ไป Service / Model
class Product extends Model
{
    public function getBadgeAttribute(): ?string
    {
        if ($this->stock <= 0) return 'สินค้าหมด';
        if ($this->stock <= 5) return 'เหลือน้อย';
        return null;
    }

    public function getFinalPriceAttribute(): float
    {
        if ($this->discount_percent > 0) {
            return $this->price * (1 - $this->discount_percent / 100);
        }
        return $this->price;
    }
}
```

```php
// ✅ Step 3c: Controller สะอาดขึ้น
class ProductController
{
    public function __construct(
        private ProductRepository $products,
    ) {
        //
    }

    public function search(Request $request)
    {
        $products = $this->products->search(
            keyword: $request->input('q', ''),
            categoryId: $request->input('category'),
        );

        return view('products.search', compact('products'));
    }
}
```

ตอนนี้แต่ละ layer test ได้แยกกันแล้ว:
* **Repository** — test ว่า query ถูก
* **Model** — test ว่า badge กับ final_price คำนวณถูก
* **Controller** — test ว่า wire ถูก ส่ง data ถูก view

---

### Step 4: ผ่า God Class

God Class คือ class ที่ **ทำทุกอย่าง** — ยาวเป็นพันบรรทัด มี method เป็นสิบๆ

วิธีผ่ามันคือ **หา group ของ method ที่ทำงานเกี่ยวกัน** แล้วย้ายออกไปเป็น class ใหม่

```php
// ❌ ก่อน — UserService 2,000 บรรทัด
class UserService
{
    // กลุ่ม 1: Authentication
    public function login() { /* ... */ }
    public function logout() { /* ... */ }
    public function resetPassword() { /* ... */ }
    public function verifyEmail() { /* ... */ }

    // กลุ่ม 2: Profile
    public function updateProfile() { /* ... */ }
    public function uploadAvatar() { /* ... */ }
    public function changeEmail() { /* ... */ }

    // กลุ่ม 3: Notification
    public function sendWelcomeEmail() { /* ... */ }
    public function sendPasswordResetEmail() { /* ... */ }
    public function notifyAdmin() { /* ... */ }

    // กลุ่ม 4: Subscription
    public function subscribe() { /* ... */ }
    public function cancelSubscription() { /* ... */ }
    public function checkSubscriptionStatus() { /* ... */ }
}
```

```php
// ✅ หลัง — แยกเป็น class ย่อย ตาม responsibility
class AuthService
{
    public function login() { /* ... */ }
    public function logout() { /* ... */ }
    public function resetPassword() { /* ... */ }
    public function verifyEmail() { /* ... */ }
}

class ProfileService
{
    public function updateProfile() { /* ... */ }
    public function uploadAvatar() { /* ... */ }
    public function changeEmail() { /* ... */ }
}

class UserNotificationService
{
    public function sendWelcomeEmail() { /* ... */ }
    public function sendPasswordResetEmail() { /* ... */ }
    public function notifyAdmin() { /* ... */ }
}

class SubscriptionService
{
    public function subscribe() { /* ... */ }
    public function cancel() { /* ... */ }
    public function checkStatus() { /* ... */ }
}
```

💡 **เทคนิค Facade:** ถ้าคนอื่นเรียก `UserService` อยู่เยอะ ไม่ต้องแก้ทุกจุดพร้อมกัน ให้สร้าง `UserService` ตัวเดิมเป็น **facade** ที่ delegate ไปยัง class ย่อย

```php
// UserService ตัวเดิม — ทำหน้าที่เป็น facade ชั่วคราว
class UserService
{
    public function __construct(
        private AuthService $auth,
        private ProfileService $profile,
        private UserNotificationService $notification,
        private SubscriptionService $subscription,
    ) {
        //
    }

    /** @deprecated ใช้ AuthService::login() แทน */
    public function login()
    {
        return $this->auth->login();
    }

    /** @deprecated ใช้ ProfileService::updateProfile() แทน */
    public function updateProfile()
    {
        return $this->profile->updateProfile();
    }

    // ... delegate ตัวอื่นเหมือนกัน
}
```

แล้วค่อยๆ ย้ายที่เรียกจาก `UserService` ไปเรียก class ย่อยตรงๆ ทีละจุด

---

### Step 5: จัดการ Shared Mutable State

โค้ดเก่าชอบมี **global state** หรือ **property ที่ถูกแก้จากหลายที่** ซึ่งทำให้ debug ยาก

```php
// ❌ ก่อน — class มี state เยอะ แก้จากหลาย method
class Cart
{
    public array $items = [];
    public float $total = 0;
    public float $discount = 0;
    public string $couponCode = '';
    public bool $freeShipping = false;

    public function addItem($item): void
    {
        $this->items[] = $item;
        $this->total += $item['price'] * $item['qty'];
        $this->recalcDiscount();
    }

    public function applyCoupon(string $code): void
    {
        $this->couponCode = $code;
        $this->recalcDiscount(); // แก้ $discount
        $this->checkFreeShipping(); // แก้ $freeShipping
    }

    private function recalcDiscount(): void
    {
        // แก้ $this->discount จาก logic ที่ซับซ้อน
        // บาง case แก้ $this->freeShipping ด้วย!
    }
}
```

```php
// ✅ หลัง — ใช้ immutable value object แทน
class CartSummary
{
    public function __construct(
        public readonly float $subtotal,
        public readonly float $discount,
        public readonly float $shipping,
        public readonly float $total,
    ) {
        //
    }
}

class Cart
{
    private array $items = [];
    private ?string $couponCode = null;

    public function addItem(CartItem $item): void
    {
        $this->items[] = $item;
    }

    public function applyCoupon(string $code): void
    {
        $this->couponCode = $code;
    }

    // คำนวณ summary ใหม่ทุกครั้ง ไม่เก็บ state
    public function getSummary(): CartSummary
    {
        $subtotal = array_sum(
            array_map(fn (CartItem $i) => $i->price * $i->qty, $this->items)
        );

        $discount = $this->calculateDiscount($subtotal, $this->couponCode);
        $shipping = $this->calculateShipping($subtotal - $discount);
        $total = $subtotal - $discount + $shipping;

        return new CartSummary($subtotal, $discount, $shipping, $total);
    }
}
```

ข้อดีคือ `CartSummary` เป็น **readonly** แก้ไม่ได้ และ **คำนวณใหม่ทุกครั้ง** ไม่มี stale state

---

### Step 6: ใส่ Type ให้ชัดเจน

PHP 7+ รองรับ type hints ครับ — ถ้าโค้ดเก่าไม่มี type ให้ค่อยๆ ใส่เข้าไป

```php
// ❌ ก่อน — ไม่มี type เลย ไม่รู้ว่า parameter/return เป็นอะไร
class InvoiceService
{
    public function create($customer, $items, $options = [])
    {
        // $customer คืออะไร? array? object? int (id)?
        // $items เป็น array ของอะไร?
        // $options มี key อะไรบ้าง?
    }
}
```

```php
// ✅ หลัง — type ชัด อ่านรู้เรื่องทันที
class InvoiceService
{
    /**
     * @param array<InvoiceItem> $items
     */
    public function create(
        Customer $customer,
        array $items,
        InvoiceOptions $options = new InvoiceOptions(),
    ): Invoice {
        // ชัดเจนเลยว่ารับอะไร คืนอะไร
    }
}

// ใช้ DTO/Value Object แทน associative array
class InvoiceOptions
{
    public function __construct(
        public readonly string $currency = 'THB',
        public readonly bool $includeVat = true,
        public readonly ?string $note = null,
    ) {
        //
    }
}
```

💡 **ใส่ทีละไฟล์** ที่เปิดมาแก้ bug/เพิ่ม feature — ไม่ต้องไล่ใส่ทั้ง project วันเดียว ตาม **Boy Scout Rule**

---

### Step 7: ลบ Dead Code

โค้ดที่ไม่ได้ใช้ ยิ่งเก็บไว้ยิ่งสร้างความสับสน — กล้าลบเลยครับ Git จำได้

```php
// ❌ ก่อน — comment out ไว้ "เผื่อใช้"
class PaymentService
{
    // public function processPaypal($data)
    // {
    //     // เปลี่ยนไปใช้ Stripe แล้ว - 2021-03-15
    //     // แต่เก็บไว้ก่อนเผื่อ...
    //     $paypal = new PaypalGateway();
    //     ...
    // }

    public function processStripe(array $data): PaymentResult
    {
        // ...
    }

    // ไม่มีใครเรียก method นี้เลย
    public function generateLegacyReport(): void
    {
        // ...
    }
}
```

```php
// ✅ หลัง — ลบ dead code ออก
class PaymentService
{
    public function processStripe(array $data): PaymentResult
    {
        // ...
    }
}
```

วิธีหา dead code:
* ใช้ IDE — PhpStorm บอกได้ว่า method ไหนไม่มีคนเรียก (สีเทา)
* ใช้ tool เช่น `phpstan --level=0` หรือ `psalm`
* ค้นหา `grep -r "methodName" .` — ถ้าเจอแค่ที่ประกาศ แสดงว่าไม่มีใครใช้

---

### Refactoring Checklist

ก่อน refactor ทุกครั้ง ลอง check ดูครับ:

| ✅ เช็ค | ทำไม |
|---------|------|
| มี test ครอบ behavior เดิมแล้ว? | ถ้าไม่มี อย่าเพิ่ง refactor |
| แยก PR refactor กับ PR feature? | revert ง่าย review ง่าย |
| รัน test ผ่านหลัง refactor? | ถ้า fail = behavior เปลี่ยน ต้องตรวจสอบ |
| โค้ดอ่านง่ายขึ้นจริงหรือเปล่า? | refactor ไม่ใช่ over-engineer |
| ทีมเข้าใจโครงสร้างใหม่ไหม? | refactor ที่คนอื่นอ่านไม่ออก = ไม่มีประโยชน์ |

---

### สรุป

* **เขียน test ก่อน refactor เสมอ** — Characterization Test บันทึก behavior เดิม
* **หา Seams** — จุดที่แยก dependency ออกได้ แล้วใช้ DI
* **แยก layer** — Repository, Service, Model แต่ละชั้น test ได้แยกกัน
* **ผ่า God Class** — หา group ของ method ที่เกี่ยวกัน แยกเป็น class ย่อย
* **ลด mutable state** — ใช้ immutable value object แทน
* **ใส่ type hints** — ทีละไฟล์ตาม Boy Scout Rule
* **ลบ dead code** — กล้าลบ Git จำได้

อ่านต่อ: [ย้ายระบบใหญ่ด้วย Strangler Fig](strangler-fig.md) สำหรับกรณีที่ต้องย้ายระบบเก่าทั้งระบบ
