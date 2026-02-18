---
description: เพิ่มความสามารถให้ object โดยไม่ต้องแก้ class เดิม — wrap แล้ว wrap อีกได้เรื่อยๆ
---

# Decorator Pattern

### ปัญหาที่เจอ

สมมติเรามี class สำหรับส่ง notification:

```php
class EmailNotifier
{
    public function send(string $message): void
    {
        // ส่ง email
        mail($this->to, 'Notification', $message);
    }
}
```

วันนึง boss บอก "เพิ่ม log ทุกครั้งที่ส่งด้วย" แก้ยังไงดี?

**วิธีที่ 1: แก้ class เดิม**

```php
class EmailNotifier
{
    public function send(string $message): void
    {
        Log::info('Sending email', ['message' => $message]);
        mail($this->to, 'Notification', $message);
        Log::info('Email sent');
    }
}
```

ก็ได้ แต่ถ้ามาอีก "เพิ่ม rate limit ด้วย", "เพิ่ม retry ด้วย", "เพิ่ม metric ด้วย"... class นี้จะบวมขึ้นเรื่อยๆ แล้วแต่ละที่ก็ไม่ได้อยากได้ทุก feature — บางที่อยากได้แค่ log บางที่อยากได้แค่ retry

**วิธีที่ 2: สร้าง subclass**

```php
class LoggingEmailNotifier extends EmailNotifier { ... }
class RetryEmailNotifier extends EmailNotifier { ... }
class LoggingRetryEmailNotifier extends EmailNotifier { ... }
class RateLimitedLoggingRetryEmailNotifier extends EmailNotifier { ... }
// 💀 class ระเบิด
```

ยิ่งเพิ่ม feature ยิ่งต้องสร้าง class ใหม่แบบ combination ไม่ไหวครับ

---

### Decorator Pattern มาช่วย

หลักการคือ **wrap object ด้วย object อีกตัวที่มี interface เดียวกัน** แล้วเพิ่ม behavior ก่อนหรือหลังเรียก object ข้างใน

เหมือนห่อของขวัญครับ — ของข้างในยังเหมือนเดิม แต่ข้างนอกเพิ่มสิ่งใหม่ไปเรื่อยๆ

---

### Step 1: สร้าง Interface

```php
<?php

namespace App\Contracts;

interface Notifier
{
    public function send(string $to, string $message): void;
}
```

---

### Step 2: สร้าง Base Implementation

```php
<?php

namespace App\Notifications;

use App\Contracts\Notifier;

class EmailNotifier implements Notifier
{
    public function send(string $to, string $message): void
    {
        // ส่ง email จริงๆ
        mail($to, 'Notification', $message);
    }
}
```

---

### Step 3: สร้าง Decorator Base Class

```php
<?php

namespace App\Notifications\Decorators;

use App\Contracts\Notifier;

abstract class NotifierDecorator implements Notifier
{
    public function __construct(
        protected Notifier $inner,
    ) {
        //
    }
}
```

สังเกตนะครับ:
* Decorator **implement interface เดียวกัน** กับ object ที่ wrap
* รับ `Notifier` เข้ามาเก็บไว้ใน `$inner` — นี่คือ object ที่เรา wrap

---

### Step 4: สร้าง Concrete Decorators

#### Logging Decorator

```php
<?php

namespace App\Notifications\Decorators;

use App\Contracts\Notifier;
use Psr\Log\LoggerInterface;

class LoggingNotifier extends NotifierDecorator
{
    public function __construct(
        Notifier $inner,
        private LoggerInterface $logger,
    ) {
        parent::__construct($inner);
    }

    public function send(string $to, string $message): void
    {
        $this->logger->info('Sending notification', [
            'to' => $to,
            'message_length' => strlen($message),
        ]);

        $this->inner->send($to, $message);

        $this->logger->info('Notification sent', ['to' => $to]);
    }
}
```

#### Retry Decorator

```php
<?php

namespace App\Notifications\Decorators;

use App\Contracts\Notifier;

class RetryNotifier extends NotifierDecorator
{
    public function __construct(
        Notifier $inner,
        private int $maxRetries = 3,
        private int $delayMs = 1000,
    ) {
        parent::__construct($inner);
    }

    public function send(string $to, string $message): void
    {
        $lastException = null;

        for ($attempt = 1; $attempt <= $this->maxRetries; $attempt++) {
            try {
                $this->inner->send($to, $message);
                return; // สำเร็จ ออกเลย
            } catch (\Exception $e) {
                $lastException = $e;

                if ($attempt < $this->maxRetries) {
                    usleep($this->delayMs * 1000 * $attempt); // exponential backoff
                }
            }
        }

        throw $lastException;
    }
}
```

#### Rate Limit Decorator

