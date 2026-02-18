---
description: เปลี่ยน algorithm ได้ตอน runtime โดยไม่ต้องแก้ class หลัก — pattern ที่ใช้บ่อยที่สุดตัวนึงเลย
---

# Strategy Pattern

### ปัญหาที่เจอบ่อย

สมมติเราเขียนระบบคำนวณค่าจัดส่งครับ ตอนแรกมีแค่ Kerry:

```php
class ShippingCalculator
{
    public function calculate(Order $order): float
    {
        // คำนวณค่าส่ง Kerry
        return $order->weight * 25;
    }
}
```

ง่ายดีนะ แต่แล้ว business ก็บอก "เพิ่ม Flash ด้วย"

```php
class ShippingCalculator
{
    public function calculate(Order $order, string $carrier): float
    {
        if ($carrier === 'kerry') {
            return $order->weight * 25;
        } elseif ($carrier === 'flash') {
            return $order->weight * 20;
        }

        throw new \Exception("Unknown carrier: {$carrier}");
    }
}
```

แล้วก็มาอีก "เพิ่ม J&T, Thailand Post, Ninja Van..."

```php
// 💀 6 เดือนผ่านไป...
public function calculate(Order $order, string $carrier): float
{
    if ($carrier === 'kerry') {
        return $order->weight * 25;
    } elseif ($carrier === 'flash') {
        return $order->weight * 20;
    } elseif ($carrier === 'jt') {
        if ($order->weight > 10) {
            return $order->weight * 18 + 50;
        }
        return $order->weight * 22;
    } elseif ($carrier === 'thaipost') {
        // ...อีก 30 บรรทัด...
    } elseif ($carrier === 'ninja') {
        // ...อีก 20 บรรทัด...
    }
}
```

เห็นปัญหาไหมครับ? ทุกครั้งที่เพิ่มขนส่งใหม่ ต้องมาแก้ class นี้ ยิ่งนานยิ่งยาว ยิ่ง test ยาก

---

### Strategy Pattern มาช่วย

หลักการง่ายมากครับ: **แยก algorithm แต่ละตัวออกเป็น class ของมันเอง แล้วให้เลือกใช้ตอน runtime**

มี 3 ส่วน:

1. **Strategy Interface** — สัญญาว่า algorithm ต้องมี method อะไร
2. **Concrete Strategies** — algorithm แต่ละตัว (Kerry, Flash, J&T)
3. **Context** — class ที่เรียกใช้ strategy

---

### Step 1: สร้าง Interface

```php
<?php

namespace App\Contracts;

use App\Models\Order;

interface ShippingStrategy
{
    /**
     * คำนวณค่าจัดส่ง
     */
    public function calculate(Order $order): float;

    /**
     * ชื่อผู้ให้บริการ
     */
    public function name(): string;
}
```

interface นี้เป็น "สัญญา" ว่า ไม่ว่าจะเป็นขนส่งไหน ต้องมี `calculate()` กับ `name()`

---

### Step 2: สร้าง Concrete Strategies

```php
<?php

namespace App\Shipping;

use App\Contracts\ShippingStrategy;
use App\Models\Order;

class KerryStrategy implements ShippingStrategy
{
    private const RATE_PER_KG = 25;
    private const MIN_CHARGE = 50;

    public function calculate(Order $order): float
    {
        $cost = $order->weight * self::RATE_PER_KG;

        return max($cost, self::MIN_CHARGE);
    }

    public function name(): string
    {
        return 'Kerry Express';
    }
}
```

```php
<?php

namespace App\Shipping;

use App\Contracts\ShippingStrategy;
use App\Models\Order;

class FlashStrategy implements ShippingStrategy
{
    private const RATE_PER_KG = 20;
    private const HEAVY_THRESHOLD = 10;
    private const HEAVY_SURCHARGE = 30;

    public function calculate(Order $order): float
    {
        $cost = $order->weight * self::RATE_PER_KG;

        if ($order->weight > self::HEAVY_THRESHOLD) {
            $cost += self::HEAVY_SURCHARGE;
        }

        return $cost;
    }

    public function name(): string
    {
        return 'Flash Express';
    }
}
```

```php
<?php

namespace App\Shipping;

use App\Contracts\ShippingStrategy;
use App\Models\Order;

class ThaiPostStrategy implements ShippingStrategy
{
    public function calculate(Order $order): float
    {
        // คิดตามขนาด + น้ำหนัก
        $weightCost = $order->weight * 15;
        $sizeCost = $order->isOversized() ? 40 : 0;

        return $weightCost + $sizeCost;
    }

    public function name(): string
    {
        return 'Thailand Post';
    }
}
```

แต่ละ class ดูแลแค่ logic ของตัวเอง สะอาด อ่านง่าย test ง่าย

---

### Step 3: สร้าง Context

