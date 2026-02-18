---
description: มาตรฐาน Logger Interface — เขียน Log ยังไงให้เปลี่ยน library ได้โดยไม่ต้องแก้โค้ด
---

# PSR-3 Logger Interface

> 📌 PSR-3 เป็นมาตรฐานที่ Laravel ใช้อยู่เบื้องหลังผ่าน Monolog — ถ้าคุณเคยเขียน `Log::info()` หรือ `Log::error()` นั่นแหละ PSR-3

### ปัญหาที่ PSR-3 มาแก้

สมมติเราเขียน log ใน project แบบนี้ครับ

```php
// ใช้ Monolog ตรงๆ
$logger = new \Monolog\Logger('app');
$logger->pushHandler(new StreamHandler('app.log'));

$logger->info('User logged in', ['user_id' => 1]);
$logger->error('Payment failed', ['order_id' => 123]);
```

ดูก็ปกตินะ แต่วันนึง boss บอกว่า "เปลี่ยนไปใช้ Sentry แทน Monolog ได้ไหม"

ถ้าเราเรียก Monolog ตรงๆ ทั่ว project... ก็ต้องไล่แก้ทุกไฟล์เลยครับ 💀

PSR-3 แก้ปัญหานี้โดยกำหนด **interface กลาง** ว่า logger ต้องมี method อะไรบ้าง ไม่ว่าจะใช้ library ไหนอยู่เบื้องหลัง

---

### 1. LoggerInterface — หัวใจของ PSR-3

PSR-3 กำหนด interface ไว้แบบนี้ครับ

```php
<?php

namespace Psr\Log;

interface LoggerInterface
{
    public function emergency(string|\Stringable $message, array $context = []): void;
    public function alert(string|\Stringable $message, array $context = []): void;
    public function critical(string|\Stringable $message, array $context = []): void;
    public function error(string|\Stringable $message, array $context = []): void;
    public function warning(string|\Stringable $message, array $context = []): void;
    public function notice(string|\Stringable $message, array $context = []): void;
    public function info(string|\Stringable $message, array $context = []): void;
    public function debug(string|\Stringable $message, array $context = []): void;
    public function log($level, string|\Stringable $message, array $context = []): void;
}
```

มี 8 levels ตามมาตรฐาน RFC 5424 เรียงจากรุนแรงที่สุดไปน้อยที่สุด

---

### 2. Log Levels — ใช้ตัวไหนเมื่อไหร่

อันนี้สำคัญมากครับ เพราะหลายคนใช้แต่ `info` กับ `error` ทั้งที่มันมีตั้ง 8 ระดับ

#### emergency

ระบบพัง ใช้งานไม่ได้เลย

```php
$logger->emergency('Database server is down', [
    'server' => 'db-primary',
    'last_seen' => '2024-01-15 14:30:00',
]);
```

💡 ใช้เมื่อ: ระบบทั้งหมดใช้ไม่ได้ ต้องปลุกคนมาแก้ตอนตี 3

#### alert

ต้องแก้ไขทันที

```php
$logger->alert('Disk space below 5%', [
    'disk' => '/dev/sda1',
    'available' => '2.1GB',
]);
```

💡 ใช้เมื่อ: ยังไม่พัง แต่กำลังจะพัง

#### critical

ระบบบางส่วนพัง

```php
$logger->critical('Payment gateway connection failed', [
    'gateway' => 'stripe',
    'error' => 'Connection timeout',
]);
```

💡 ใช้เมื่อ: feature สำคัญใช้ไม่ได้ แต่ระบบอื่นยังปกติ

#### error

เกิด error แต่ระบบยังทำงานต่อได้

```php
$logger->error('Failed to send welcome email', [
    'user_id' => 42,
    'email' => 'user@example.com',
    'exception' => $e->getMessage(),
]);
```

💡 ใช้เมื่อ: มี error แต่ไม่กระทบ flow หลัก

#### warning

เตือนว่ามีอะไรผิดปกติ

```php
$logger->warning('API rate limit reaching threshold', [
    'current' => 850,
    'limit' => 1000,
    'reset_at' => '2024-01-15 15:00:00',
]);
```

💡 ใช้เมื่อ: ยังไม่ error แต่ถ้าไม่ดูแลจะเป็น error

#### notice

เหตุการณ์ปกติ แต่สำคัญพอจะบันทึกไว้

```php
$logger->notice('User changed their email address', [
    'user_id' => 42,
    'old_email' => 'old@example.com',
    'new_email' => 'new@example.com',
]);
```

#### info

ข้อมูลทั่วไป

```php
$logger->info('Order created successfully', [
    'order_id' => 'ORD-2024-001',
    'total' => 1500.00,
]);
```

#### debug

ข้อมูลสำหรับ debug เท่านั้น

```php
$logger->debug('Cache hit for user profile', [
    'user_id' => 42,
    'cache_key' => 'user:42:profile',
    'ttl_remaining' => 3600,
]);
```

💡 **Best Practice:** อย่าเปิด debug ใน production นะครับ log จะเยอะมากจนหา error จริงๆ ไม่เจอ

---

### 3. Context Array — ข้อมูลเพิ่มเติม

สังเกตว่าทุก method มี parameter `$context` ครับ ตัวนี้สำคัญมาก อย่า log แค่ข้อความเปล่าๆ

