---
description: ผู้ให้บริการเรื่องการรับสารต่างๆจากพระเจ้า
---

# 🙋♀ EventServiceProvider

> สำหรับ Laravel >= 11.0 เป็นต้นไป เค้าเอาออกแล้วนะครับ

EventServiceProvider ใน Laravel เป็นคลาสที่จัดการกับ event และ listener ภายในแอปพลิเคชัน

มันเป็นส่วนหนึ่งของ [ServiceProvider](./) ทั้งหลายใน Laravel ที่ช่วยให้คุณสามารถลงทะเบียน events และ listeners ได้ตามความต้องการ&#x20;



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
* method `boot` เป็นส่วนที่ใช้เพื่อกำหนดการทำงานเพิ่มเติมเมื่อ [service provider](./) นี้ถูกเรียกใช้งาน
* method `shouldDiscoverEvents` ที่เมื่อคืนค่าเป็น `false` จะทำให้ Laravel ไม่ไปค้นหา events และ listeners ให้อัตโนมัติ ทำให้คุณต้องลงทะเบียนทั้งหมดเองอย่างชัดเจนใน array `$listen`.


#### ยกตัวอย่างการใช้งาน

ยกตัวอย่างเคส "ลูกค้าเรา ลงทะเบียนสำเร็จ" เราไม่รู้ว่าในอนาคตเราจะโจทย์อะไรเพิ่มเติมมั้ย 
อาจจะเป็นการส่งอีเมล์ยืนยันตัวตน หรืออะไรก็ตาม ในกรณีนี้เราสามารถใช้ EventServiceProvider ในการจัดการได้

  - สร้าง event ขึ้นมา

```bash
php artisan make:event CustomerRegistered
```

นี่คือตัวอย่างของ Event ที่เราสร้างขึ้นมา

{% code title="app/Events/CustomerRegistered.php" %}
```php
namespace App\Events;

use App\Models\Customer;

class CustomerRegistered
{
    public $customer;

    /**
     * Create a new event instance.
     *
     * @param  Customer  $customer
     * @return void
     */
    public function __construct(Customer $customer)
    {
        $this->customer = $customer;
    }
}
```
{% endcode %}


การใช้งาน Event นี้ สามารถทำได้โดยการส่ง event ไปให้กับระบบ เช่น

{% code title="app/Http/Controllers/Auth/CustomerRegistrationController.php" %}
```php

namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Models\Customer;
use App\Events\CustomerRegistered;

class CustomerRegistrationController extends Controller
{
    /**
     * Handle a registration request for the application.
     *
     * @return \Illuminate\Http\Response
     */
    public function register()
    {
        // ลงทะเบียนลูกค้า
        $customer = Customer::create(request()->all());

        // เราส่ง event ให้กับระบบไปก่อน
        event(new CustomerRegistered($customer));

        return response()->json($customer);
    }
}
```
{% endcode %}

ในที่นี้เราส่ง event `Registered` ไปให้กับระบบ เราเรียก Design Pattern นี้ว่า "Observer Pattern" 

ช่วยให้เราสามารถแยกการทำงานของแต่ละส่วนออกจากกันได้ และทำให้ระบบของเรามีความยืดหยุ่นมากขึ้น


> * ข้อควรระวัง เราไม่แนะนำให้ใช้หากเป็น Logic ที่สำคัญมากๆ เช่นการทำงานที่ต้องการความแม่นยำสูง เพราะมันอาจจะทำให้ระบบของเรามีความซับซ้อนขึ้นได้



   - หลังจากนั้น มี Requirement ใหม่เกิดขึ้นคือ 
เราต้องการส่งเมลแจ้งเตือนให้กับทาง Admin ว่ามีลูกค้าลงทะเบียนเข้ามาใหม่

เราสามารถสร้าง Listener ขึ้นมาเพื่อทำงานนี้ได้ โดยใช้คำสั่ง
```bash
php artisan make:listener SendEmailNotification
```

นี่คือตัวอย่างของ Listener ที่เราสร้างขึ้นมา

{% code title="app/Listeners/SendEmailNotification.php" %}
```php
namespace App\Listeners;

use App\Events\CustomerRegistered;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class SendEmailNotification implements ShouldQueue
{
    use InteractsWithQueue;

    /**
     * Handle the event.
     *
     * @param  CustomerRegistered  $event
     * @return void
     */
    public function handle(CustomerRegistered $event)
    {
        // ส่งอีเมล์ไปยัง Admin
        // โดยใช้ $event->customer
    }
}
```
{% endcode %}

เราสามารถลงทะเบียน Listener นี้ได้ใน EventServiceProvider ด้วย

{% code title="app/Providers/EventServiceProvider.php" %}
```php
namespace App\Providers;

use App\Events\CustomerRegistered;
use App\Listeners\SendEmailNotification;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;

class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        CustomerRegistered::class => [
            SendEmailNotification::class,
        ],
    ];

    public function boot(): void
    {
        parent::boot();
    }
}
```
{% endcode %}

หรือหากท่านใช้ Laravel >= 11.0 ขึ้นไป 
ท่านสามารถสร้าง Provider ขึ้นมาใหม่เพื่อลงทะเบียน Event และ Listener ได้ หรือ นำไปลงทะเบียนใน `app/Providers/AppServiceProvider.php` ก็ได้เช่นกัน

{% code title="app/Providers/AppServiceProvider.php" %}
```php
namespace App\Providers;

use App\Events\CustomerRegistered;
use App\Listeners\SendEmailNotification;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        //
    }

    public function boot(): void
    {
        // ลงทะเบียน Event และ Listener
        Event::listen(
            CustomerRegistered::class,
            SendEmailNotification::class
        );
    }
}
```
{% endcode %}


### Reference

- [https://laravel.com/docs/events](https://laravel.com/docs/events)