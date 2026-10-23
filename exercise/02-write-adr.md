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
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) |ใช้ตัวเลขสำคัญ เช่น PlaceOrder p95 ไม่เกิน 500 ms, peak 200 orders/s, MailFast p95 1.5 s, p99 8 s, failure 0.5% และ limit 50 emails/s|ใช้ตัวเลขชุดเดียวกันครบถ้วน และเชื่อมโยงกับ current flow ของ checkoutservice ได้ชัดกว่า|
| ทางเลือกที่พิจารณา (ระบุ) |1. RabbitMQ event exchange และ durable queues 2. synchronous gRPC calls 3. Kafka event log|พิจารณา 3 ทางเลือกเช่นเดียวกัน แต่ข้อดีข้อเสียผูกกับ Kubernetes, Go/gRPC, RabbitMQ experience และระบบปัจจุบันละเอียดกว่า|
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร |เสนอ stable event ID, idempotent consumer และ deduplication สำหรับ loyalty; แยก queue และ rate-limit MailFast ที่ 50 calls/s|เสนอแนวทางเดียวกัน แต่ระบุชัดขึ้นว่า loyaltyservice ต้องเก็บ processed-event record และต้องมี durable handoff/outbox|
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) |โดยรวมไม่แต่งตัวเลขหลัก แต่รายละเอียดบางอย่าง เช่น storage ของ outbox, RabbitMQ topology และ transaction boundary ยังไม่ได้มาจากหลักฐาน|มี context มากกว่า จึงอธิบาย current flow ได้ดีขึ้น แต่ยังต้องตรวจว่า claims เกี่ยวกับ EMAIL_SERVICE_ADDR และ synchronous calls ตรงกับ source จริงหรือไม่|

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 |เพิ่มเงื่อนไขว่า order event ต้องถูกบันทึกด้วย durable outbox ก่อนจึงถือว่า order ถูก durably accepted|ป้องกันกรณี order สำเร็จแล้ว process ล้มก่อน publish event ซึ่งจะทำให้ email, loyalty และ analytics ไม่ทำงาน|
| 2 |ระบุว่า exactly-once loyalty ต้องใช้ atomic update ระหว่างการเพิ่มแต้มกับการบันทึก processed-event ID|stable event ID เพียงอย่างเดียวไม่พอ หากการเพิ่มแต้มสำเร็จแต่การบันทึก deduplication ล้มเหลว อาจเกิดการเพิ่มแต้มซ้ำ|
| 3 |เพิ่ม queue-depth threshold, retry policy, dead-letter policy และ alerting ในส่วน Validation|ต้องพิสูจน์ว่า loyalty ทำเสร็จภายใน 1 นาที และรองรับ MailFast ที่จำกัด 50 emails/s ได้จริง|