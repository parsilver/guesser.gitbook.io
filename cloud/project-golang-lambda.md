---
description: เอา Project Golang ของท่าน Build Image ด้วย Docker และ Deploy ขึ้น AWS Lambda
coverY: 0
---

# 🦉 ส่ง Project Golang ขึ้น Lambda

ภารกิจของเราคือ

* สร้าง Project ด้วย Golang
* Build Image ด้วย Docker และ Upload image ขึ้น AWS ECR Repository
* สร้าง [Lambda Function](../aws/aws-lambda.md)
* สร้าง API Gateway เพื่อใช้งาน [Lambda Function](../aws/aws-lambda.md)



**เตรียม Project ของคุณเอง**

ในที่นี้เราใช้ Golang เป็น Project นะครับ

> สำหรับท่านยังใหม่เรื่อง Docker อยู่ เราต้องขออภัยด้วยจริงๆครับ
>
> ผมแนะนำให้ท่านอ่านไปดูจากที่อื่นๆก่อนนะครับ เพราะบทความนี้อาจจะเล่ากว้างๆหน่อย ไม่ได้เจาะลงลึกมาก



เอาหล่ะ เรามาเริ่มภารกิจแรก



### **สร้าง Project Golang**

ผมจะยกตัวอย่าง Code ง่ายๆ ที่เรียกใช้งานผ่าน Http Request ได้เลย

```go
package main

import (
	"github.com/gofiber/fiber/v2"
)

func main() {
	app := fiber.New()

	app.Get("/api/v1/example", func(c *fiber.Ctx) error {
		return c.SendString("Hello, World!")
	})

	app.Listen(":3000")
}
```

ส่วนโครงสร้างของ Folder structure project จะมีตามนี้

```
.
├── go.mod
├── go.sum
├── main.go
```



### **Build Image ด้วย Docker และ Upload image ขึ้น Registry (AWS ECR)**

1. ทำการสร้าง Dockerfile&#x20;

```
.
├── Dockerfile <-- ตรงนี้
├── go.mod
├── go.sum
├── main.go
```

มี Code ตามนี้

```docker
# เรียกใช้แม่แบบจาก golang:1.21
FROM golang:1.21 as builder

# ตั้ง Folder ทำงานหลัก
WORKDIR /app

# คัดลอก Dependencies versioning
COPY go.mod .
COPY go.sum .

# ติดตั้ง Dependencies
RUN go mod download

# คัดลอก Project มายัง Image
COPY . .

# Build 
RUN CGO_ENABLED=0 go build -o /binary

# ------------
# เปลี่ยน Image แม่แบบ เพื่อลดขนาด Image ลง
FROM scratch

# คัดลอก binary จากผลลัพท์ของ image ก่อนหน้า
COPY --from=builder /binary /binary

# ตั้ง Command ตั้งต้น
ENTRYPOINT ["/binary"]
```



2. เมื่อคุณได้ `Dockerfile` แล้ว ต่อไปเราจะทำการ Build Image ด้วย Docker

โดยการรันคำสั่ง :

```bash
docker build -t my-golang-project .
```

Command นี้จะทำการ Build Image&#x20;

โดยอิงจาก Dockerfile จาก Folder ปัจจุบัน และติด Tag `my-golang-project`



3. ต่อไป นำ image tag ที่คุณได้มาติด tag สำหรับ "เตรียมส่งขึ้น" `AWS ECR Repository`:

```bash
docker tag my-golang-project:latest <aws_account_id>.dkr.ecr.<region>.amazonaws.com/my-golang-project:latest
```

อย่าลืมแก้ไข `<aws_account_id>` และ `<region>`&#x20;



4. ก่อนจะส่งขึ้น AWS ECR, คุณต้องให้ Docker ในเครื่องคุณสามารถเข้าถึง AWS ECR Repository ให้ได้ก่อน

โดยการรันคำสั่ง :

```bash
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
```



5. หลังจากทำให้ Docker ของคุณเข้าสู่ระบบได้แล้ว, ต่อมาให้รันคำสั่งนี้เพื่อ Upload image ของคุณไปยัง ECR:

```bash
docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/my-golang-project:latest
```



### สร้าง Lambda Function จาก Container Image

หลังจาก Image ของคุณอยู่บน ECR แล้ว, ต่อมาเราจะต้องสร้าง Server ให้กับมัน (Lambda Function)

1. ในหน้า AWS Console, ไปส่วนของ AWS Lambda console.
2. เลือก "Create function" และเลือก "Container image" เพื่อใช้งาน Project เราเป็นต้นแบบ

<figure><img src="../.gitbook/assets/Screenshot 2024-02-03 at 15.07.29.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/Screenshot 2024-02-03 at 15.11.23.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/Screenshot 2024-02-03 at 15.12.03.png" alt=""><figcaption></figcaption></figure>

3. ตั้งชื่อ function
4. ส่วนช่อง "Container image URI" , วาง URI ของ ECR ที่คุณเพิ่ง Upload ไป

> \<aws\_account\_id>.dkr.ecr..amazonaws.com/my-golang-project:latest

5. กด "Create function".



หลังจากสร้างแล้ว,​ น้อง Lambda Function ของเราก็พร้อมทำงานแล้ว เย้+++++



แต่ๆๆๆๆๆๆ

น้องยังไม่มีหูเพื่อรับฟังคำร้องขอ Http (เส้นทางเข้าถึง Lambda)

เราต้องเพิ่มหูให้น้องก่อน



### ตั้งค่า API Gateway เพื่อเข้าถึง Lambda Function

เพื่อให้ต้นทางนั้นเรียกน้อง Lambda function ของคุณได้, เรามาจัดการ API Gateway กัน

#### การตั้งค่า API Gateway สำหรับ Lambda Function

หลังจากที่เราสร้าง Lambda Function ด้วย Container Image เรียบร้อยแล้ว ต่อไปเราจะสร้าง API Gateway เพื่อทำการอนุญาตให้มีการเรียกใช้งานฟังก์ชันนี้ผ่าน HTTP Request.

1. ไปที่หน้า **Amazon API Gateway** ใน AWS Console.
2. กดปุ่ม **Create API** และเลือก **HTTP API** หรือ **REST API** ขึ้นอยู่กับความต้องการของคุณ.
3. ในขั้นตอนตั้งค่า, ให้เลือก **New API** และใส่ชื่อ API.
4. ในส่วนของการตั้งค่า Route, กำหนด HTTP Method และ Resource Path ที่คุณต้องการ.
5. ที่หน้า **Integrations**, เลือก Lambda Function ที่คุณสร้างไว้ก่อนหน้านี้เป็นต้นทางข้อมูล (ใส่ชื่อ Lambda Function หรือ ARN).
6. ตามตัวเลือกส่วนอื่นๆ ที่จำเป็นต้องมีในการตั้งค่า และกด **Create** เมื่อเสร็จสิ้นทุกขั้นตอน.

เมื่อทุกอย่างเสร็จสิ้น, API Gateway ของคุณจะกลายเป็น 'หู' ที่พร้อมรับฟังและส่งต่อคำร้องขอไปยัง Lambda Function ที่คุณได้จัดกำหนดไว้ข้างต้น.

