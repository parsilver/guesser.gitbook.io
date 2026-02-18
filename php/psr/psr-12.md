---
description: มาตรฐานการจัดรูปแบบโค้ด PHP ที่ทุกคนควรใช้ — แทนที่ PSR-2 ที่ deprecated แล้ว
---

# PSR-12 Extended Coding Style

> 📌 PSR-12 เป็นมาตรฐานที่ใช้แทน PSR-2 ตั้งแต่ปี 2019 ถ้าคุณเคยอ่าน PSR-2 มาก่อน ให้ลืมไปได้เลยครับ ใช้ PSR-12 แทน

### ทำไมต้อง PSR-12?

เคยเจอไหมครับ ทำงานเป็นทีม แล้วเปิดโค้ดของเพื่อนมาดู... อ่านไม่รู้เรื่องเลย

ไม่ใช่เพราะโค้ดมันยาก แต่เพราะ **แต่ละคนเขียนคนละ style** — คนนึงใช้ tab คนนึงใช้ space, คนนึงเปิดปีกกา `{` บรรทัดเดียวกับ function คนนึงขึ้นบรรทัดใหม่

PSR-12 มาแก้ปัญหานี้ครับ มันกำหนดว่า "โค้ด PHP ที่ดี ควรจัดรูปแบบยังไง" ให้เป็นมาตรฐานเดียวกัน

---

### 1. พื้นฐานที่ต้องรู้ก่อน

ก่อนจะไปเรื่องอื่น ให้จำ 3 อย่างนี้ไว้เลยครับ

* **ใช้ 4 spaces** สำหรับ indentation (ห้ามใช้ tab เด็ดขาด)
* **1 บรรทัดไม่ควรเกิน 120 ตัวอักษร** (soft limit) แต่พยายามอย่าเกิน 80 จะดีที่สุด
* **ท้ายไฟล์ต้องมีบรรทัดว่าง 1 บรรทัด** และห้ามมี `?>` ปิดท้าย

```php
<?php

namespace App\Services;

use App\Contracts\PaymentGateway;
use App\Models\Order;

class PaymentService
{
    // โค้ดของเรา...
}
// ← บรรทัดว่างท้ายไฟล์
```

💡 **Best Practice:** ตั้งค่า editor ให้ใส่ newline ท้ายไฟล์อัตโนมัติเลยครับ ใน VS Code ตั้ง `"files.insertFinalNewline": true`

---

### 2. การประกาศ Namespace และ Use

ลำดับการเขียนด้านบนของไฟล์ ต้องเป็นแบบนี้เสมอครับ

```php
<?php

// 1. declare strict_types (ถ้ามี)
declare(strict_types=1);

// 2. namespace
namespace App\Services;

// 3. use statements (เรียงตามตัวอักษร)
use App\Contracts\Logger;
use App\Models\User;
use Illuminate\Support\Collection;

// 4. use function (ถ้ามี)
use function App\Helpers\format_money;

// 5. use const (ถ้ามี)
use const App\Config\MAX_RETRY;
```

สังเกตนะครับ **ต้องมีบรรทัดว่างคั่นระหว่างแต่ละกลุ่ม** เสมอ

💡 **Best Practice:** ใช้ `declare(strict_types=1);` ทุกไฟล์เลยครับ มันช่วยจับ bug เรื่อง type ได้เยอะมาก

---

### 3. Class, Property และ Method

มาถึงส่วนที่เขียนบ่อยที่สุดครับ

#### 3.1 การประกาศ Class

```php
<?php

namespace App\Services;

use App\Contracts\Cacheable;
use App\Contracts\Loggable;

class UserService extends BaseService implements Cacheable, Loggable
{
    use HasLogger;
    use HasCache;

    // constants ก่อน
    public const MAX_LOGIN_ATTEMPTS = 5;

    // properties
    private string $defaultRole = 'member';

    // constructor
    public function __construct(
        private readonly UserRepository $userRepo,
        private readonly Logger $logger,
    ) {
        //
    }

    // methods
    public function findById(int $id): ?User
    {
        return $this->userRepo->find($id);
    }
}
```

