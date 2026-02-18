---
description: สร้าง object หลาย type จากจุดเดียว โดยไม่ต้อง if-else ยาวเหยียด
---

# Factory Pattern

### ปัญหาที่เจอ

สมมติเราเขียนระบบสร้าง notification ที่ส่งได้หลายช่องทาง:

```php
class NotificationController
{
    public function send(Request $request)
    {
        $channel = $request->input('channel');
        $message = $request->input('message');

        if ($channel === 'email') {
            $notifier = new EmailNotifier(
                config('mail.host'),
                config('mail.port'),
                config('mail.username'),
            );
        } elseif ($channel === 'sms') {
            $notifier = new SmsNotifier(
                config('sms.api_key'),
                config('sms.sender_id'),
            );
        } elseif ($channel === 'line') {
            $notifier = new LineNotifier(
                config('line.channel_token'),
            );
        } elseif ($channel === 'slack') {
            $notifier = new SlackNotifier(
                config('slack.webhook_url'),
                config('slack.channel'),
            );
        } else {
            throw new \Exception("Unknown channel: {$channel}");
        }

        $notifier->send($message);
    }
}
```

ปัญหา:

* **if-else ยาวขึ้นเรื่อยๆ** ทุกครั้งที่เพิ่มช่องทางใหม่
* **โค้ดสร้าง object ซ้ำ** ถ้ามี controller อื่นที่ต้องสร้าง notifier เหมือนกัน
* **Controller รู้มากเกินไป** — รู้ว่าแต่ละ notifier ต้อง config อะไร

---

### Factory Pattern มาช่วย

หลักการคือ **ย้ายโค้ดสร้าง object ไปอยู่ที่เดียว** ให้คนอื่นแค่บอกว่าอยากได้อะไร ไม่ต้องรู้วิธีสร้าง

มี 2 แบบที่ใช้บ่อย:

1. **Simple Factory** — class เดียว มี method สร้าง object ตาม parameter
2. **Factory Method** — ให้ subclass ตัดสินใจว่าจะสร้าง object อะไร

เราจะดูทั้ง 2 แบบครับ

---

## Simple Factory

### Step 1: สร้าง Interface

```php
<?php

namespace App\Contracts;

interface Notifier
{
    public function send(string $to, string $message): void;
    public function channel(): string;
}
```

### Step 2: สร้าง Implementations

```php
<?php

namespace App\Notifications;

use App\Contracts\Notifier;

class EmailNotifier implements Notifier
{
    public function __construct(
        private string $host,
        private int $port,
    ) {
        //
    }

    public function send(string $to, string $message): void
    {
        // ส่ง email ผ่าน SMTP
    }

    public function channel(): string
    {
        return 'email';
    }
}

class SmsNotifier implements Notifier
{
    public function __construct(
        private string $apiKey,
    ) {
        //
    }

    public function send(string $to, string $message): void
    {
        // ส่ง SMS ผ่าน API
    }

    public function channel(): string
    {
        return 'sms';
    }
}

class LineNotifier implements Notifier
{
    public function __construct(
        private string $channelToken,
    ) {
        //
    }

    public function send(string $to, string $message): void
    {
        // ส่ง Line message
    }

    public function channel(): string
    {
        return 'line';
    }
}
```

### Step 3: สร้าง Factory

```php
<?php

namespace App\Notifications;

use App\Contracts\Notifier;

class NotifierFactory
{
    /**
     * สร้าง Notifier จากชื่อ channel
     */
    public static function make(string $channel): Notifier
    {
        return match ($channel) {
            'email' => new EmailNotifier(
                host: config('mail.host'),
                port: config('mail.port'),
            ),
            'sms' => new SmsNotifier(
                apiKey: config('sms.api_key'),
            ),
            'line' => new LineNotifier(
                channelToken: config('line.channel_token'),
            ),
            default => throw new \InvalidArgumentException(
                "Unknown channel: {$channel}. Available: email, sms, line"
            ),
        };
    }

    /**
     * ดูว่ามี channel อะไรบ้าง
     */
    public static function available(): array
    {
        return ['email', 'sms', 'line'];
    }
}
```

