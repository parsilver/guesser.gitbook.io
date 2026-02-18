---
description: ทำความเข้าใจ Service Provider ใน Laravel — วิธีใช้งาน register(), boot() พร้อมตัวอย่างจริง
---

# 🌞 Service Provider คืออะไร ?

> 📌 บทความนี้เขียนสำหรับ Laravel 10.x (ใช้ได้กับ Laravel 11 เช่นกัน)

**ServiceProvider**&#x20;

ใน Laravel มันเป็นส่วนหนึ่งของระบบที่เราสามารถจัดการระบบและสร้างสิ่งที่เรียกว่า **"ผู้ให้บริการ"** ให้กับเราได้



เช่น&#x20;

* บริการจัดการ container
* บริการเตรียมสิ่งต่างๆก่อนการทำงานของระบบ หรือการกำหนดค่าต่างๆ
* อื่นๆ



ซึ่ง มันเป็นส่วนหลักของการทำงานใน Laravel Application ของท่านทั้งหมด

ทุกครั้งที่ Laravel ถูกเรียกใช้งาน เจ้าตัว ServiceProvider ทั้งหมดจะถูกโหลดใช้งานทุกครั้ง เพื่อทำการลงทะเบียน หรือเตรียมชุดข้อมูลของท่าน เพื่อพร้อมสำหรับการทำงานของระบบของท่านต่อไป



นี่คือตัวอย่างไฟล์จาก `app/Providers/AppServiceProvider.php`

{% code title="app/Providers/AppServiceProvider.php" %}
```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     *
     * @return void
     */
    public function register()
    {
        // ตรงนี้คุณสามารถลงทะเบียน หรือเตรียมการข้อมูลของท่านได้
    }

    /**
     * Bootstrap any application services.
     *
     * @return void
     */
    public function boot()
    {
        // ตรงนี้จะทำงานเมื่อระบบต่างๆ หรือ providers ต่างๆถูกลงทะเบียนแล้ว
    }
}

```
{% endcode %}

ซึ่งตัวไฟล์นี้มันจะถูกลงทะเบียนให้เรียกใช้งานที่ config/app.php

ท่านสามารถดูเพิ่มเติมได้ที่ [https://laravel.com/docs/providers](https://laravel.com/docs/providers)



เอาหละ เรามาลองยกตัวอย่างซักหน่อย


### ตัวอย่างการใช้งาน ServiceProvider เบื้องต้น

เมื่อท่านต้องการให้ระบบของท่านมีการถือข้อมูลอะไรก็ตาม เช่น ค่าของ version ของระบบ หรือ ข้อมูลอื่นๆ ที่ท่านต้องการให้ระบบของท่านมีการเรียกใช้งานได้ง่ายๆ

{% code title="app/Providers/AppServiceProvider.php" %}
```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     *
     * @return void
     */
    public function register()
    {
        // ตรงนี้คุณสามารถลงทะเบียน หรือเตรียมการข้อมูลของท่านได้
        $this->app->bind('my-app-version', function () {
            return '1.0.0';
        });
    }

    /**
     * Bootstrap any application services.
     *
     * @return void
     */
    public function boot()
    {
        //
    }
}

```
{% endcode %}


อธิบายโค้ดด้านบนเบื้องต้นจาก Lifecycle ของ Laravel

1. ตอนที่ Laravel ถูกเรียกใช้งาน ตัว AppServiceProvider จะถูกโหลดขึ้นมา
2. ตอนที่ AppServiceProvider ถูกโหลดขึ้นมา ตัว `register()` จะถูกเรียกใช้งาน
3. ใน `register()` ท่านสามารถลงทะเบียน หรือเตรียมข้อมูลต่างๆ ได้
4. ในตัวอย่างด้านบน เราได้ลงทะเบียน `my-app-version` ให้กับ container ของ Laravel แล้ว
5. ตอนที่มีการเรียกใช้งาน `my-app-version` จะได้ค่า `1.0.0` กลับมา

เช่น ใน Controller หรือ Blade Template ท่านสามารถเรียกใช้งานได้ดังนี้

{% code title="app/Http/Controllers/MyController.php" %}
```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class MyController extends Controller
{
    public function index()
    {
        $version = app('my-app-version'); // <-- ได้ค่า '1.0.0' กลับมา

        return $version;
    }
}

```
{% endcode %}


### ข้อแตกต่างระหว่าง `register()` และ `boot()`

* `register()` จะถูกเรียกใช้งานก่อนที่ Laravel Application จะถูกโหลดขึ้นมา ซึ่งใช้สำหรับการลงทะเบียน หรือเตรียมข้อมูลต่างๆ ให้กับ container ของ Laravel
  
* `boot()` จะถูกเรียกใช้งานหลังจากที่ Laravel Application ถูกโหลดขึ้นมาแล้ว ซึ่งใช้สำหรับการทำงานต่างๆ หลังจากที่ทุกอย่างถูกโหลดขึ้นมาแล้ว


### Reference

* [https://laravel.com/docs/providers](https://laravel.com/docs/providers)