---
description: เอา Project Golang ของท่าน Build Image ด้วย Docker และ Deploy ขึ้น AWS Lambda
coverY: 0
---

# 🦉 ส่ง Project Golang ขึ้น Lambda

ภารกิจของเราคือ

* [สร้าง Project ด้วย Golang](project-golang-lambda.md#project-golang)
* [Build Image ด้วย Docker และ Upload image ขึ้น AWS ECR Repository](project-golang-lambda.md#build-image-docker-upload-image-registry-aws-ecr)
* [สร้าง Lambda Function](project-golang-lambda.md#lambda-function-container-image)
* [สร้าง API Gateway เพื่อใช้งาน Lambda Function](project-golang-lambda.md#api-gateway-lambda-function)



**เตรียม Project ของคุณเอง**

ในที่นี้เราใช้ Golang เป็น Project นะครับ

> สำหรับท่านยังใหม่เรื่อง Docker อยู่ เราต้องขออภัยด้วยจริงๆครับ
>
> ผมแนะนำให้ท่านอ่านไปดูจากที่อื่นๆก่อนนะครับ เพราะบทความนี้อาจจะเล่ากว้างๆหน่อย ไม่ได้เจาะลงลึกมาก

หรือสำหรับใครที่ยังไม่รู้จักเจ้า Lambda นะครับสามารถไปดูได้ตามลิงค์นี้เลยครับ

[aws-lambda.md](../aws/aws-lambda.md "mention")



เอาล่ะ เรามาเริ่มภารกิจแรก



### **สร้าง Project Golang**

Project ของเราจะใช้ Fiber framework เพื่อจัดการ HTTP requests เนื่องจาก Fiber มีประสิทธิภาพสูงและ syntax ที่เข้าใจง่าย

เริ่มต้นด้วยการสร้าง module และติดตั้ง dependencies:

```bash
go mod init my-golang-project
go get github.com/gofiber/fiber/v2
```

สร้างไฟล์ main.go ด้วย code ต่อไปนี้:

```go
package main

import (
	"context"
	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
	"github.com/gofiber/fiber/v2"
	"github.com/valyala/fasthttp"
)

var fiberApp *fiber.App

func init() {
    fiberApp = fiber.New()
    
    fiberApp.Get("/api/v1/example", func(c *fiber.Ctx) error {
        return c.JSON(fiber.Map{
            "message": "Hello from Lambda!",
            "status": "success",
        })
    })
}

func handler(ctx context.Context, req events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    // Convert API Gateway request to fasthttp request
    fctx := &fasthttp.RequestCtx{}
    fctx.Request.SetRequestURI(req.Path)
    fctx.Request.Header.SetMethod(req.HTTPMethod)
    
    // Process the request through Fiber
    err := fiberApp.Handler()(fctx)
    if err != nil {
        return events.APIGatewayProxyResponse{
            StatusCode: 500,
            Body:      "Internal Server Error",
        }, nil
    }
    
    // Convert fasthttp response to API Gateway response
    return events.APIGatewayProxyResponse{
        StatusCode: fctx.Response.StatusCode(),
        Body:      string(fctx.Response.Body()),
        Headers: map[string]string{
            "Content-Type": string(fctx.Response.Header.ContentType()),
        },
    }, nil
}

func main() {
    lambda.Start(handler)
}
```

อัพเดท go.mod เพื่อเพิ่ม dependencies ที่จำเป็น:

```go
module my-golang-project

go 1.21

require (
    github.com/aws/aws-lambda-go v1.46.0
    github.com/gofiber/fiber/v2 v2.52.0
    github.com/valyala/fasthttp v1.51.0
)
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



3. ต่อไป นำ image tag ที่คุณได้มาติด tag สำหรับ "เตรียมส่งขึ้น" `AWS ECR Repository`

เริ่มโดยคุณต้องไปสร้างที่จัดเก็บ image หรือเรียกอีกชื่อว่า Repository URI ก่อน ด้วยคำสั่ง

```
aws ecr create-repository \
    --repository-name my-golang-project \
    --region <region>
```

คุณก็จะได้ผลลัพท์ตามนี้

<pre class="language-json" data-line-numbers><code class="lang-json">{
    "repository": {
        "repositoryArn": "arn:aws:ecr:&#x3C;region>:&#x3C;aws_account_id>:repository/my-golang-project",
        "registryId": "&#x3C;aws_account_id>",
        "repositoryName": "my-golang-project",
<strong>        "repositoryUri": "&#x3C;aws_account_id>.dkr.ecr.&#x3C;region>.amazonaws.com/my-golang-project",
</strong>        "createdAt": "2024-02-03T16:00:14.640000+07:00",
        "imageTagMutability": "MUTABLE",
        "imageScanningConfiguration": {
            "scanOnPush": false
        },
        "encryptionConfiguration": {
            "encryptionType": "AES256"
        }
    }
}
</code></pre>

สังเกตบรรทัดที่ 6 นะครับ เราจะได้ repositoryUri มา ให้เรานำมาใช้ในการรันคำสั่งต่อไปนี้เพื่อติด tag

```bash
docker tag my-golang-project:latest \
    <aws_account_id>.dkr.ecr.<region>.amazonaws.com/my-golang-project:latest
```

อย่าลืมแก้ไข `<aws_account_id>` และ `<region>`&#x20;





4. ก่อนจะส่งขึ้น AWS ECR, คุณต้องให้ Docker ในเครื่องคุณสามารถเข้าถึง AWS ECR Repository ให้ได้ก่อน

โดยการรันคำสั่ง :

```bash
aws ecr get-login-password --region <region> \
    | docker login --username AWS --password-stdin \
    <aws_account_id>.dkr.ecr.<region>.amazonaws.com
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

3. ตั้งชื่อ function ในที่นี้เราตั้งชื่อ `myGolangProject`
4. ส่วนช่อง "Container image URI" , วาง URI ของ ECR ที่คุณเพิ่ง Upload ไป หรือกดที่ Browse images
5. กด "Create function".



หลังจากสร้างแล้ว,​ น้อง Lambda Function ของเราก็พร้อมทำงานแล้ว เย้+++++



แต่ๆๆๆๆๆๆ

น้องยังไม่มีหูเพื่อรับฟังคำร้องขอ Http (เส้นทางเข้าถึง Lambda)

เราต้องเพิ่มหูให้น้องก่อน



### ตั้งค่า API Gateway เพื่อเข้าถึง Lambda Function

เพื่อให้ต้นทางนั้นเรียกน้อง Lambda function ของคุณได้, เรามาจัดการ API Gateway กัน

1. ไปที่หน้า **Amazon API Gateway** ใน AWS Console.
2. กดปุ่ม **Create API** และเลือกเป็น **REST API**&#x20;
3. ในขั้นตอนตั้งค่า, ให้เลือก **New API** และใส่ชื่อ API ในที่นี้เราใส่ `my-golang-project-api`

<figure><img src="../.gitbook/assets/Screenshot 2024-02-03 at 21.39.22.png" alt=""><figcaption></figcaption></figure>

4. ในส่วนของการตั้งค่า Route, กำหนด HTTP Method และ Resource Path ตามภาพด้านล่าง

<figure><img src="../.gitbook/assets/Screenshot 2024-02-03 at 21.42.19.png" alt=""><figcaption></figcaption></figure>

5. ที่หน้า **Integrations**, เลือก Lambda Function ที่คุณสร้างไว้ก่อนหน้านี้เป็นต้นทางข้อมูล (ใส่ชื่อ Lambda Function หรือ ARN).

<figure><img src="../.gitbook/assets/Screenshot 2024-02-03 at 21.46.54.png" alt=""><figcaption></figcaption></figure>

6. เลือก "Create Resource"

<div align="center">

<figure><img src="../.gitbook/assets/Screenshot 2024-02-03 at 21.50.46.png" alt=""><figcaption></figcaption></figure>

</div>

7. ติ๊ก "Proxy resource", ส่วน Resource name ให้ใส่ `{proxy+}`

<div align="center">

<figure><img src="../.gitbook/assets/Screenshot 2024-02-03 at 21.51.50.png" alt=""><figcaption></figcaption></figure>

</div>

8. คลิกที่ ANY ล่าง {proxy+} (ตามรูป) ท่านจะเห็นข้อความเตือนด้านล่าง ให้กด "Edit integration"

<figure><img src="../.gitbook/assets/Screenshot 2024-02-03 at 21.52.21.png" alt=""><figcaption></figcaption></figure>

9. เลือก Lambda function และใส่ชื่อ Lambda ที่ใช้งาน

<figure><img src="../.gitbook/assets/Screenshot 2024-02-03 at 21.53.07.png" alt=""><figcaption></figcaption></figure>

10. กด Deploy ท่านจะพบกับ Dialog แจ้งเติอนให้ท่านสร้าง Stage เริ่มต้น

<figure><img src="../.gitbook/assets/Screenshot 2024-02-03 at 22.05.20.png" alt=""><figcaption></figcaption></figure>

<div>

<figure><img src="../.gitbook/assets/Screenshot 2024-02-03 at 22.05.45.png" alt=""><figcaption></figcaption></figure>

 

<figure><img src="../.gitbook/assets/Screenshot 2024-02-03 at 22.08.55.png" alt=""><figcaption></figcaption></figure>

</div>

เมื่อทุกอย่างเสร็จสิ้น

ท่านก็จะได้ Invoke URL ในส่วนของ "Stages" เป็นที่เรียบร้อยแล้ว

คุณสามารถเอา URL ที่ได้มาทดลองได้เลย



### การทดสอบ Lambda Function

หลังจาก deploy แล้ว คุณสามารถทดสอบ endpoint ได้ด้วย curl:

```bash
curl https://your-api-gateway-url/api/v1/example
```

ผลลัพธ์ที่ได้ควรจะเป็น:

```json
{
    "message": "Hello from Lambda!",
    "status": "success"
}
```

### การ Monitor และ Debug

Lambda function ของคุณจะส่ง logs ไปยัง CloudWatch โดยอัตโนมัติ คุณสามารถดู logs ได้ที่:

1. เข้าไปที่ CloudWatch ใน AWS Console
2. ไปที่ Log groups
3. ค้นหา log group ที่มีชื่อขึ้นต้นด้วย "/aws/lambda/myGolangProject"

สำหรับการ monitor performance คุณสามารถดูค่าต่างๆ ได้ที่:
- Execution duration
- Memory usage
- Concurrent executions
- Error count