จุดสำคัญ:

* `extends` และ `implements` อยู่บรรทัดเดียวกับชื่อ class
* เปิด `{` ขึ้นบรรทัดใหม่ (ต่างจาก if/for ที่อยู่บรรทัดเดียวกัน)
* ลำดับในคลาส: `use` traits → constants → properties → constructor → methods

#### 3.2 ถ้า implements เยอะมาก

```php
class UserService extends BaseService implements
    Cacheable,
    Loggable,
    Serializable
{
    // ...
}
```

ขึ้นบรรทัดใหม่แล้ว indent เข้ามา 1 ชั้นครับ

---

### 4. Control Structures (if, for, switch)

ส่วนนี้คนเขียนผิดกันเยอะมากครับ มาดูทีละตัวเลย

#### 4.1 If / Elseif / Else

```php
if ($user->isActive()) {
    $this->sendWelcomeEmail($user);
} elseif ($user->isPending()) {
    $this->sendReminder($user);
} else {
    $this->notifyAdmin($user);
}
```

สังเกตนะครับ:

* เปิด `{` **อยู่บรรทัดเดียวกัน** (ไม่เหมือน class ที่ขึ้นบรรทัดใหม่)
* ใช้ `elseif` ไม่ใช่ `else if` (คำเดียว)
* มี space หลัง `if` และก่อน `{` เสมอ

#### 4.2 Switch

```php
switch ($status) {
    case 'active':
        $this->activate($user);
        break;

    case 'suspended':
        $this->suspend($user);
        break;

    default:
        $this->deactivate($user);
        break;
}
```

💡 **Best Practice:** ถ้า case ไหนตั้งใจจะ fall-through (ไม่มี break) ให้ comment `// no break` ไว้ด้วยครับ เพื่อให้คนอ่านรู้ว่าไม่ได้ลืม

```php
case 'admin':
    $this->grantAdminAccess($user);
    // no break
case 'moderator':
    $this->grantModAccess($user);
    break;
```

#### 4.3 For / Foreach / While

```php
for ($i = 0; $i < $count; $i++) {
    // ...
}

foreach ($users as $user) {
    // ...
}

while ($retry < self::MAX_RETRY) {
    // ...
}
```

ทุกตัวเหมือนกันครับ — เปิด `{` บรรทัดเดียวกัน, มี space รอบ keywords

---

### 5. Method และ Function Parameters

#### 5.1 Parameter สั้นๆ

```php
public function createUser(string $name, string $email): User
{
    // ...
}
```

#### 5.2 Parameter ยาวๆ (เกิน 120 ตัวอักษร)

```php
public function createOrder(
    int $userId,
    string $productCode,
    int $quantity,
    ?string $couponCode = null,
): Order {
    // ...
}
```

สังเกตนะครับ:

* parameter แต่ละตัวขึ้นบรรทัดใหม่ indent เข้า 1 ชั้น
* ปิด `)` กับเปิด `{` อยู่บรรทัดเดียวกัน
* **trailing comma** หลังตัวสุดท้ายได้ (PHP 8.0+)

💡 **Best Practice:** ใช้ trailing comma เสมอครับ เวลาเพิ่ม parameter ใหม่ git diff จะสะอาดมาก

---

### 6. Closure (Anonymous Function)

```php
$filtered = array_filter($users, function (User $user) {
    return $user->isActive();
});
```

ถ้ามี `use`:

```php
$greeting = function (string $name) use ($prefix, $suffix): string {
    return $prefix . $name . $suffix;
};
```

#### Arrow Function (PHP 7.4+)

```php
$filtered = array_filter($users, fn (User $user) => $user->isActive());
```

สั้นกว่าเยอะเลยครับ ถ้า logic ไม่ซับซ้อน แนะนำใช้ arrow function

---

### 7. Return Type Declaration

```php
// return type ปกติ
public function getUser(int $id): User
{
    //
}

// nullable return type
public function findUser(int $id): ?User
{
    //
}

// union types (PHP 8.0+)
public function getId(): int|string
{
    //
}

// void
public function sendEmail(User $user): void
{
    //
}
```

