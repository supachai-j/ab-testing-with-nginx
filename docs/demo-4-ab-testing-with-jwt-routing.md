## Demo-4: A/B Testing with JWT.

4.1) ทำการแก้ไขไฟล์ [docker-compose.yml](../docker-compose.yml) ดังนี้
```yaml
version: "3.7"
services:
    nginx:
        image: nginx-plus:latest
        restart: always
        ports:
          - 80:80
        volumes:
          - ./conf/ab-testing-with-jwt-routing.conf:/etc/nginx/nginx.conf
          - ./conf/api_secret.jwk:/etc/nginx/api_secret.jwk
```
จากสั่ง run ก็ใช้คำสั่ง 
```sh
docker-compose up -d
```
ถ้าอยากดู logs แบบ realtime ของ nginx บน container ใช้คำสั่ง
```sh
docker-compose logs -f
```
4.2) ในการทดสอบนี้จะอ้างอิง nginx configuration จากไฟล์  [ab-testing-with-jwt-routing.conf](../conf/ab-testing-with-jwt-routing.conf) นี้ 
```nginx
# JWT validation
auth_jwt "JWT Test Realm" token=$arg_myjwt;
auth_jwt_key_file /etc/nginx/api_secret.jwk;
```
และมีไฟล์​ [api_secret.jwt](../conf/api_secret.jwt) 
```json
{"keys":
    [{
        "k":"bmdpbngxMjM",
        "kty":"oct"
    }]
}
```
> กำหนดค่า k คือ ``symmetric key`` เป็น ``nginx123`` โดย encode ด้วย base64 จากคำสั่งนี้ 
```sh
echo -n nginx123 | base64 | tr '+/' '-_' | tr -d '='
```
![encode-base64](images/demo-4-1.png)
4.3) ทำการสร้าง jwt token แบบง่ายๆ โดยไปที่เว็บ [https://jwt.io](https://jwt.io/) โดยกำหนดรายละเอียดดังนี้

Header:
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
Payload:
```json
{
  "exp": 1659812957,
  "name": "Create New User",
  "sub": "cuser",
  "gname": "wheel",
  "guid": "10",
  "fullName": "John Doe",
  "uname": "jdoe",
  "uid": "111",
  "sudo": true,
  "dept": "IT",
  "url": "http://secure.example.com"
}
```
Verify Signature:
```sh
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),nginx123
)
```
ตัวอย่าง ที่ใช้ทดสอบ จะกำหนด ``"uid":"111"`` 
![jwt-uid-111](images/demo-4-2.png)
และ ``"uid":"222"`` ดังนี้
![jwt-uid-222](images/demo-4-3.png)

4.4) ทดสอบเรียกโดย ```uid = 111```  ส่ง jwt token ผ่านทาง ``myjwt`` บน URI 
```sh
curl -i http://localhost/\?myjwt\=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NTk4MTI5NTcsIm5hbWUiOiJDcmVhdGUgTmV3IFVzZXIiLCJzdWIiOiJjdXNlciIsImduYW1lIjoid2hlZWwiLCJndWlkIjoiMTAiLCJmdWxsTmFtZSI6IkpvaG4gRG9lIiwidW5hbWUiOiJqZG9lIiwidWlkIjoiMTExIiwic3VkbyI6dHJ1ZSwiZGVwdCI6IklUIiwidXJsIjoiaHR0cDovL3NlY3VyZS5leGFtcGxlLmNvbSJ9.WAKIXeopm8TVeDSMiDi3M1CY-4l4cvZZO5iIlzayBP4
```
![curl-with-jwt-uid111](images/demo-4-4.png)
ทดสอบเรียกโดย โดย ```uid = 222``` ส่ง jwt token ผ่านทาง ``myjwt`` บน URI 
```sh
curl -i http://localhost/?myjwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NTk4MTI5NTcsIm5hbWUiOiJDcmVhdGUgTmV3IFVzZXIiLCJzdWIiOiJjdXNlciIsImduYW1lIjoid2hlZWwiLCJndWlkIjoiMTAiLCJmdWxsTmFtZSI6IkpvaG4gRG9lIiwidW5hbWUiOiJqZG9lIiwidWlkIjoiMjIyIiwic3VkbyI6dHJ1ZSwiZGVwdCI6IklUIiwidXJsIjoiaHR0cDovL3NlY3VyZS5leGFtcGxlLmNvbSJ9.j3kaqRXkbkJrCqTlWw0iKxlcgUVR_z3f5CLTaWPPS3M
```
![curl-with-jwt-uid222](images/demo-4-5.png)

