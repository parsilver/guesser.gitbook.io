---
description: เครื่องมือช่วยจัดการ Dependencies ของ Class
---

# Service Container

### Service Container คืออะไร

Service Container เป็นส่วนเครื่องมือหลักในโครงสร้างของ Laravel Framework&#x20;

ทำหน้าที่หลักในการจัดการการโยนออบเจ็กต์  (Dependency Injection) หรือเราเรียกกันว่าฉีดออบเจ็กต์นั่นแหละครับ และการจัดการลูปการเชื่อมโยงออบเจ็กต์ (Object Binding Lifecycle) ในระบบ&#x20;

การใช้งาน Service Container ทำให้ทาง Dev สามารถกำหนดและจัดการ dependencies ของ Class ได้อย่างมีประสิทธิภาพ ช่วยให้การออกแบบซอฟต์แวร์เป็นไปตามหลักการ Single Responsibility Principle และ Interface Segregation Principle ซึ่งเป็นส่วนหนึ่งของ SOLID principles

#### ประโยชน์หลักของ Service Container

1.  **การฉีดออบเจ็กต์อัตโนมัติ**

    สามารถจัดการ dependencies ของคลาสได้อัตโนมัติโดยไม่ต้องสร้างออบเจ็กต์ เช่นใน Code ชุดนี้

```php
use Illuminate\Support\ServiceProvider;
use App\PersonalWriter;

class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
        // กำหนดการผูก (Binding) ของ dependency โดยใช้เมธอด bind
        // ซึ่งทำให้สามารถสร้างอินสแตนซ์ของ PersonalWriter ได้เมื่อมีการร้องขอการใช้งาน
        $this->app->bind(PersonalWriter::class, function ($app) {
            return new PersonalWriter();
        });
    }
}

// ทดลองใช้ใน Controller
class ExampleController extends Controller
{
    protected $writer;

    // รับ PersonalWriter ผ่านทาง constructor 
    // ซึ่งช่วยให้คลาสสามารถใช้งานอินสแตนซ์ที่ต้องการได้โดยไม่ต้องสร้างเอง
    public function __construct(PersonalWriter $writer)
    {
        $this->writer = $writer;
    }
}
```



2.  **ความยืดหยุ่น**

    ช่วยให้โค้ดสามารถขยายหรือปรับแต่งได้ง่ายขึ้นโดยไม่ต้องแก้ไขในหลายๆ จุด

```php
use Illuminate\Support\ServiceProvider;
use App\ReportGeneratorInterface;
use App\MockReportGenerator;

class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
        // ผูกอินเทอร์เฟซกับ Mock Implementation ใช้ในการทดสอบ
        $this->app->bind(ReportGeneratorInterface::class, function ($app) {
            return new MockReportGenerator();
        });
    }
}

// Controller ที่ใช้ dependency
class ReportController extends Controller
{
    protected $reportGenerator;

    // รับ ReportGeneratorInterface ผ่าน constructor
    public function __construct(ReportGeneratorInterface $reportGenerator)
    {
        $this->reportGenerator = $reportGenerator;
    }

    public function generate()
    {
        return $this->reportGenerator->generateReport();
    }
}
```



3.  **การทดสอบ**

    ทำให้การทดสอบระดับหน่วย (Unit Testing) เป็นไปได้ง่ายขึ้น เพราะสามารถ mock การทำงานของ dependencies ได้ง่าย

```php
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;
use App\ReportGeneratorInterface;
use App\MockReportGenerator;

class ReportControllerTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_generate_report()
    {
        $this->mock(ReportGeneratorInterface::class, function ($mock) {
            $mock->shouldReceive('generateReport')->once()->andReturn('Mock Report');
        });

        $response = $this->get('/report');

        $response->assertStatus(200);
        $response->assertSee('Mock Report');
    }
}
```



4.  **ความสามารถในการกำหนดค่าเอง (Binding)**

    ทาง Dev สามารถผูกอินเทอร์เฟซกับการทำงานเฉพาะเจาะจงหรือกำหนดไปยังการทำงานที่ต่างออกไปได้ เช่น

```php
use App\Contracts\PaymentGatewayInterface;
use App\Services\StripePaymentGateway;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
        // Binding the interface to a specific implementation
        $this->app->bind(PaymentGatewayInterface::class, StripePaymentGateway::class);
    }
}
```

ในตัวอย่างนี้ เราจะผูก `PaymentGatewayInterface` กับ `StripePaymentGateway` ซึ่งทำให้ทุกครั้งที่มีการร้องขอ `PaymentGatewayInterface` จะได้รับอินสแตนซ์ของ `StripePaymentGateway` แทน



#### การทำงานพื้นฐาน

เมื่อมีการร้องขอออบเจ็กต์จาก Service Container, Container จะหาว่าออบเจ็กต์ขึ้นต่อ (Dependency) ใดต้องการและจะทำการสร้างออบเจ็กต์เหล่านั้นให้เอง นอกจากนี้ ยังสามารถกำหนดรูปแบบการผูก (Binding) โดยใช้ `bind` หรือ `singleton` เพื่อกำหนดว่าควรจะใช้ออบเจ็กต์เดียวกันหรือสร้างใหม่ทุกครั้งที่ร้องขอ

#### ตัวอย่างการใช้งาน

```php
use App\Services\UserService;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->bind(UserService::class, function ($app) {
            return new UserService($app->make(UserRepository::class));
        });
    }
}
```

ในตัวอย่างข้างต้น การใช้ Service Container ช่วยลดภาระในการจัดการ dependencies ของ `UserService` โดยสร้างและจัดการ `UserRepository` ที่จำเป็นสำหรับ `UserService` อัตโนมัติ





