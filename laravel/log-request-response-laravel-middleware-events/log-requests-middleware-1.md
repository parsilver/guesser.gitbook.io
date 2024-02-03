---
description: เริ่มต้นวิธีการเขียน Log http requests ผ่านตัว Middleware
---

# เก็บ Log ทุกๆ requests ผ่าน Middleware - ตอนที่ 1

[จากบทความก่อนหน้า](./)

เอาจริงๆแล้วมันเป็นวิธีที่ง่าย และ simple ที่สุดแล้วครับ เพราะเราสามารถจัดการได้ที่เดียวเลย และแก้ไขได้ง่าย

มาดูขั้นตอนการสร้างกันครับ

### 1. Middleware

คือระบบที่เราสามารถใช้เพื่อจัดการ request และ response ใน Laravel ซึ่งเราสามารถใช้เพื่อเก็บ log ได้

สร้าง middleware ใหม่ด้วยคำสั่ง:

{% code fullWidth="false" %}
```bash
php artisan make:middleware LogRequestMiddleware
```
{% endcode %}

เราก็จะได้ไฟล์ app/Http/Middleware/LogRequestMiddleware.php มีหน้าตาอย่างงี้

{% code title="app/Http/Middleware/LogRequestMiddleware.php" lineNumbers="true" fullWidth="false" %}
```php
<?php

namespace App\Http\Middleware;

use Illuminate\Http\Request;

class LogRequestMiddleware
{
    /**
     * Handle incoming request
     */
    public function handle(Request $request, \Closure $next)
    {
        //

        return $next($request);
    }
}
```
{% endcode %}

แล้วเอา Middleware ตัวนี้ไปลงทะเบียนลำดับการทำงานที่ `app/Http/Kernel.php`

เช่นตัวอย่างนี้

{% code title="app/Http/Kernel.php" %}
```php
<?php

namespace App\Http;

use Illuminate\Foundation\Http\Kernel as HttpKernel;

class Kernel extends HttpKernel
{
    /**
     * The application's global HTTP middleware stack.
     *
     * These middleware are run during every request to your application.
     *
     * @var array
     */
    protected $middleware = [
       // .....
       \App\Http\Middleware\LogRequestMiddleware::class,
    ];
    
    // ......
}
```
{% endcode %}



มาอธิบาย middleware กันหน่อย

ดูจากตัวแปลที่รับเข้ามาใน method `handler` มันจะมีแค่ `$request` และ `$next`

ซึ่งคำสั่ง $next นั้น เอาไว้กระทำกับ middleware ตัวถัดไป จนกว่าจะเสร็จทุกขั้นตอน&#x20;

(ดูได้ที่ app/Http/Kernel.php)

เรามาปรับรูปแบบใหม่เพื่อเตรียมทำ log เราก็จะได้หน้าตาแบบนี้

```php
/**
 * Handle incoming request
 */
public function handle(Request $request, \Closure $next)
{
   /** @var \Illuminate\Http\Request $response */
   $response = $next($request);
   
   // Do something...
   
   return $response;
}
```

ถ้าเราเขียน log จากตรงนี้มันก็ทำได้แหละครับ, แต่มันจะมีปัญหาตามมาคือ\
"บางที method handle ตรงนี้อาจจะมาไม่ถึงหากมี request พังก่อนหน้า"



ดังนั้นเรามาดูตรงนี้กันครับ

เพิ่ม method `terminate` เข้ามา ซึ่ง method นี้จะทำงานก็ต่อเมื่อ request นั้นสิ้นสุดลง

พอเพิ่มแล้วเราก็จะได้หน้าตาแบบนี้ครับ

```php
<?php

namespace App\Http\Middleware;

use Illuminate\Http\Request;
use App\Contracts\HttpLogger;

class LogRequestMiddleware
{
    /**
     * Handle incoming request
     */
    public function handle(Request $request, \Closure $next)
    {
        return $next($request);
    }
    
    /**
     * Handle response
     */
    public function terminate($request, $response)
    {
        //
    }
}
```

มันก็จะดักได้ง่ายขึ้น เพราะเราไม่ต้องกังวลตัว middleware ที่อยู่ก่อนหน้าตัวนี้

ทีนี้ เรามาออกแบบตัวเขียน log กันครับ

### 2. ออกแบบตัวเขียน Log

ประเด็นเลยคือ พอเราจะเขียน log แต่ละทีหนิ, เราชอบนึกเรื่องที่มันดูซับซ้อนก่อนเลยคือ "เขียนลง DB ไหนดีหล่ะ ถึงจะเร็ว"

ผมจะบอกว่า "เลิกคิดเถอะครับ" เพราะสุดท้าย คุณต้องหาสิ่งที่ดีกว่าที่สุดอยู่เรื่อยๆ

ดังนั้น, ใช้ Strategy Pattern ละกัน

โจทย์คือ

* ต้องเก็บทุก request, method, url, header, user agent, ip address, payload
* เก็บทุก response, status code, response header, response body

เริ่ม

สร้างไฟล์ `app/Contracts/HttpLogger.php`

{% code title="app/Contracts/HttpLogger.php" lineNumbers="true" %}
```php
<?php

namespace App\Contracts;

use Illuminate\Http\Request;
use Illuminate\Http\Response;

interface HttpLogger
{
    /**
     * Write log
     * 
     * @param \Illuminate\Http\Request $request
     * @param \Illuminate\Http\Response $response
     */
    public function write(Request $request, $response);
}
```
{% endcode %}

แล้วทำไปใช้งานที่ middleware ได้เลย ก็จะได้หน้าตาแบบนี้

{% code title="" lineNumbers="true" %}
```php
<?php

namespace App\Http\Middleware;

use Illuminate\Http\Request;
use App\Contracts\HttpLogger;

class LogRequestMiddleware
{
    public function __construct(
        private HttpLogger $logger,
    )
    {
        //
    }
    
    /**
     * Handle incoming request
     */
    public function handle(Request $request, \Closure $next)
    {
        return $next($request);
    }
    
    /**
     * Handle response
     */
    public function terminate($request, $response)
    {
        $this->logger->write($request, $response);
    }
}
```
{% endcode %}

"อย่าเพิ่งไป Run นะ, พังนะ"

### 3. เขียน Log ลง Database

"คิดอะไรไม่ออก ก็เขียน log ลง db ตัวเองแบบง่ายๆนี่แหละ"

สร้างไฟล์ `app/Features/HttpLogger/DatabaseLogger.php`

{% code title="app/Features/HttpLogger/DatabaseLogger.php" lineNumbers="true" %}
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
        DB::table('http_log_requests')->insert([
            // Do someting...
        ]);
    }
}
```
{% endcode %}

พอเราได้ทุกอย่างที่ต้องการแล้ว, เราก็เอาไปลงทะเบียนที่ Provider กันครับ

ในที่นี่ ผมเอาไว้ที่ `app/Providers/AppServiceProvider.php`

```php
<?php

namespace App\Providers;

use App\Contracts;
use Illuminate\Support\ServiceProvider;

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
             \App\Features\HttpLogger\DatabaseLogger::class,
        );
    }
}
```

เป็นอันจบครับ สำหรับการเขียน log ผ่าน middleware
