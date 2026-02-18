---
description: เมื่อมีอะไรเกิดขึ้น ให้หลายส่วนรู้พร้อมกัน โดยไม่ต้องผูกกันแน่น
---

# Observer Pattern

### ปัญหาที่เจอ

สมมติเราเขียนระบบ e-commerce เมื่อมี order ใหม่ ต้องทำหลายอย่าง:

```php
class OrderService
{
    public function create(array $data): Order
    {
        $order = Order::create($data);

        // ส่ง email ยืนยัน
        Mail::to($order->user)->send(new OrderConfirmation($order));

        // แจ้ง Slack
        Slack::send("New order #{$order->id}");

        // อัปเดต stock
        $this->inventoryService->deduct($order->items);

        // ส่ง analytics
        Analytics::track('order_created', $order->toArray());

        // อัปเดต loyalty points
        $this->loyaltyService->addPoints($order->user, $order->total);

        return $order;
    }
}
```

ดูปัญหาครับ:

* **OrderService ต้องรู้จักทุกอย่าง** — email, Slack, inventory, analytics, loyalty
* **เพิ่ม feature ใหม่ = แก้ OrderService** — ผิด Open/Closed Principle
* **Test ยากมาก** — ต้อง mock ทุก dependency
* **ถ้า analytics พัง ก็กระทบ order** — ทั้งที่ไม่เกี่ยวกัน

---

### Observer Pattern มาช่วย

หลักการคือ **แยกระหว่าง "สิ่งที่เกิดขึ้น" กับ "สิ่งที่ต้องทำเมื่อเกิดขึ้น"**

มี 2 ฝ่าย:
* **Subject (Publisher)** — บอกว่า "เฮ้ มีอะไรเกิดขึ้นแล้วนะ"
* **Observer (Subscriber/Listener)** — บอกว่า "โอเค ฉันจะจัดการเอง"

Subject ไม่ต้องรู้ว่า Observer ทำอะไร แค่แจ้งไป

---

### Step 1: สร้าง Event (สิ่งที่เกิดขึ้น)

```php
<?php

namespace App\Events;

class OrderCreated
{
    public function __construct(
        public readonly Order $order,
        public readonly \DateTimeImmutable $occurredAt,
    ) {
        //
    }

    public static function from(Order $order): self
    {
        return new self($order, new \DateTimeImmutable());
    }
}
```

Event เป็นแค่ **data class** ครับ ไม่มี logic อะไร แค่บอกว่า "order นี้ถูกสร้างแล้ว" พร้อมข้อมูลที่จำเป็น

---

### Step 2: สร้าง Event Dispatcher

```php
<?php

namespace App\Events;

class EventDispatcher
{
    /** @var array<string, array<callable>> */
    private array $listeners = [];

    /**
     * ลงทะเบียน listener สำหรับ event
     */
    public function listen(string $eventClass, callable $listener): void
    {
        $this->listeners[$eventClass][] = $listener;
    }

    /**
     * แจ้ง event ให้ listener ทุกตัว
     */
    public function dispatch(object $event): void
    {
        $eventClass = get_class($event);

        foreach ($this->listeners[$eventClass] ?? [] as $listener) {
            $listener($event);
        }
    }
}
```

Dispatcher ทำหน้าที่เป็น **ตัวกลาง** — รับ event เข้ามา แล้วส่งต่อให้ listener ทุกตัวที่ลงทะเบียนไว้

---

### Step 3: สร้าง Listeners (Observer)

```php
<?php

namespace App\Listeners;

use App\Events\OrderCreated;

class SendOrderConfirmationEmail
{
    public function __invoke(OrderCreated $event): void
    {
        Mail::to($event->order->user)->send(
            new OrderConfirmation($event->order)
        );
    }
}
```

```php
<?php

namespace App\Listeners;

use App\Events\OrderCreated;

class NotifySlack
{
    public function __invoke(OrderCreated $event): void
    {
        Slack::send("New order #{$event->order->id} — ฿{$event->order->total}");
    }
}
```

```php
<?php

namespace App\Listeners;

use App\Events\OrderCreated;

class DeductInventory
{
    public function __construct(
        private InventoryService $inventory,
    ) {
        //
    }

    public function __invoke(OrderCreated $event): void
    {
        $this->inventory->deduct($event->order->items);
    }
}
```

```php
<?php

namespace App\Listeners;

use App\Events\OrderCreated;

class TrackAnalytics
{
    public function __invoke(OrderCreated $event): void
    {
        Analytics::track('order_created', [
            'order_id' => $event->order->id,
            'total' => $event->order->total,
        ]);
    }
}
```

แต่ละ listener **ดูแลแค่ 1 เรื่อง** เท่านั้น สะอาดมากครับ

---

### Step 4: ประกอบร่างและใช้งาน

```php
// ลงทะเบียน listeners
$dispatcher = new EventDispatcher();
$dispatcher->listen(OrderCreated::class, new SendOrderConfirmationEmail());
$dispatcher->listen(OrderCreated::class, new NotifySlack());
$dispatcher->listen(OrderCreated::class, new DeductInventory($inventoryService));
$dispatcher->listen(OrderCreated::class, new TrackAnalytics());

// OrderService สะอาดขึ้นมาก
class OrderService
{
    public function __construct(
        private EventDispatcher $dispatcher,
    ) {
        //
    }

    public function create(array $data): Order
    {
        $order = Order::create($data);

        // แค่แจ้ง event เดียว จบ!
        $this->dispatcher->dispatch(OrderCreated::from($order));

        return $order;
    }
}
```

