---
description: มาตรฐาน Container Interface — เข้าใจ Dependency Injection Container แบบ PSR-11
---

# PSR-11 Container Interface

> 📌 ถ้าคุณใช้ Laravel อยู่ คุณใช้ PSR-11 อยู่แล้วโดยไม่รู้ตัวครับ เพราะ Service Container ของ Laravel implement PSR-11

### Dependency Injection Container คืออะไร

ก่อนจะไป PSR-11 มาทำความเข้าใจปัญหาก่อนครับ

สมมติเรามี `OrderService` ที่ต้องใช้หลาย dependency:

```php
// ❌ สร้าง dependency เองทุกครั้ง
class OrderController
{
    public function store(Request $request)
    {
        $db = new DatabaseConnection('mysql', 'localhost', 'mydb');
        $mailer = new SmtpMailer('smtp.gmail.com', 587, 'user', 'pass');
        $logger = new FileLogger('/var/log/app.log');
        $paymentGateway = new StripeGateway(config('stripe.key'));

        $orderService = new OrderService($db, $mailer, $logger, $paymentGateway);
        $orderService->create($request->all());
    }
}
```

เห็นปัญหาไหมครับ? ทุกครั้งที่จะใช้ `OrderService` ต้องมานั่งสร้าง dependency ทั้งหมดเอง ถ้ามี 10 controller ที่ใช้ `OrderService` ก็ต้องเขียนโค้ดนี้ 10 ที่

**Container** มาแก้ปัญหานี้ครับ — มันเป็นเหมือน "กล่องเก็บของ" ที่เรา register วิธีสร้าง object ไว้ แล้วเวลาต้องการก็ขอจากกล่องได้เลย

```php
// ✅ ขอจาก Container
class OrderController
{
    public function __construct(
        private OrderService $orderService,
    ) {
        // Container สร้าง OrderService + dependencies ให้อัตโนมัติ
    }

    public function store(Request $request)
    {
        $this->orderService->create($request->all());
    }
}
```

สะอาดกว่าเยอะเลยครับ

---

### 1. PSR-11 กำหนดอะไร

PSR-11 กำหนด interface ง่ายๆ แค่ 2 method:

```php
<?php

namespace Psr\Container;

interface ContainerInterface
{
    /**
     * ดึง service ออกมาจาก container
     *
     * @throws NotFoundExceptionInterface ถ้าไม่เจอ
     * @throws ContainerExceptionInterface ถ้าเกิด error
     */
    public function get(string $id): mixed;

    /**
     * ตรวจว่า container มี service นี้ไหม
     */
    public function has(string $id): bool;
}
```

แค่นี้เลยครับ `get()` กับ `has()` ง่ายมาก

แต่ความง่ายนี้แหละที่ทำให้มันเป็น "สัญญา" ที่ทุก framework ใช้ร่วมกันได้

---

### 2. Exception ที่ PSR-11 กำหนด

PSR-11 กำหนด exception ไว้ 2 ตัว:

```php
// 1. ไม่เจอ service ที่ขอ
namespace Psr\Container;

interface NotFoundExceptionInterface extends ContainerExceptionInterface
{
    // $container->get('something_that_does_not_exist');
    // → throws NotFoundExceptionInterface
}

// 2. error ระหว่างสร้าง service
interface ContainerExceptionInterface
{
    // $container->get('broken_service');
    // → throws ContainerExceptionInterface
}
```

💡 **Best Practice:** เวลา catch exception จาก container ให้ catch `ContainerExceptionInterface` ครับ เพราะ `NotFoundExceptionInterface` เป็น sub-interface ของมันอยู่แล้ว

---

### 3. ใช้กับ Laravel ยังไง

#### 3.1 วิธีใช้พื้นฐาน

ใน Laravel, `app()` helper return ตัว Container ที่ implement PSR-11 ครับ:

```php
// ดึง service ด้วย get()
$logger = app()->get(LoggerInterface::class);

// ตรวจว่ามี service ไหม
if (app()->has(PaymentGateway::class)) {
    $gateway = app()->get(PaymentGateway::class);
}
```

แต่จริงๆ ใน Laravel เราไม่ค่อยเรียก `get()` ตรงๆ ครับ เพราะ Laravel มี **Auto Resolution** ที่ดีกว่า

#### 3.2 Auto Resolution (วิธีที่แนะนำ)

```php
class OrderController
{
    // Laravel จะ resolve dependencies ให้อัตโนมัติ
    public function __construct(
        private OrderService $orderService,
        private LoggerInterface $logger,
    ) {
        //
    }
}
```

แค่ type-hint ใน constructor Laravel จะสร้างให้อัตโนมัติ ไม่ต้องเรียก `get()` เอง