💡 **Best Practice:** ใส่ return type ทุก method เลยครับ มันช่วยให้ IDE autocomplete ได้ดีขึ้น และจับ bug ได้เร็วขึ้นมาก

---

### 8. Operators

เรื่องเล็กแต่สำคัญครับ — **ต้องมี space รอบ operator เสมอ**

```php
// ✅ ถูก
$total = $price * $quantity;
$name = $firstName . ' ' . $lastName;
$isValid = $age >= 18 && $hasConsent === true;

// ❌ ผิด
$total=$price*$quantity;
$name=$firstName.' '.$lastName;
```

ยกเว้น **string concatenation** ที่มีหลายบรรทัด ให้จุด `.` อยู่ต้นบรรทัดถัดไป:

```php
$query = 'SELECT * FROM users'
    . ' WHERE active = 1'
    . ' ORDER BY created_at DESC';
```

---

### 9. Nullable Type และ Union Type

```php
// nullable parameter
public function setNickname(?string $nickname): void
{
    //
}

// union type (PHP 8.0+)
public function process(int|float $amount): void
{
    //
}

// intersection type (PHP 8.1+)
public function handle(Countable&Iterator $items): void
{
    //
}
```

สังเกตนะครับ `?` ติดกับ type ไม่มี space, `|` และ `&` ก็ไม่มี space เช่นกัน

---

### 10. ตั้งค่าให้จัดการอัตโนมัติ

เรื่องทั้งหมดนี้ไม่ต้องจำหมดครับ เพราะมีเครื่องมือช่วย

#### PHP CS Fixer (แนะนำ)

```bash
composer require --dev friendsofphp/php-cs-fixer
```

สร้างไฟล์ `.php-cs-fixer.php` ที่ root ของโปรเจกต์:

```php
<?php

$finder = PhpCsFixer\Finder::create()
    ->in(__DIR__ . '/app')
    ->in(__DIR__ . '/config')
    ->in(__DIR__ . '/routes');

return (new PhpCsFixer\Config())
    ->setRules([
        '@PSR12' => true,
        'array_syntax' => ['syntax' => 'short'],
        'no_unused_imports' => true,
        'ordered_imports' => ['sort_algorithm' => 'alpha'],
        'trailing_comma_in_multiline' => true,
    ])
    ->setFinder($finder);
```

แล้วรัน:

```bash
# ดูก่อนว่าจะแก้อะไรบ้าง
vendor/bin/php-cs-fixer fix --dry-run --diff

# แก้เลย
vendor/bin/php-cs-fixer fix
```

💡 **Best Practice:** ตั้ง pre-commit hook ให้รัน php-cs-fixer อัตโนมัติทุกครั้งที่ commit ครับ จะได้ไม่ต้องมานั่งเถียงเรื่อง style กันในทีม

#### Laravel Pint (สำหรับ Laravel)

ถ้าใช้ Laravel อยู่แล้ว ใช้ Pint ได้เลยครับ มันมาพร้อม Laravel ตั้งแต่ v9:

```bash
# ดูก่อน
./vendor/bin/pint --test

# แก้เลย
./vendor/bin/pint
```

Pint ใช้ PSR-12 เป็น default อยู่แล้ว สะดวกมากครับ

---

### สรุป

ถ้าให้จำแค่ไม่กี่ข้อ จำแค่นี้ก็พอครับ:

* **4 spaces** ไม่ใช่ tab
* **เปิด `{`** — class/method ขึ้นบรรทัดใหม่, control structure อยู่บรรทัดเดียวกัน
* **Space** รอบ operator, หลัง keyword (`if`, `for`, `foreach`)
* ใช้ `elseif` ไม่ใช่ `else if`
* **Return type** ใส่ทุก method
* **ใช้ PHP CS Fixer หรือ Pint** จัดการให้อัตโนมัติ — อันนี้สำคัญที่สุด

จำไม่หมดก็ไม่เป็นไรครับ แค่ตั้ง tool ให้จัดการให้ แล้วโค้ดเราก็จะเป็นมาตรฐานเดียวกับคนทั้งโลกแล้ว 😄
