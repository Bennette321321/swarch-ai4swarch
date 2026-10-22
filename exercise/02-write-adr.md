**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้เลย เพราะไม่ได้อ่าน scenario แต่ไม่แต่งตัวเลขเอง ยกไปเป็น Open questions 10 ข้อ | ใช้ครบ: p95 <= 500 ms (ปัจจุบัน 300 ms), 200 orders/s, MailFast p95 1.5 s / p99 8 s / fail 0.5% / 50 emails/s, แต้มภายใน 1 นาที และคำนวณ backlog 9,000 ฉบับต่อนาทีช่วงพีก |
| ทางเลือกที่พิจารณา (ระบุ) | A: synchronous orchestration, B: pub/sub (เลือก, ไม่ระบุ broker), C: post-order orchestrator | 1: sync orchestration, 2: in-process goroutine, 3: RabbitMQ (เลือก), 4: Kafka |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | แต้ม: consumer idempotent ใช้ order ID เป็น key; rate limit: พูดกว้าง ๆ แค่ retry + backoff เพราะไม่รู้ตัวเลข 50/s | แต้ม: unique constraint บน `order_id` ใน transaction เดียวกันก่อน ack และแยก queue จากอีเมล; rate limit: limiter รวมทุก replica ≤ 50/s ให้ queue รับ backlog |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | แทบไม่มี ทุกข้อสันนิษฐานติดป้าย [Inferred] / Assumption ไว้ (เดาว่า checkout เรียก email แบบ sync ถูก ทั้งที่ไม่ได้อ่านโค้ด) | ระบุ `HTTPS` ระหว่าง customer กับ frontend โดยไม่มีหลักฐาน (manifest เป็น HTTP port 80); ใส่ `frontend → checkout` ที่ไม่อยู่ใน context (ติด [Inferred]) |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | เปลี่ยน protocol ระหว่าง customer กับ frontend จาก HTTPS เป็น HTTP | AI แต่งขึ้นเอง หลักฐานใน `frontend.yaml` คือ HTTP port 80 |
| 2 | เพิ่ม transactional outbox ให้ checkout แทนการ log แล้วใช้ reconciliation job | ร่างของ AI ขัดกันเอง (เสนอ reconciliation แต่ยอมรับว่าไม่มีที่เก็บ order) และถ้าแต้มหาย ลูกค้าเสียหายโดยตรง |
| 3 | ลบบรรทัดระบุ agent และใส่ชื่อตัวเองเป็น Deciders | Deciders เป็นผู้รับผิดชอบ ไม่ใช่ AI |
