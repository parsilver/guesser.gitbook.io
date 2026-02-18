# PSR คือ มาตรฐานการเขียนโค้ดที่ควรทราบ

PSR ย่อมาจาก PHP Standard Recommendation

เป็นกลุ่มของมาตรฐานการเขียนโค้ด PHP ที่ถูกสร้างขึ้นโดยกลุ่ม PHP-FIG (Framework Interoperability Group).

มาตรฐานเหล่านี้ถูกออกแบบมาเพื่อให้ผู้พัฒนาสามารถเขียนโค้ดที่สามารถทำงานร่วมกันได้ในระดับที่สูงขึ้น

และเพื่อให้โค้ดที่เขียนมามีความเข้ากันได้.

### ตัวอย่างของ PSR ที่ครอบคลุมในบล็อกนี้

#### PSR-1: Basic Coding Standard

มาตรฐานเริ่มต้นในการเขียนโค้ด เช่น ชื่อ class ต้องเป็น StudlyCaps, method ต้องเป็น camelCase, ใช้ tags `<?php` หรือ `<?=`

#### PSR-3: Logger Interface

มาตรฐานสำหรับ logger — กำหนด 8 log levels (emergency ถึง debug) ให้เปลี่ยน logger ได้โดยไม่ต้องแก้โค้ด Laravel ใช้ PSR-3 ผ่าน Monolog อยู่เบื้องหลัง

#### PSR-4: Autoloader

มาตรฐานสำหรับการโหลด class อัตโนมัติ ใช้กับ Composer ที่เราใช้กันอยู่ทุกวัน

#### PSR-11: Container Interface

มาตรฐานสำหรับ Dependency Injection Container — กำหนดว่าจะดึง service ออกมาจาก container ยังไง Laravel Service Container implement PSR-11 อยู่แล้ว

#### PSR-12: Extended Coding Style

มาตรฐานการจัดรูปแบบโค้ดที่ใช้กันในปัจจุบัน (แทนที่ PSR-2) ครอบคลุม indentation, spacing, brace placement และอื่นๆ

#### PSR-0: Autoloader (Deprecated)

มาตรฐาน autoloader ตัวเก่าที่ถูกแทนที่ด้วย PSR-4 แล้ว ไม่แนะนำให้ใช้ในโปรเจกต์ใหม่

### สรุปแล้ว ทำไมเราต้องใช้ PSR?

จริงๆมันก็ไม่จำเป็นต้องตามกันครับ แต่มันก็จะมีข้อดีอยู่หลายข้อด้วยจากข้างล่างนี้

* ทำให้ทำงานเป็นทีมได้
* โค้ดของคุณเข้ากับผู้พัฒนาผู้อื่นได้ หรือเข้ากับ Community ได้ เพราะเราอยู่ในมาตรฐานเดียวกันแล้ว
* ง่ายต่อการอ่าน
* จัดการ structure ได้ง่าย
* ง่ายต่อการบำรุงรักษา สามารถพัฒนาต่อยอดในอนาคตได้
