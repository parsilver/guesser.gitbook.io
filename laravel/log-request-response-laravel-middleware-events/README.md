# วิธีการเก็บ Log Request และ Response ใน Laravel ผ่าน Middleware และ Events

โดยปกติแล้ว พอเราสร้าง project ขึ้นมาแล้ว หนึ่งในส่ิงต่อไปที่เราต้องทำนั้น เราจะนึกขึ้นได้อัตโนมัติเลยก็คือเรื่อง&#x20;

"การเก็บ log request, response"

> "แล้วเราจะใช้วิธีไหนดีหล่ะ, เขียนไว้ใน middleware หรอ?"

คำตอบคือ "ใช่" แหละครับ

> "อ่าว แล้วมันมีวิธีอื่นด้วยหรอ?"

"ครับ"



โดยที่ส่วนใหญ่เลยทาง Dev ก็เขียนกันบน Middleware กันทั้งนั้นแหละครับ&#x20;

เพียงแต่ว่ามันก็มีวิธีอื่นที่เค้าใช้กันอีก

ซึ่งในบทความต่อจากนี้ ผมจะแนะนำสองวิธีใหญ่ๆที่เค้าใช้กันนะครับ

* [เขียน log ผ่าน middleware](log-requests-middleware-1.md)
* เขียน log ผ่าน events