### Step 4: ใช้งาน

```php
class NotificationController
{
    public function send(Request $request)
    {
        $notifier = NotifierFactory::make($request->input('channel'));
        $notifier->send($request->input('to'), $request->input('message'));

        return response()->json(['sent_via' => $notifier->channel()]);
    }
}
```

สะอาดขึ้นเยอะเลยครับ Controller ไม่ต้องรู้ว่า EmailNotifier ต้องใช้ host กับ port อะไร

💡 **Best Practice:** ใช้ `match` แทน `if-else` ครับ (PHP 8.0+) อ่านง่ายกว่า แล้วถ้าไม่ตรงกับ case ไหน จะ throw error ให้อัตโนมัติ

---

## Factory Method

Simple Factory เหมาะกับกรณีง่ายๆ แต่ถ้า logic การสร้างซับซ้อนขึ้น ใช้ Factory Method ดีกว่าครับ

หลักการคือ **กำหนด interface สำหรับการสร้าง object แล้วให้ subclass ตัดสินใจว่าจะสร้างอะไร**

### สร้าง Abstract Factory

```php
<?php

namespace App\Export;

abstract class ReportExporter
{
    /**
     * Factory method — subclass ต้อง implement
     */
    abstract protected function createFormatter(): ReportFormatter;

    /**
     * Template method — logic หลักอยู่ตรงนี้
     */
    public function export(array $data): string
    {
        $formatter = $this->createFormatter();

        $header = $formatter->formatHeader($data['title']);
        $body = $formatter->formatBody($data['rows']);
        $footer = $formatter->formatFooter($data['summary']);

        return $header . $body . $footer;
    }
}
```

### สร้าง Interface สำหรับ Product

```php
<?php

namespace App\Export;

interface ReportFormatter
{
    public function formatHeader(string $title): string;
    public function formatBody(array $rows): string;
    public function formatFooter(string $summary): string;
    public function extension(): string;
}
```

### สร้าง Concrete Products

```php
class CsvFormatter implements ReportFormatter
{
    public function formatHeader(string $title): string
    {
        return "# {$title}\n";
    }

    public function formatBody(array $rows): string
    {
        $lines = [];
        foreach ($rows as $row) {
            $lines[] = implode(',', $row);
        }
        return implode("\n", $lines) . "\n";
    }

    public function formatFooter(string $summary): string
    {
        return "# {$summary}\n";
    }

    public function extension(): string
    {
        return 'csv';
    }
}

class HtmlFormatter implements ReportFormatter
{
    public function formatHeader(string $title): string
    {
        return "<html><body><h1>{$title}</h1>";
    }

    public function formatBody(array $rows): string
    {
        $html = '<table>';
        foreach ($rows as $row) {
            $html .= '<tr>';
            foreach ($row as $cell) {
                $html .= "<td>{$cell}</td>";
            }
            $html .= '</tr>';
        }
        $html .= '</table>';
        return $html;
    }

    public function formatFooter(string $summary): string
    {
        return "<p>{$summary}</p></body></html>";
    }

    public function extension(): string
    {
        return 'html';
    }
}
```

### สร้าง Concrete Factories

```php
class CsvReportExporter extends ReportExporter
{
    protected function createFormatter(): ReportFormatter
    {
        return new CsvFormatter();
    }
}

class HtmlReportExporter extends ReportExporter
{
    protected function createFormatter(): ReportFormatter
    {
        return new HtmlFormatter();
    }
}
```

### ใช้งาน