ดูครับ OrderService ไม่รู้จัก email, Slack, inventory, analytics เลย แค่ dispatch event แล้วจบ

💡 **Best Practice:** ตั้งชื่อ event เป็น **past tense** เสมอ — `OrderCreated`, `PaymentReceived`, `UserRegistered` เพราะมันคือ "สิ่งที่เกิดขึ้นแล้ว"

---

### ทำให้ดีขึ้น — Listener ด้วย Priority

บางทีลำดับ listener สำคัญ เช่น ต้อง deduct stock ก่อน ค่อยส่ง email:

```php
class EventDispatcher
{
    /** @var array<string, array<array{listener: callable, priority: int}>> */
    private array $listeners = [];

    public function listen(string $eventClass, callable $listener, int $priority = 0): void
    {
        $this->listeners[$eventClass][] = [
            'listener' => $listener,
            'priority' => $priority,
        ];
    }

    public function dispatch(object $event): void
    {
        $eventClass = get_class($event);
        $listeners = $this->listeners[$eventClass] ?? [];

        // sort by priority (สูง = รันก่อน)
        usort($listeners, fn ($a, $b) => $b['priority'] <=> $a['priority']);

        foreach ($listeners as $item) {
            ($item['listener'])($event);
        }
    }
}

// ใช้งาน
$dispatcher->listen(OrderCreated::class, new DeductInventory($inv), priority: 100);
$dispatcher->listen(OrderCreated::class, new SendOrderConfirmationEmail(), priority: 50);
$dispatcher->listen(OrderCreated::class, new TrackAnalytics(), priority: 10);
```

---

### ทำให้ดีขึ้นอีก — หยุด Event Propagation

บางครั้งอยากให้ listener ตัวนึงหยุดการทำงานของตัวอื่นได้:

```php
class StoppableEvent
{
    private bool $stopped = false;

    public function stopPropagation(): void
    {
        $this->stopped = true;
    }

    public function isPropagationStopped(): bool
    {
        return $this->stopped;
    }
}

class OrderCreated extends StoppableEvent
{
    public function __construct(
        public readonly Order $order,
    ) {
        //
    }
}

// ใน Dispatcher
public function dispatch(object $event): void
{
    foreach ($this->getListeners($event) as $listener) {
        if ($event instanceof StoppableEvent && $event->isPropagationStopped()) {
            break;
        }
        $listener($event);
    }
}
```

```php
// Listener ที่ตรวจ fraud
class FraudCheck
{
    public function __invoke(OrderCreated $event): void
    {
        if ($this->isFraudulent($event->order)) {
            $event->order->cancel();
            $event->stopPropagation(); // หยุด! ไม่ต้องทำอะไรต่อ
        }
    }
}
```

💡 **Best Practice:** ถ้าใช้ stoppable events ให้ตั้ง priority ให้ listener ที่อาจหยุดการทำงาน (เช่น fraud check) มีค่าสูงสุดเพื่อให้รันก่อน

---

### Observer Pattern ในงานจริง

Pattern นี้อยู่ทุกที่ครับ:

* **Event system** — DOM events ใน JavaScript, Laravel Events, Symfony EventDispatcher
* **Webhook** — เมื่อ Stripe ชำระเงินสำเร็จ ส่ง event มาหา server เรา
* **Message Queue** — RabbitMQ, Redis Pub/Sub, AWS SNS/SQS
* **Database trigger** — เมื่อ row ถูกเพิ่ม ให้ทำอะไรบางอย่าง
* **Model Observer** — Laravel Eloquent Observer ก็ใช้ pattern นี้

---

### เมื่อไหร่ควรใช้ Observer

✅ **ใช้เมื่อ:**
* เมื่อเกิด event แล้วต้องทำหลายอย่าง (ส่ง email + อัปเดต stock + log)
* ไม่อยากให้ class หลักรู้จักทุก dependency
* อยากเพิ่ม/ลบ listener ได้ง่ายโดยไม่แก้โค้ดเดิม
* ต้องการ loose coupling ระหว่าง module

❌ **ไม่ควรใช้เมื่อ:**
* มีแค่ 1-2 อย่างที่ต้องทำ — เรียกตรงๆ ง่ายกว่า
* ลำดับการทำงานสำคัญมากและซับซ้อน — observer ทำให้ flow อ่านยาก
* ต้องการ return value จาก listener (observer ปกติเป็น fire-and-forget)

---

### สรุป

* **แยก "สิ่งที่เกิดขึ้น" (Event) กับ "สิ่งที่ต้องทำ" (Listener)** ออกจากกัน
* Subject **แค่ dispatch event** ไม่ต้องรู้ว่าใครฟังอยู่
* เพิ่ม/ลบ listener ได้ **โดยไม่ต้องแก้ class หลัก**
* ตั้งชื่อ event เป็น past tense: `OrderCreated`, `UserRegistered`
* ใน Laravel ดูตัวอย่างจริงได้ที่บทความ [EventServiceProvider](../laravel/service-provider/eventserviceprovider.md)
