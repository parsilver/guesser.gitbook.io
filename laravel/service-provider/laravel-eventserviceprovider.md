---
description: ผู้ให้บริการเรื่องการรับสารต่างๆจากพระเจ้า
---

# 🙋♀ \[Laravel] EventServiceProvider

EventServiceProvider ใน Laravel เป็นคลาสที่จัดการกับ event และ listener ภายในแอปพลิเคชัน

มันเป็นส่วนหนึ่งของ ServiceProvider ทั้งหลายใน Laravel ที่ช่วยให้คุณสามารถลงทะเบียน events และ listeners ได้ตามความต้องการ&#x20;



ตัวอย่างภายใน Code ของ EventServiceProvider

```php
// app/Providers/EventServiceProvider.php

namespace App\Providers;

use Illuminate\Auth\Events\Registered;
use Illuminate\Auth\Listeners\SendEmailVerificationNotification;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;
use Illuminate\Support\Facades\Event;

class EventServiceProvider extends ServiceProvider
{
    /**
     * The event to listener mappings for the application.
     *
     * @var array<class-string, array<int, class-string>>
     */
    protected $listen = [
        Registered::class => [
            SendEmailVerificationNotification::class,
        ],
    ];

    /**
     * Register any events for your application.
     */
    public function boot(): void
    {
        //
    }

    /**
     * Determine if events and listeners should be automatically discovered.
     */
    public function shouldDiscoverEvents(): bool
    {
        return false;
    }
}
```

#### อธิบายการทำงาน EventServiceProvider

* &#x20;`$listen` ซึ่งเป็น array ที่ใช้สำหรับ map ระหว่าง event กับ listeners  เมื่อ event "ถูกยิง", Laravel จะตรวจสอบ array นี้และเรียกใช้ listener ภายใต้ event นั้นๆ
* method `boot` เป็นส่วนที่ใช้เพื่อกำหนดการทำงานเพิ่มเติมเมื่อ service provider นี้ถูกเรียกใช้งาน
* method `shouldDiscoverEvents` ที่เมื่อคืนค่าเป็น `false` จะทำให้ Laravel ไม่ไปค้นหา events และ listeners ให้อัตโนมัติ ทำให้คุณต้องลงทะเบียนทั้งหมดเองอย่างชัดเจนใน array `$listen`.

