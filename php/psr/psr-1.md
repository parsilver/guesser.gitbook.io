---
description: มาตรฐานโค้ต
---

# PSR-1 มาตรฐานของโค้ต

#### PSR-1: Basic Coding Standard

มีจุดประสงค์เพื่อสร้างความสอดคล้องกันในโค้ด PHP ทุกๆโปรเจ็ค และเพิ่มความเข้าใจง่ายในการอ่านโค้ดของนักพัฒนา&#x20;

เมื่อพิจารณาถึง PSR-1 นั้น โค้ดของนักพัฒนาควรจะปฏิบัติตามกฎบางประการ&#x20;

เช่น&#x20;

* ไฟล์ทั้งหมดจะต้องใช้แท็ก `<?php` หรือ `<?=` และไม่ควรใช้แท็กปิด `?>`.

```php
<?php

namespace Vendor\Package;

class ClassName
{
    const VERSION = '1.0';
    const DATE_APPROVED = '2012-06-01';

    public function methodName($arg1, $arg2)
    {
        // method body
    }
}

// ต้องไม่มีแท็กปิด
```



* ควรใช้การเข้ารหัส UTF-8 กับ Code ทั้งหมด.
* คลาสนั้นควรถูกประกาศใน `StudlyCaps`.

```php
class ExampleClass
{
    const MAX_USERS = 10;
    const DEFAULT_TIMEOUT = 300;

    public function getUserProfile($userId, $profileId)
    {
        // ...
    }
}
```

* ชื่อไฟล์ควรเป็นชื่อเดียวกันกับชื่อคลาสที่อยู่ภายในไฟล์นั้นโดยตรง และควรตั้งชื่อไฟล์ด้วยการใช้การเขียนแบบ StudlyCaps เช่น `"MyClass.php"`.
* ชื่อ Class, Constant และ Method ควรประกาศอยู่ใน Namespace ตามกฎ PSR-1.
