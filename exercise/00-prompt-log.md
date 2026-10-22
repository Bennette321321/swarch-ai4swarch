**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** ธนกร ไชยยุทธ / 6631321321
**agent ที่ใช้:** Claude Code (VS Code extension)
**โมเดล:** Claude Opus 5.5
**แพ็กเกจที่ใช้:** Claude Pro

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| #   | งาน / run | agent + โมเดล                 | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน                                                              | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ                                        |
| --- | --------- | ----------------------------- | ------------------- | ------------------------------------------------------------------------------- | ----------------------------------- | ---------------- | ----------------------------------------------- |
| 1   | 1-A       | Claude Code + Claude Opus 5.5 | `prompts/1-…` run A | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | อยู่                                | 4                | ไม่มี dependency เลย, แยกสิ่งที่เดาไปไว้ใน Gaps |
| 2   | 1-B       | Claude Code + Claude Opus 5.5 | `prompts/1-…` run B | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto`, ทุกไฟล์ใน `system/online-boutique/kubernetes-manifests/` (13 ไฟล์) | อยู่ | 5 | ได้ 12 containers, 17 dependencies; เจอ frontend, redis-cart, loadgenerator ที่ run A ไม่เห็น; ผิดเล็กน้อย: บอกว่า image เดียวที่ระบุ tag คือ redis:alpine แต่ยังมี busybox ใน loadgenerator.yaml |
| 3   | 2-a       | Claude Code + Claude Opus 5.5 | `prompts/2-…` run a | `prompts/2-write-adr.md`, `docs/adr/template.md` | อยู่ (แต่ list ชื่อไฟล์ใน `exercise/outputs/` โดยไม่เปิดอ่าน) | 4 | ไม่ได้อ่าน scenario; ไม่แต่งตัวเลข ยกไป Open questions 10 ข้อ; เลือก pub/sub แต่ไม่เลือก broker |
| 4   | 2-b       |                               | `prompts/2-…` run b |                                                                                 |                                     |                  |                                                 |
| 5   | 3         |                               | `prompts/3-…`       |                                                                                 |                                     |                  |                                                 |
| 6   | 1 (ตรวจ 1B) | Claude Code + Claude Opus 5.5 | ขอให้ช่วยตรวจ 5 dependency ของ 1B กับซอร์สโค้ด | `src/checkoutservice/main.go`, `src/frontend/main.go`, `src/frontend/rpc.go`, `src/recommendationservice/recommendation_server.py`, `src/cartservice/src/Startup.cs` | อยู่ | 5 | เปิดดูบรรทัดที่ AI อ้างเพื่อ verify เอง |
| 7   |           |                               |                     |                                                                                 |                                     |                  |                                                 |

