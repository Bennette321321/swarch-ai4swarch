Agent: Claude Code (VS Code extension) · Model: Claude Opus 5.5 (`claude-opus-5-5`)

# Review: ADR-0002 Publish an "OrderPlaced" event to RabbitMQ

ADR ที่รีวิว: `docs/adr/0002-publish-order-placed-event.md` (Status: Proposed) · ตัวเลขอ้างอิงจาก `docs/scenario.md`

หมายเหตุ: การอ้างบรรทัดใน `main.go` ของ ADR ไม่ได้ตรวจสอบในรีวิวนี้ เพราะ task อนุญาตให้อ่านแค่ ADR และ scenario

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| S1 (Performance) | ลูกค้าช่วง 11.11 | `PlaceOrder` เข้ามา 200 orders/s | peak load และ MailFast ตอบช้า (p95 1.5 s, p99 8 s) | order สำเร็จ และ event `OrderPlaced` ถูกบันทึกอย่าง durable (outbox หรือ broker) ก่อนตอบ client | p95 ของ `PlaceOrder` ≤ 500 ms (เดิมประมาณ 300 ms จึงเหลือ budget ราว 200 ms) |
| S2 (Availability) | MailFast หรือ `loyaltyservice` | ล่มติดต่อกัน 10 นาที | peak 200 orders/s | order ยังสำเร็จ ข้อความค้างใน queue แล้วถูกประมวลผลเมื่อปลายทางกลับมา | ไม่มี order ใดล้มเพราะ email/loyalty/analytics และไม่มี event หาย (ยกเว้นที่อยู่ใน DLQ พร้อม alert) |
| S3 (Correctness) | RabbitMQ หรือ outbox relay | ส่ง `OrderPlaced` ซ้ำ หรือ pod ของ consumer ถูก kill ก่อน ack | peak 200 orders/s | `loyaltyservice` บันทึก points ต่อ `order_id` เพียงครั้งเดียว | ทุก order ได้ points ภายใน 60 s, duplicate 0 ครั้ง, ไม่มี order ที่ตัดเงินแล้วแต่ไม่ได้ points |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | Outbox (Decision ข้อ 7) **ไม่ atomic กับ `chargeCard`** เพราะการตัดเงินเป็น remote call ไป `paymentservice` ไม่ได้อยู่ใน transaction เดียวกับตาราง outbox ช่องว่าง "ตัดเงินแล้วแต่เขียน event ไม่สำเร็จ" จึงยังอยู่ | ลูกค้าโดนตัดเงินแต่ไม่ได้ points ซึ่งผิด R-12 (ภายใน 1 นาที) และ mitigation ที่มีคือแค่ log กับ alert ให้คนตรวจ |
| Risk | ADR ขัดกันเอง: ข้อ 1 ให้ publish ตรงไป broker แล้ว log ถ้า fail แต่ข้อ 7 บอกให้ใช้ outbox แทน ส่วน C4 diagram ไม่มี outbox DB และ relay และ Positive consequence ข้อแรกยังพูดถึง "publish พร้อม confirm" บน critical path | ทีมไม่รู้ว่าต้อง build อะไร และ board ประเมิน latency budget ของ S1 ไม่ได้ เพราะไม่รู้ว่าบน critical path มี DB write หรือ AMQP confirm |
| Risk | ไม่ได้ระบุว่า **ถ้า outbox DB ล่ม** checkout จะทำอย่างไร และไม่ได้ระบุ HA ของ DB ("database ขนาดเล็ก") | ถ้า DB เป็น instance เดียว จะเป็นจุดล้มเดียวที่แย่กว่า RabbitMQ 3-node quorum ที่ ADR ตั้งใจจะป้องกัน ถ้าให้ order fail ก็ลด availability ของ checkout แต่ถ้าให้ผ่านไป points ก็หาย |
| Trade-off point | การเพิ่ม outbox: ได้ durability ของ event (correctness ของ loyalty) แต่เสีย latency (DB write อยู่บน critical path), availability (dependency แบบ sync ตัวใหม่) และ operability (ทีม 4 คนต้องดูแลทั้ง RabbitMQ cluster และ DB) | เป็นจุดตัดสินใจหลักที่ ADR ให้เหตุผลแค่ด้าน correctness แต่ไม่ได้ชั่งน้ำหนักอีกสามด้าน |
| Trade-off point | Queue + rate limit 50/s: ทำให้เคารพข้อจำกัดของ MailFast และไม่กระทบ checkout แต่ backlog อีเมลจะโต 150 ฉบับ/s หรือ 9,000 ฉบับ/นาที | ถ้า peak 11.11 ยาว 1 ชั่วโมง (scenario ไม่ได้ระบุ) backlog จะถึง 540,000 ฉบับ และต้องใช้ ≥ 3 ชั่วโมงจึงส่งหมด ทั้งยังต้อง sizing disk/memory ของ quorum queue ให้รับไหว |
| Sensitivity point | Latency ของ loyalty ตั้งแต่ต้นจนจบ = ช่วง poll ของ relay + เวลาใน broker + throughput ของ `loyaltyservice` (ต้อง ≥ 200 events/s แบบต่อเนื่อง) | ข้อกำหนด 60 s ไวต่อค่าเหล่านี้มาก แต่ ADR ยังไม่กำหนดค่าใดเลย (Open question 3) |
| Sensitivity point | Budget 200 ms บน critical path ไวต่อ latency ของ outbox insert (fsync) หรือ timeout ของ publisher confirm ("timeout สั้นๆ" ที่ไม่มีตัวเลข) | ถ้าไม่ตั้งตัวเลขไว้ จะ validate S1 ไม่ได้ |
| Sensitivity point | Concurrency ของ `emailservice` (75 ถึง 400 in-flight ตาม Little's law) และ retry policy ซึ่งใช้ quota 50/s ร่วมกับงานใหม่ | ถ้า retry ถี่เกินไปจะกิน rate limit และทำให้ backlog โตเร็วขึ้นอีก |
| Non-risk | `loyaltyservice` ใช้ idempotent consumer ที่มี unique constraint บน `order_id` ใน transaction เดียวกันแล้วจึง ack และแยก queue `orders.loyalty` ออกจากอีเมล | เป็นวิธีที่ถูกต้องในการได้ผลแบบ effectively-once บน at-least-once delivery และ backlog ของ MailFast จะไม่หน่วง loyalty |
| Non-risk | เลือก RabbitMQ แทน Kafka | ปริมาณ 200 events/s, ต้องการ retry/DLQ ทีละข้อความ, มี 2 คนในทีมเคยใช้ และเป็น OSS self-hosted ซึ่งเข้ากับ driver 6 |
| Trade-off point | Fan-out exchange ที่ส่ง payload เต็ม (รวมอีเมลลูกค้า): เพิ่ม consumer ใหม่ง่าย (modifiability) แต่ทุก queue ใหม่จะได้ personal data ไปโดยอัตโนมัติ (confidentiality, PDPA) | analytics และ consumer ในอนาคต (SMS, recommendations) จะได้ข้อมูลเกินกว่าที่จำเป็น |

## 3. Missing or weak

- **Options ที่ไม่ได้พิจารณา**
  - เขียน order record สถานะ `PENDING` ลง DB **ก่อน** `chargeCard` แล้วเปลี่ยนเป็น `PLACED` พร้อมเขียน outbox หลัง `shipOrder` วิธีนี้ทำให้ reconcile กับ payment ได้ ปิดช่องว่างใน Risk ข้อแรก และตอบ Open question 4 ไปด้วย
  - เมื่อเพิ่ม DB แล้ว ก็ควรเทียบกับ **DB-backed job queue** (outbox table กับ worker ต่อ action โดยไม่ใช้ broker) ซึ่งใช้ component น้อยกว่า แต่ modifiability ต่ำกว่า
  - **Thin event** (มีแค่ `order_id` แล้วให้ consumer ดึงข้อมูลเพิ่ม) หรือ topic exchange ที่แยก payload ตามสิทธิ์ เพื่อลดการกระจาย personal data
- **Consequences ที่ไม่ได้เขียน**
  - outbox insert เพิ่ม latency บน critical path
  - ข้อความ loyalty ที่ตกไปอยู่ใน DLQ ผิดข้อกำหนด 1 นาทีทันที จึงต้องมี SLA ของการแก้ด้วยมือ
  - ยังไม่ได้ระบุ cost/resources ของ DB และ relay
- **Claims ที่ไม่มีหลักฐานหรือน่าจะผิด**
  - "โอกาสต่ำกว่าเดิมมาก" (Risks ข้อแรก) ไม่มีตัวเลขหรือเหตุผลรองรับ และอาจผิดถ้า DB ไม่ได้ทำ HA
  - "ตอบ driver 1–5 ได้พร้อมกัน" ยังเกินจริง เพราะ driver 3 ยังมีช่องว่างที่ทำให้ points หายได้ ส่วน outbox ช่วยแค่กันไม่ให้ event หายหลังจากบันทึกลงตารางแล้ว ไม่ได้ทำให้เกิด exactly once ด้วยตัวเอง
  - "action ใหม่ไม่ต้องแก้ `checkoutservice`" จะจริงก็ต่อเมื่อ payload มีข้อมูลพอ อีกทั้ง fraud scoring มักต้องทำ **ก่อน** ตัดเงิน จึงอาจไม่ใช่ post-order consumer **[Inferred]**
  - ใน Validation ใช้ "p99.9 < 60 s" ซึ่งหลวมกว่าข้อกำหนด "within 1 minute" ของ scenario
  - Open question 1 กับ 3 ซ้ำกัน และ Validation ยังไม่มี failure injection ของ RabbitMQ, outbox DB และกรณี "kill checkout หลัง charge"

## 4. Verdict

**Accept with changes.** ทิศทาง (async event, RabbitMQ, idempotent loyalty) ถูกต้องสำหรับตัวเลขใน scenario แต่ส่วน durability ของ event ยังไม่สอดคล้องกันและยังมีช่องว่าง

สิ่งที่ต้องแก้ 3 ข้อที่สำคัญที่สุด:

1. **ทำให้ decision เป็นแบบเดียว** โดยเลือกระหว่าง outbox กับ direct publish แล้วแก้ข้อ 1, diagram (เพิ่ม outbox DB และ relay), consequences และ latency budget ให้ตรงกัน
2. **ปิดช่องว่าง "ตัดเงินแล้วแต่ไม่มี event"** ด้วยการเขียน order record `PENDING` ก่อน `chargeCard` พร้อม reconciliation job หรือเหตุผลชัดเจนว่าทำไมยอมรับได้ รวมถึงระบุ HA ของ DB และพฤติกรรมของ checkout เมื่อ DB ล่ม
3. **ใส่ตัวเลขให้ทุก sensitivity point** ได้แก่ timeout ของ DB/publish, ช่วง poll ของ relay, throughput ของ loyalty ≥ 200/s, เป้าเวลาส่งอีเมลและ sizing queue ตามความยาวของ peak และเพิ่ม failure injection ของ broker, DB และ checkout crash ลงใน Validation