```php
// ❌ ไม่ดี — ไม่มี context
$logger->error('Payment failed');

// ✅ ดี — มี context ครบ
$logger->error('Payment failed', [
    'order_id' => $order->id,
    'amount' => $order->total,
    'gateway' => 'stripe',
    'error_code' => $e->getCode(),
    'error_message' => $e->getMessage(),
]);
```

💡 **Best Practice:** ใส่ context ทุกครั้งครับ เวลาไปไล่ดู log ใน production จะขอบคุณตัวเองมาก

#### Message Placeholder

PSR-3 support placeholder ใน message ด้วย:

```php
$logger->info('User {username} logged in from {ip}', [
    'username' => 'john',
    'ip' => '192.168.1.1',
]);
// ผลลัพธ์: "User john logged in from 192.168.1.1"
```

ใช้ `{key}` แล้วส่ง key เดียวกันใน context array ครับ

---

### 4. ใช้กับ Laravel ยังไง

Laravel implement PSR-3 ให้หมดแล้วครับ เราใช้ผ่าน Facade ได้เลย

```php
use Illuminate\Support\Facades\Log;

// ใช้ Log Facade
Log::info('Order created', ['order_id' => 123]);
Log::error('Payment failed', ['error' => $e->getMessage()]);

// หรือ inject ผ่าน constructor (แนะนำ)
use Psr\Log\LoggerInterface;

class PaymentService
{
    public function __construct(
        private LoggerInterface $logger,
    ) {
        //
    }

    public function charge(Order $order): void
    {
        try {
            // process payment...
            $this->logger->info('Payment successful', [
                'order_id' => $order->id,
            ]);
        } catch (\Exception $e) {
            $this->logger->error('Payment failed', [
                'order_id' => $order->id,
                'exception' => $e->getMessage(),
            ]);
            throw $e;
        }
    }
}
```

สังเกตนะครับ constructor รับ `Psr\Log\LoggerInterface` ไม่ใช่ Monolog ตรงๆ ทำให้เปลี่ยน logger ได้ง่ายโดยไม่ต้องแก้ class นี้

💡 **Best Practice:** ใช้ dependency injection แทน `Log::` Facade ครับ จะ test ง่ายกว่ามาก

---

### 5. เขียน Logger ของเราเอง

ลองเขียน logger ง่ายๆ ที่ implement PSR-3 กันครับ

```php
<?php

namespace App\Logging;

use Psr\Log\AbstractLogger;
use Psr\Log\LogLevel;

class SlackLogger extends AbstractLogger
{
    public function __construct(
        private string $webhookUrl,
        private string $channel = '#alerts',
    ) {
        //
    }

    public function log($level, string|\Stringable $message, array $context = []): void
    {
        // ส่งแค่ error ขึ้นไป
        $severeLevels = [
            LogLevel::EMERGENCY,
            LogLevel::ALERT,
            LogLevel::CRITICAL,
            LogLevel::ERROR,
        ];

        if (!in_array($level, $severeLevels)) {
            return;
        }

        $text = "[{$level}] {$message}";

        if (!empty($context)) {
            $text .= "\n```" . json_encode($context, JSON_PRETTY_PRINT) . "```";
        }

        // ส่ง HTTP request ไป Slack webhook
        // ...
    }
}
```

สังเกตว่าเรา extend `AbstractLogger` แทน implement `LoggerInterface` ตรงๆ ครับ เพราะ AbstractLogger implement method ทั้ง 8 ตัวให้แล้ว เราแค่เขียน `log()` ตัวเดียวก็พอ

---

### 6. Config Log Channel ใน Laravel

Laravel ใช้ระบบ channel ในการจัดการ log ตั้งค่าที่ `config/logging.php`:

```php
'channels' => [
    'stack' => [
        'driver' => 'stack',
        'channels' => ['daily', 'slack'],
    ],

    'daily' => [
        'driver' => 'daily',
        'path' => storage_path('logs/laravel.log'),
        'level' => 'debug',
        'days' => 14,
    ],

    'slack' => [
        'driver' => 'slack',
        'url' => env('LOG_SLACK_WEBHOOK_URL'),
        'level' => 'error',
    ],
],
```

ตัวอย่างนี้จะ log ทุกอย่างลง file แต่ส่ง Slack เฉพาะ error ขึ้นไป

💡 **Best Practice:** ใช้ `stack` channel เสมอครับ จะได้ส่ง log ไปหลายที่พร้อมกัน และตั้ง `days` สำหรับ daily driver ด้วย ไม่งั้น log จะกินพื้นที่ server ไม่หยุด

---

### สรุป

* PSR-3 กำหนด **interface กลาง** สำหรับ logger ทุกตัว
* มี **8 log levels** — ใช้ให้ถูกระดับ ไม่ใช่ยัด `info` กับ `error` อย่างเดียว
* **ใส่ context** ทุกครั้ง — อย่า log แค่ข้อความเปล่า
* ใน Laravel ใช้ `Psr\Log\LoggerInterface` inject เข้ามา ดีกว่าใช้ Facade ตรงๆ
* เปลี่ยน logger ได้ง่ายโดยไม่ต้องแก้ business logic เลย

ถ้าอ่านบทความ [วิธีการเก็บ Log Request และ Response ใน Laravel](../../laravel/log-request-response-laravel-middleware-events/) ด้วยจะเห็นภาพมากขึ้นครับ ว่าทำไม interface ถึงสำคัญ
