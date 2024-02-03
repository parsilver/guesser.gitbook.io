# Service Provider คืออะไร ?

ServiceProvider ใน Laravel มันเป็นส่วหนึ่งของระบบที่เราสามารถสร้าง "ผู้ให้บริการ" ได้

เช่น&#x20;

* บริการข้อมูล, events
* บริการจัดการ container
* บริการเตรียมสิ่งต่างๆก่อนการทำงานของระบบ หรือการกำหนดค่าต่างๆ
* อื่นๆ

ซึ่ง มันเป็นส่วนหลักของการทำงานใน laravel application ของท่านทั้งหมด

ทุกครั้งที่ laravel ถูกเรียกใช้งาน, ServiceProvider ทั้งหมดจะถูกโหลดใช้งานทุกครั้ง เพื่อทำการลงทะเบียน หรือเตรียมชุดข้อมูลของท่าน เพื่อพร้อมสำหรับการทำงานของระบบของท่านต่อไป



นี่คือตัวอย่างไฟล์จาก app/Providers/AppServiceProvider.php

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

ท่านสามารถดูเพิ่มเติมได้ที่ [https://laravel.com/docs/providers](https://laravel.com/docs/10.x/providers)

