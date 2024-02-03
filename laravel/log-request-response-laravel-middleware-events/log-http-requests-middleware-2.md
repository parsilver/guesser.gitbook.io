---
description: Requirement เปลี่ยนว่ะเพื่อน ทำไงดี
---

# Log http ทุกๆ requests ผ่าน Middleware - ตอนที่ 2

พอดีหัวหน้าบอกว่า อยากให้เช็คด้วยว่า "เก็บเฉพาะที่ POST มาเท่านั้นจ๊ะ, GET ไม่อยากได้"



โอเครครับ, วิธีแก้ปัญหาเราคืออะไรครับ\
ทุกคนอาจจะตอบเหมือนกันหมดว่า&#x20;

"ก็ไปแก้ที่ไฟล์ `app/Features/HttpLogger/DatabaseLogger.php` เลยสิ"

ก็เพิ่ม if method === "POST" ก็สิ้นเรื่อง

```php
<?php

namespace App\Features\HttpLogger;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use App\Contracts\HttpLogger as HttpLoggerContract;

class DatabaseLogger implements HttpLoggerContract
{
    /**
     * Write log
     */
    public function write(Request $request, $response)
    {
        if (! $request->method("POST")) {
            return
        }

        DB::table('http_log_requests')->insert([
            // Do someting...
        ]);
    }
}
```

จบเแล้วครับ

แต่คุณๆครับๆ, แบบนี้ ถ้าเราเพิ่มวิธีเก็บแบบอื่นขึ้นมา เราต้องเอา condition นี้ไปด้วย จริงมั้ยครับ

.....

"ใช่ครับ"

มันจะไม่จบง่ายๆ และมีปัญหาในอนาคตครับแน่นอนครับ เพราะตรงนี้มัน Data Layer !!!

เราต้องเอา business logic ออกไปครับ, ​ไปเขียนที่ middleware ยังได้เลยครับ



แต่เราทำงานเป็นทีม, เราไม่ทำงั้นครับ

เพราะเราไม่รู้อนาคตว่า requirement เราจะเพิ่มอะไรมาอีก หรือทำมาแล้ว "ไม่เอา" ซะงั้น

<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

งั้นก็ใช้ design pattern: decorator แล้วกัน

เพราะตัวที่ใช้เขียน log เราก็ไม่อยากไปแตะมันเท่าไหร่ เพราะไม่รู้ว่าทีมเราคนอื่นๆอาจจะมีวิธีเขียน log เพิ่มขึ้นมา



งั้นเรามาสร้างตัว decorator กันครับ

```php
<?php

namespace App\Features\HttpLogger;

use Illuminate\Http\Request;
use Illuminate\Http\Response;
use App\Contracts\HttpLogger as HttpLoggerContract;

class WriteOnlyPostMethodHttpLoggerDecorator implements HttpLoggerContract
{
    protected $logger;

    public function __construct(HttpLogger $logger)
    {
        $this->logger = $logger;
    }

    public function write(Request $request, $response)
    {
        if ($this->shouldLog($request, $response)) {
            $this->logger->write($request, $response);
        }
    }

    protected function shouldLog(Request $request, $response)
    {
        return $request->method('post');
    }
}

```

ที่นี้, เราไปแก้ไขเพิ่มเติมที่ app/Providers/AppServiceProvider.php

```php
<?php

namespace App\Providers;

use App\Contracts;
use Illuminate\Support\ServiceProvider;
use App\Features\HttpLogger\WriteOnlyPostMethodHttpLoggerDecorator;
use App\Features\HttpLogger\DatabaseLogger;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register services.
     *
     * @return void
     */
    public function register()
    {
        $this->app->bind(
             Contracts\HttpLogger::class, 
             function () {
                // เรียกใช้งาน Logger ตัวเดิม
                $logger = $this->app->make(DatabaseLogger::class);
                
                // ใช้งานตัว decorator เข้าไปแทน
                return new WriteOnlyPostMethodHttpLoggerDecorator($logger);
             }
        );
    }
}
```

เพียงเท่านี้ คุณก็จะไม่โดนทีมคุณด่าแล้วครับว่า "พังอีกแล้วววว, เพิ่มอะไรมาอี๊กกกก"&#x20;

หรือ "แก้ interface class ก็ช่วยแจ้งกันก่อนนิดนึงเน้อออ"

