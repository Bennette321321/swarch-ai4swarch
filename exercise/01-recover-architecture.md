**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก -> ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| `checkoutservice` -> `cartservice` | `checkoutservice.yaml (CART_SERVICE_ADDR)` | ✅ | `src/checkoutservice/main.go` บรรทัด 113 อ่าน env, บรรทัด 120 เชื่อม gRPC, เรียก `GetCart` (325) และ `EmptyCart` (333) |
| `checkoutservice` -> `emailservice` | `checkoutservice.yaml (EMAIL_SERVICE_ADDR)` | ✅ | `src/checkoutservice/main.go` บรรทัด 115 อ่าน env, เรียก `SendOrderConfirmation` (380); ถ้าล้มเหลวแค่ `log.Warnf` แล้วคำสั่งซื้อยังสำเร็จ (273-274) |
| `frontend` -> `shippingservice` | `frontend.yaml (SHIPPING_SERVICE_ADDR)` | ✅ | `src/frontend/main.go` บรรทัด 137 อ่าน env, 145 เชื่อม gRPC; `src/frontend/rpc.go` บรรทัด 88 เรียกเฉพาะ `GetQuote` |
| `recommendationservice` -> `productcatalogservice` | `recommendationservice.yaml (PRODUCT_CATALOG_SERVICE_ADDR)` | ✅ | `src/recommendationservice/recommendation_server.py` บรรทัด 131 อ่าน env, 135 ใช้ `grpc.insecure_channel`, 73 เรียก `ListProducts` |
| `cartservice` -> `redis-cart` | `cartservice.yaml (REDIS_ADDR)` | ✅ | `src/cartservice/src/Startup.cs` บรรทัด 29 อ่าน `REDIS_ADDR`, 36-40 ใช้ `AddStackExchangeRedisCache` + `RedisCartStore` |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): dependency มีอยู่จริงใน code แต่ Run B ยังมีจุดที่พลาดอยู่
  1. `checkoutservice` ส่ง trace ไปที่ OpenTelemetry collector ผ่าน gRPC (`COLLECTOR_SERVICE_ADDR`, `src/checkoutservice/main.go` บรรทัด 164-165) แต่จะทำงานเฉพาะเมื่อตั้ง `ENABLE_TRACING=1` (บรรทัด 90) ซึ่ง manifest ไม่ได้ตั้งไว้ ทำให้ Run B ไม่เห็น
  2. ตารางบอกแค่ว่า "ใครเรียกใคร" แต่ไม่บอกว่าเรียก RPC ไหน เช่น `checkoutservice` เรียกทั้ง `GetQuote` และ `ShipOrder` (บรรทัด 314, 387) ส่วน `frontend` เรียกแค่ `GetQuote`
  3. Run B บอกว่า image เดียวที่ระบุ tag คือ `redis:alpine` แต่ `loadgenerator.yaml` บรรทัด 67 ยังมี `busybox:1.38.0` อีกตัว
  นอกจากนี้ `cartservice` ไม่ได้ผูกติดกับ Redis เพราะ `Startup.cs` บรรทัด 42-56 สลับไปใช้ Spanner, AlloyDB หรือ in-memory ได้ ข้อนี้ manifest ไม่ได้บอกไว้

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

Run A ไม่รู้ว่า service ไหนเรียก service ไหน และยังไม่เห็น frontend กับ redis-cart เพราะ .proto บอกแค่ว่าแต่ละ service มี RPC อะไรให้เรียก แต่ไม่ได้บอกว่าใครเป็นผู้เรียก ส่วน frontend ไม่ได้เปิด gRPC service ให้ใคร และ Redis ก็ไม่ได้ใช้ gRPC จึงไม่มีอยู่ใน proto ข้อมูลพวกนี้ต้องดูจาก env var *_ADDR ใน manifest แทน

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

- คนที่อ่านแผนภาพจะเชื่อว่า checkout service ไม่ได้ส่งข้อมูลออกไปที่อื่นเลย แต่ถ้าเปิด tracing ขึ้นมาจะมีข้อมูลไหลออกไปอีกทาง ซึ่งมีผลทั้งต่อการ debug และต่อเรื่องความปลอดภัย