4.5) ลองทำการส่ง jwt token ผ่านทาง Cookie กัน ลองเปลี่ยนตรง ``token`` จาก ``$arg_myjwt`` เป็น ``$cookie_myjwt``
```nginx
# JWT validation

# auth_jwt "JWT Test Realm" token=$arg_myjwt;
auth_jwt "JWT Test Realm" token=$cookie_myjwt;
auth_jwt_key_file /etc/nginx/api_secret.jwk;

```
และทดสอบเรียกด้วย Cookie ดูครับ
```sh
curl -v --cookie myjwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NTk4MTI5NTcsIm5hbWUiOiJDcmVhdGUgTmV3IFVzZXIiLCJzdWIiOiJjdXNlciIsImduYW1lIjoid2hlZWwiLCJndWlkIjoiMTAiLCJmdWxsTmFtZSI6IkpvaG4gRG9lIiwidW5hbWUiOiJqZG9lIiwidWlkIjoiMTExIiwic3VkbyI6dHJ1ZSwiZGVwdCI6IklUIiwidXJsIjoiaHR0cDovL3NlY3VyZS5leGFtcGxlLmNvbSJ9.WAKIXeopm8TVeDSMiDi3M1CY-4l4cvZZO5iIlzayBP4 http://localhost
```
![curl-cookie-with-jwt-uid222](images/demo-4-6.png)

> ถึงตรงนี้ ถือว่า เสร็จสิ้น การทดสอบเรื่อง A/B Testing ด้วย jwt ครับ

## ส่งท้ายด้วย ในโหมด แอบซน และสงสัย****

0.1) ลองแกล้ง เปลี่ยนตัวอักษรบางตัว ``payload`` ของ jwt token ดูว่า NGINX สามารถตรวจสอบ หรือ Validate JWT Authentication ได้จริงไหม ? 
```sh
curl -i http://localhost/?myjwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NTk4MTI5NTcsIm5hbWUiOiJDcmVhdGUgTmV3IFVzZXIiLCJzdWIiOiJjdXNlciIsImduYW1lIjoid2hlZWwiLCJndWlkIjoiMTAiLCJmdWxsTmFtZSI6IkpvaG4gRG9lIiwidW5hbWUiOiJqZG9lIiwidWlkIjoiMjIyIiwic3VkbyI6dHJ1ZSwiZGVwdCI6IklUIiwidXJsIjoiaHR0cDovL3NlY3VyZS5leGFtcGxlLmNvbSJ0.j3kaqRXkbkJrCqTlWw0iKxlcgUVR_z3f5CLTaWPPS3M
```
หมายเหตุ: ผมเปลี่ยนค่า ``J9`` เป็น ``J0`` ตรงท้าย payload jwt นะครับ เผื่อลืม format ของ jwt เป็น {``Header.Payload.Signature``}

ตัวอย่าง
![curl-edit-jwt-payload](images/demo-4-7.png)

0.2) ลองแกล้ง โดยทำการสร้าง jwt token ด้วย ``symmetric key`` เป็น ``nginx-community-thailand`` แต่ในไฟล์ ``api_secret.jwk`` ยังเหมือนเดิม.
![gen-jwt-newkey](images/demo-4-8.png)
```
curl -i http://localhost/?myjwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NTk4MTI5NTcsIm5hbWUiOiJDcmVhdGUgTmV3IFVzZXIiLCJzdWIiOiJjdXNlciIsImduYW1lIjoid2hlZWwiLCJndWlkIjoiMTAiLCJmdWxsTmFtZSI6IkpvaG4gRG9lIiwidW5hbWUiOiJqZG9lIiwidWlkIjoiMTExIiwic3VkbyI6dHJ1ZSwiZGVwdCI6IklUIiwidXJsIjoiaHR0cDovL3NlY3VyZS5leGFtcGxlLmNvbSJ9.n9IiB-zJ2HF9D1mb7dMRg0YoIA2uNI-JLtvkCV5lDv4
```
ตัวอย่าง
![curl-jwt-signature-dif](images/demo-4-9.png)

0.3) ลองแกล้ง โดยกำหนดค่า exp หรือ [Expiration time](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.4) ซึ่งต้องกำหนดค่าเป็น [seconds in unix epoch](https://en.wikipedia.org/wiki/Unix_time) ครับ โดยผมจะเปลี่ยนจาก ``1659812957 =>(2022-08-07 02:09:17)`` เป็น ``1517417999 => (2018:01:31 23:59:59)`` นั้นหมายความว่า Expire แล้ว

```sh
$ date -r '1659812957' '+%Y-%m-%d %H:%M:%S'
2022-08-07 02:09:17
```

```sh
$ date -j -f "%Y:%m:%d %H:%M:%S" '2018:01:31 23:59:59' +%s
1517417999
```
ตัวอย่าง 
![convert-date-time](images/demo-4-10.png)

ทำการสร้าง token ใหม่ลองดูจาก https://jwt.io
![jwt-gen-expired](images/demo-4-11.png)

ลองเรียกเข้าไปด้วย jwt ที่หมดอายุแล้ว
![curl-with-jwt-expired](images/demo-4-12.png)


หลังการทดสอบเสร็จแล้ว ก็ทำการลบ Container 
```sh
docker-compose down
```

 เรียบร้อยครับ จะเห็นว่า เราสามารถกำหนด กลุ่มของ A และ B ได้ ผ่านทาง JWT Claim ถ้าไม่ใช่ กลุ่ม A (uid=111) ก็ไปทาง B ทั้งหมด  

 ส่วนบทส่งท้าย คือ ผมอยากจำลอง หรือสร้างสถาณการณ์ต่างๆ ดูว่า ถ้าเกิดเหตุการณ์ต่างๆ แบบที่ผมสมมุติ ขึ้นแล้ว NGINX จะสามารถทำงานได้อย่างสมบูรณ์ หรือไม่ 