```php
<?php

namespace App\Services;

use App\Contracts\ShippingStrategy;
use App\Models\Order;

class ShippingCalculator
{
    public function __construct(
        private ShippingStrategy $strategy,
    ) {
        //
    }

    /**
     * เปลี่ยน strategy ตอน runtime
     */
    public function setStrategy(ShippingStrategy $strategy): void
    {
        $this->strategy = $strategy;
    }

    /**
     * คำนวณค่าจัดส่ง
     */
    public function calculate(Order $order): float
    {
        return $this->strategy->calculate($order);
    }

    /**
     * ชื่อขนส่งที่ใช้อยู่
     */
    public function carrierName(): string
    {
        return $this->strategy->name();
    }
}
```

สังเกตนะครับ `ShippingCalculator` ไม่รู้เลยว่ากำลังใช้ Kerry หรือ Flash มันแค่เรียก `calculate()` ตาม interface

---

### Step 4: ใช้งานจริง

```php
use App\Services\ShippingCalculator;
use App\Shipping\KerryStrategy;
use App\Shipping\FlashStrategy;

// สร้างด้วย Kerry
$calculator = new ShippingCalculator(new KerryStrategy());
echo $calculator->calculate($order); // 250

// เปลี่ยนเป็น Flash ตอน runtime
$calculator->setStrategy(new FlashStrategy());
echo $calculator->calculate($order); // 200
```

---

### ทำให้ดีขึ้น — ใช้ Factory ช่วยเลือก Strategy

ถ้าไม่อยากให้ controller ต้องรู้จัก strategy class ทุกตัว สร้าง factory ช่วยได้:

```php
<?php

namespace App\Shipping;

use App\Contracts\ShippingStrategy;

class ShippingStrategyFactory
{
    /**
     * @var array<string, class-string<ShippingStrategy>>
     */
    private static array $strategies = [
        'kerry' => KerryStrategy::class,
        'flash' => FlashStrategy::class,
        'thaipost' => ThaiPostStrategy::class,
    ];

    public static function make(string $carrier): ShippingStrategy
    {
        $class = self::$strategies[$carrier] ?? null;

        if ($class === null) {
            throw new \InvalidArgumentException(
                "Unknown carrier: {$carrier}. Available: " . implode(', ', array_keys(self::$strategies))
            );
        }

        return new $class();
    }

    /**
     * ดูว่ามีขนส่งอะไรบ้าง
     */
    public static function available(): array
    {
        return array_keys(self::$strategies);
    }
}
```

ใช้งาน:

```php
// ใน Controller
public function calculateShipping(Request $request)
{
    $strategy = ShippingStrategyFactory::make($request->input('carrier'));
    $calculator = new ShippingCalculator($strategy);

    return response()->json([
        'carrier' => $calculator->carrierName(),
        'cost' => $calculator->calculate($order),
    ]);
}
```

💡 **Best Practice:** เวลาเพิ่มขนส่งใหม่ แค่สร้าง class ใหม่ที่ implement `ShippingStrategy` แล้วเพิ่มใน factory **ไม่ต้องแก้โค้ดเก่าเลย** — นี่คือหลัก Open/Closed Principle (SOLID)

---

### อีกตัวอย่าง — Payment Gateway

ใช้ได้กับทุกสถานการณ์ที่มี "หลาย algorithm เลือกใช้" ครับ:

```php
interface PaymentStrategy
{
    public function charge(float $amount): PaymentResult;
    public function refund(string $transactionId, float $amount): PaymentResult;
}

class StripePayment implements PaymentStrategy
{
    public function charge(float $amount): PaymentResult
    {
        // เรียก Stripe API
    }

    public function refund(string $transactionId, float $amount): PaymentResult
    {
        // เรียก Stripe Refund API
    }
}

class PromptPayPayment implements PaymentStrategy
{
    public function charge(float $amount): PaymentResult
    {
        // สร้าง QR code สำหรับ PromptPay
    }

    public function refund(string $transactionId, float $amount): PaymentResult
    {
        // PromptPay ไม่ support refund อัตโนมัติ
        throw new RefundNotSupportedException('PromptPay requires manual refund');
    }
}
```

---

### เมื่อไหร่ควรใช้ Strategy Pattern

✅ **ใช้เมื่อ:**
* มีหลาย algorithm ที่ทำงานคล้ายกัน (ขนส่ง, ชำระเงิน, ส่ง notification)
* ต้องเลือก algorithm ตอน runtime (user เลือกวิธีจัดส่ง)
* ไม่อยากให้ class หลักโตขึ้นเรื่อยๆ ตาม if-else

❌ **ไม่ควรใช้เมื่อ:**
* มีแค่ 2 ตัวเลือก ที่ไม่น่าจะเพิ่มอีก — if-else ธรรมดาก็พอ
* algorithm ไม่เคยเปลี่ยน — over-engineering

---

### สรุป

* **แยก** algorithm แต่ละตัวออกเป็น class ที่ implement interface เดียวกัน
* **Context** (class หลัก) ไม่ต้องรู้ว่าใช้ algorithm ตัวไหน แค่เรียกผ่าน interface
* **เพิ่ม algorithm ใหม่** โดยไม่ต้องแก้โค้ดเดิมเลย
* ใน Laravel จะเห็น pattern นี้ในบทความ [วิธีเก็บ Log ผ่าน Middleware](../laravel/log-request-response-laravel-middleware-events/log-requests-middleware-1.md) ที่ใช้ `HttpLogger` interface เป็น strategy