```php
$data = [
    'title' => 'Monthly Sales Report',
    'rows' => [
        ['Product A', '150', '฿45,000'],
        ['Product B', '230', '฿69,000'],
    ],
    'summary' => 'Total: ฿114,000',
];

// export เป็น CSV
$exporter = new CsvReportExporter();
$csv = $exporter->export($data);

// export เป็น HTML
$exporter = new HtmlReportExporter();
$html = $exporter->export($data);
```

สังเกตนะครับ **logic การ export (`export()`) เหมือนกันทุก format** แค่ formatter ต่างกัน ถ้าจะเพิ่ม PDF ก็สร้าง `PdfFormatter` + `PdfReportExporter` จบ ไม่ต้องแก้โค้ดเดิม

---

### ทำให้ดีขึ้น — Registry Pattern

ถ้ามี factory เยอะ ใช้ registry จัดการได้:

```php
class NotifierRegistry
{
    /** @var array<string, callable(): Notifier> */
    private array $factories = [];

    /**
     * ลงทะเบียน factory
     */
    public function register(string $channel, callable $factory): void
    {
        $this->factories[$channel] = $factory;
    }

    /**
     * สร้าง Notifier
     */
    public function make(string $channel): Notifier
    {
        if (!isset($this->factories[$channel])) {
            throw new \InvalidArgumentException("Unknown channel: {$channel}");
        }

        return ($this->factories[$channel])();
    }

    public function available(): array
    {
        return array_keys($this->factories);
    }
}

// ลงทะเบียน
$registry = new NotifierRegistry();

$registry->register('email', fn () => new EmailNotifier(
    config('mail.host'),
    config('mail.port'),
));

$registry->register('sms', fn () => new SmsNotifier(
    config('sms.api_key'),
));

// เพิ่มช่องทางใหม่ได้ง่ายมาก
$registry->register('discord', fn () => new DiscordNotifier(
    config('discord.webhook_url'),
));

// ใช้งาน
$notifier = $registry->make('email');
$notifier->send('user@example.com', 'Hello!');
```

💡 **Best Practice:** Registry ดีกว่า Simple Factory เมื่อ:
* ต้อง register จากหลาย package (plugin system)
* ไม่อยากแก้ factory ทุกครั้งที่เพิ่มของใหม่
* ต้องการให้ config ได้ง่าย (เช่น enable/disable channel)

---

### เมื่อไหร่ควรใช้ Factory

✅ **ใช้เมื่อ:**
* สร้าง object หลาย type ที่ implement interface เดียวกัน
* Logic การสร้าง object ซับซ้อน (ต้อง config เยอะ)
* ไม่อยากให้ client รู้ว่า concrete class คือตัวไหน
* ต้องการจุดเดียวสำหรับเปลี่ยน implementation

❌ **ไม่ควรใช้เมื่อ:**
* สร้าง object แค่ตัวเดียว ที่ไม่มี variation — `new` ตรงๆ ง่ายกว่า
* Object ไม่มี shared interface — factory ไม่มีประโยชน์

---

### Factory ในงานจริง

เจอ pattern นี้ทุกที่ครับ:

* **Laravel** — `Cache::store('redis')`, `DB::connection('mysql')`, `Storage::disk('s3')`
* **Logger** — `Log::channel('slack')` สร้าง logger ตาม channel
* **Payment** — สร้าง payment gateway จาก config
* **Serializer** — สร้าง serializer ตาม format (JSON, XML, YAML)

---

### สรุป

* **Simple Factory** — class เดียว `match/switch` สร้าง object ตาม parameter เหมาะกับกรณีไม่ซับซ้อน
* **Factory Method** — abstract class ให้ subclass ตัดสินใจ เหมาะกับ logic ที่ซับซ้อนกว่า
* **Registry** — ลงทะเบียน factory แบบ dynamic เหมาะกับ plugin system
* ใช้ `match` แทน `if-else` ใน PHP 8.0+
* ถ้า object มีแค่ type เดียว ไม่ต้องใช้ factory — `new` ตรงๆ ง่ายกว่า
