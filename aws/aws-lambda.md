---
description: >-
  คิดระบบเล็กๆได้ แต่ไม่รู้จะเอาขึ้นที่ไหน เน้นเอางานขึ้น ไม่เน้นคนเข้าใช้ แนะนำ
  AWS Lambda ช่วยคุณได้
---

# 😅 AWS Lambda

มาเริ่มกันที่เจ้า Lambda คืออะไรก่อน



**AWS Lambda**&#x20;

เป็นบริการการแบบ serverless ซึ่งช่วยให้คุณสามารถรัน Code Project ของคุณได้โดยไม่ต้องกังวลเรื่องการจัดการโครงสร้างพื้นฐาน เช่น เซิร์ฟเวอร์ หรือคลัสเตอร์&#x20;

คุณเพียงแค่อัพโหลดโค้ดของคุณเข้า Lambda จากนั้นเจ้า Lambda ก็จะดูแลระบบของคุณทุกอย่างเริ่มตั้งแต่การจัดการทรัพยากรไปจนถึงการเตรียมความพร้อมเพื่อรับการร้องขอหรือตอบสนองต่อเหตุการณ์ต่างๆ.



**คุณสมบัติหลักของ AWS Lambda:**

*   **โครงสร้างพื้นฐานที่จัดการได้แบบอัตโนมัติ:**&#x20;

    AWS Lambda จัดการ CPU, memory, และอื่นๆ ให้กับ Code Project ของคุณโดยอัตโนมัติ.
*   **คิดและคำนวนเงินตามการใช้งานจริง:**&#x20;

    ซึ่งมีระบบคิดเงินที่เรียกว่า "pay-as-you-go" คุณจะชำระเงินเฉพาะเวลาที่ Code Project  ของคุณเรียกใช้งาน.
*   **การตอบสนองต่อเหตุการณ์:**&#x20;

    Lambda สามารถทำงานเป็นการตอบสนองต่อเหตุการณ์จากบริการ AWS อื่นๆ เช่น S3, DynamoDB, Kinesis, และอีกมากมาย.
*   **การทำงานร่วมกับแอปพลิเคชันและบริการอื่น:**&#x20;

    สามารถนำมาใช้เป็น backend สำหรับแอปพลิเคชันเว็บหรือการบริการผ่าน API Gateway.



**ตัวอย่างการใช้งาน AWS Lambda:**

*   **Web Applications:**&#x20;

    การใช้งาน AWS Lambda เพื่อการดำเนินโค้ดที่เกิดจาก HTTP request ผ่าน Amazon API Gateway.
*   **Data Processing:**&#x20;

    Lambda สามารถประมวลผลข้อมูลแบบ real-time เช่น จากรูปภาพหรือวิดีโอที่ถูกอัพโหลดและไฟล์ log จากการเรียกใช้งาน.
*   **IoT Backends:**&#x20;

    ใช้ Lambda ในการจัดการและตอบสนองต่อเหตุการณ์จากอุปกรณ์ IoT.



**ตัวอย่างวิธีคำนวนการใช้งาน**

ซึ่งคำนวณตามจำนวนครั้งที่ฟังก์ชันของคุณถูกเรียกและระยะเวลาที่ Code Project ของคุณกำลังทำงาน&#x20;

โดยคิดเฉพาะทรัพยากรที่ใช้งานจริง

เช่นๆๆ

* สมมติฟังก์ชัน Lambda ของคุณใช้ memory 512MB&#x20;
* เวลาทำงานเป็นเวลา 120ms ต่อการเรียกใช้
* ถูกเรียกใช้ 2 ล้านครั้งหลังจากที่ได้รับฟรี 1 ล้านครั้ง

จะเท่ากับว่า

* X = ราคาต่อ 1 ล้านครั้ง
* Y = ราคาต่อระยะเวลา 100ms
* จำนวนครั้งที่เรียก = 2,000,000 calls \* $X per 1M calls
* เวลาทำงาน = (120,000 seconds / 100) \* $Y per 100ms
* Free Tier ของ AWS Lambda ประกอบด้วย คำขอฟรีหนึ่งล้านคำขอต่อเดือน และเวลาประมวลผล 400,000 GB-วินาทีต่อเดือน



ท่านสามารถดูข้อมูลเพิ่มเติมได้ที่ [https://aws.amazon.com/th/lambda/](https://aws.amazon.com/th/lambda/)