#### 3.3 Register Service ใน Service Provider

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Contracts\PaymentGateway;
use App\Services\StripeGateway;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // bind interface กับ implementation
        $this->app->bind(
            PaymentGateway::class,
            StripeGateway::class,
        );

        // singleton — สร้างครั้งเดียว ใช้ซ้ำตลอด
        $this->app->singleton(
            CacheManager::class,
            fn ($app) => new CacheManager(
                driver: config('cache.driver'),
                ttl: config('cache.ttl'),
            ),
        );
    }
}
```

💡 **Best Practice:** ใช้ `bind()` เป็น default ครับ ใช้ `singleton()` เฉพาะกรณีที่ต้องการ share instance เดียวกัน เช่น database connection หรือ cache

---

### 4. ลองเขียน Container เองแบบง่ายๆ

เพื่อให้เข้าใจว่ามันทำงานยังไง ลองเขียนกันครับ

```php
<?php

namespace App\Container;

use Psr\Container\ContainerInterface;
use Psr\Container\NotFoundExceptionInterface;

class SimpleContainer implements ContainerInterface
{
    /** @var array<string, callable> */
    private array $bindings = [];

    /** @var array<string, mixed> */
    private array $instances = [];

    /**
     * ลงทะเบียน service
     */
    public function bind(string $id, callable $factory): void
    {
        $this->bindings[$id] = $factory;
    }

    /**
     * ลงทะเบียน singleton
     */
    public function singleton(string $id, callable $factory): void
    {
        $this->bindings[$id] = function () use ($id, $factory) {
            if (!isset($this->instances[$id])) {
                $this->instances[$id] = $factory($this);
            }
            return $this->instances[$id];
        };
    }

    /**
     * ดึง service
     */
    public function get(string $id): mixed
    {
        if (!$this->has($id)) {
            throw new class("Service not found: {$id}")
                extends \Exception
                implements NotFoundExceptionInterface {};
        }

        return ($this->bindings[$id])($this);
    }

    /**
     * ตรวจว่ามี service ไหม
     */
    public function has(string $id): bool
    {
        return isset($this->bindings[$id]);
    }
}
```

ใช้งาน:

```php
$container = new SimpleContainer();

// register
$container->bind(LoggerInterface::class, function ($c) {
    return new FileLogger('/var/log/app.log');
});

$container->singleton(DatabaseConnection::class, function ($c) {
    return new DatabaseConnection('mysql', 'localhost', 'mydb');
});

// resolve
$logger = $container->get(LoggerInterface::class);
$db = $container->get(DatabaseConnection::class);
```

ตัวอย่างนี้เป็นแค่ version ง่ายๆ นะครับ Container จริงๆ อย่าง Laravel จะมี auto-wiring, contextual binding, tagged binding อีกเพียบ

---

### 5. ทำไมต้อง PSR-11 ไม่ใช่แค่ใช้ Laravel Container ตรงๆ

คำถามดีครับ คำตอบคือ **ถ้าเขียน library หรือ package ที่ต้องใช้กับหลาย framework**

```php
// ❌ ผูกติดกับ Laravel
use Illuminate\Container\Container;

class MyPackage
{
    public function __construct(Container $container) {}
}

// ✅ ใช้ PSR-11 — ใช้ได้กับทุก framework
use Psr\Container\ContainerInterface;

class MyPackage
{
    public function __construct(ContainerInterface $container) {}
}
```

version PSR-11 ใช้ได้ทั้ง Laravel, Symfony, Slim, PHP-DI ฯลฯ เพราะทุกตัว implement PSR-11 หมด

💡 **Best Practice:**
* ถ้าเขียน **application code** → ใช้ Laravel Container ตรงๆ ได้เลย ไม่ต้องคิดเยอะ
* ถ้าเขียน **package/library** → ใช้ PSR-11 interface จะดีกว่า

---

### 6. สิ่งที่ PSR-11 ไม่ได้กำหนด

PSR-11 เป็น **read-only** interface ครับ มันกำหนดแค่ว่า "ดึงยังไง" ไม่ได้กำหนดว่า "register ยังไง"

เพราะฉะนั้น method พวกนี้ไม่ใช่ส่วนของ PSR-11:
* `bind()` — register service
* `singleton()` — register shared service
* `make()` — สร้าง instance ใหม่ทุกครั้ง
* `tag()` — จัดกลุ่ม service

พวกนี้เป็น feature เพิ่มเติมของแต่ละ framework เอง

---

### สรุป

* PSR-11 กำหนดแค่ **2 method**: `get()` ดึง service, `has()` ตรวจว่ามีไหม
* ใน Laravel ใช้ผ่าน **constructor injection** สะดวกที่สุด ไม่ต้องเรียก `get()` เอง
* **bind()** สร้างใหม่ทุกครั้ง, **singleton()** สร้างครั้งเดียวใช้ซ้ำ
* เขียน application code ใช้ Laravel Container ตรงๆ ได้เลย
* เขียน package/library ใช้ `Psr\Container\ContainerInterface` จะ portable กว่า

อ่านต่อเรื่อง [Service Provider](../../laravel/service-provider/) ประกอบด้วยนะครับ จะเห็นภาพว่า register service เข้า container ยังไงใน Laravel
