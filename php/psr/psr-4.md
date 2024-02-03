---
description: มาตรฐานการใหม่สำหรับการโหลดไฟล์อัตโนมัติ
---

# PSR-4 มาตรฐานการโหลดไฟล์อัตโนมัติ

เป็นมาตรฐานการโหลดอัตโนมัติสำหรับคลาส PHP

มันเกี่ยวข้องกับวิธีการบังคับให้คลาสและเนมสเปซควรตรงกับไดเรคทอรีและไฟล์ในระบบไฟล์&#x20;

ช่วยให้นักพัฒนาสามารถมั่นใจได้ว่าสามารถโหลดคลาสได้อย่างง่ายดาย ถูกต้องและอัตโนมัติในระหว่างการเรียกใช้โปรแกรมผ่าน autoloader กับ PSR-4.

```php
spl_autoload_register(function ($class) {
    // ฐานของ project (เช่น "src")
    $base_dir = __DIR__ . '/src/';

    // เนมสเปซของโปรเจ็ก (เช่น "MyProject")
    $prefix = 'MyProject\\';

    // ตรวจสอบว่าคลาสที่กำลังโหลดเริ่มต้นด้วย prefix หรือไม่
    $len = strlen($prefix);
    if (strncmp($prefix, $class, $len) !== 0) {
        return;
    }

    // หาชื่อคลาสโดยที่นำ prefix ออก
    $relative_class = substr($class, $len);

    // แทนที่ namespace separator ด้วย directory separator และเติม .php
    $file = $base_dir . str_replace('\\', '/', $relative_class) . '.php';

    // ถ้าไฟล์อยู่จริง, ก็โหลดขึ้นมา
    if (file_exists($file)) {
        require $file;
    }
});
```