```php
<?php

namespace App\Notifications\Decorators;

use App\Contracts\Notifier;

class RateLimitedNotifier extends NotifierDecorator
{
    private array $sentTimestamps = [];

    public function __construct(
        Notifier $inner,
        private int $maxPerMinute = 60,
    ) {
        parent::__construct($inner);
    }

    public function send(string $to, string $message): void
    {
        $this->cleanOldTimestamps();

        if (count($this->sentTimestamps) >= $this->maxPerMinute) {
            throw new \RuntimeException(
                "Rate limit exceeded: {$this->maxPerMinute} per minute"
            );
        }

        $this->inner->send($to, $message);
        $this->sentTimestamps[] = time();
    }

    private function cleanOldTimestamps(): void
    {
        $oneMinuteAgo = time() - 60;
        $this->sentTimestamps = array_filter(
            $this->sentTimestamps,
            fn ($ts) => $ts > $oneMinuteAgo,
        );
    }
}
```

---

### Step 5: ประกอบร่าง

นี่คือจุดเด็ดของ Decorator ครับ — **เลือก wrap เฉพาะที่ต้องการ**:

```php
use App\Notifications\EmailNotifier;
use App\Notifications\Decorators\LoggingNotifier;
use App\Notifications\Decorators\RetryNotifier;
use App\Notifications\Decorators\RateLimitedNotifier;

// แค่ส่ง email ธรรมดา
$notifier = new EmailNotifier();

// ส่ง email + log
$notifier = new LoggingNotifier(
    new EmailNotifier(),
    $logger,
);

// ส่ง email + retry + log
$notifier = new LoggingNotifier(
    new RetryNotifier(
        new EmailNotifier(),
        maxRetries: 3,
    ),
    $logger,
);

// ส่ง email + rate limit + retry + log (เต็ม combo!)
$notifier = new LoggingNotifier(
    new RateLimitedNotifier(
        new RetryNotifier(
            new EmailNotifier(),
            maxRetries: 3,
        ),
        maxPerMinute: 100,
    ),
    $logger,
);

// ทุกตัวใช้แบบเดียวกัน!
$notifier->send('user@example.com', 'Hello!');
```

สังเกตนะครับ ไม่ว่าจะ wrap กี่ชั้น **เรียกใช้ด้วย `send()` เหมือนกันหมด** เพราะทุกตัว implement `Notifier` interface เดียวกัน

💡 **Best Practice:** ลำดับการ wrap สำคัญครับ ตัวนอกสุดทำก่อน ในตัวอย่างข้างบน log จะรันก่อน → rate limit → retry → ส่งจริง

---

### Decorator ใน PHP จริงๆ

เราเจอ Decorator Pattern ใน PHP บ่อยมากครับ:

**Middleware** — นี่คือ Decorator เลย!

```php
// แต่ละ middleware wrap request/response
$app = new RateLimitMiddleware(
    new AuthMiddleware(
        new LoggingMiddleware(
            new Application(),
        ),
    ),
);
```

**Stream Wrapper**

```php
// PHP streams ก็เป็น Decorator
$stream = fopen('php://temp', 'r+');
$filtered = stream_filter_append($stream, 'string.toupper');
// ข้อมูลถูก filter ก่อนส่งออก
```

**PSR-7 Middleware** ก็ทำงานแบบเดียวกัน

---

### เมื่อไหร่ควรใช้ Decorator

✅ **ใช้เมื่อ:**
* อยากเพิ่ม behavior โดยไม่แก้ class เดิม (logging, caching, retry, rate limit)
* อยากเลือกว่า instance ไหนจะมี behavior อะไรบ้าง (ไม่ใช่ทุกตัวต้องเหมือนกัน)
* อยาก combine features หลายตัวเข้าด้วยกัน

❌ **ไม่ควรใช้เมื่อ:**
* ต้องเข้าถึง property ภายในของ object ที่ wrap (Decorator เห็นแค่ public interface)
* มีแค่ behavior เดียวที่ต้องเพิ่ม — extend หรือแก้ class เลยง่ายกว่า
* Decorator ซ้อนกันเยอะเกินไป (5+ ชั้น) อ่านยาก debug ยาก

---

### Strategy vs Decorator

หลายคนงงว่าต่างกันยังไง:

| | Strategy | Decorator |
|---|---------|-----------|
| **จุดประสงค์** | เปลี่ยน algorithm ทั้งตัว | เพิ่ม behavior ให้ algorithm เดิม |
| **จำนวน** | ใช้ทีละ 1 ตัว | ซ้อนกันได้หลายตัว |
| **ตัวอย่าง** | เปลี่ยนจาก Kerry เป็น Flash | เพิ่ม logging + retry ให้ Kerry |

ใน blog บทความ [Log Middleware ตอนที่ 2](../laravel/log-request-response-laravel-middleware-events/log-http-requests-middleware-2.md) จะเห็นตัวอย่างการใช้ Decorator ใน Laravel จริงๆ ครับ

---

### สรุป

* Decorator **wrap object ด้วย object ที่มี interface เดียวกัน**
* เพิ่ม behavior **โดยไม่แก้ class เดิม**
* **เลือก combine** ได้ตามต้องการ — ไม่ต้องสร้าง subclass ทุก combination
* ลำดับ wrap สำคัญ — ตัวนอกรันก่อน
* เจอได้ทั่วไปใน middleware, stream filters, HTTP clients
