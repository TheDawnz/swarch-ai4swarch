**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:**
อัครชัย ยงสุวรรณกุล
**agent ที่ใช้:** เช่น GitHub Copilot (Agent mode), Gemini Code Assist, Gemini CLI, Claude Code, Codex, Cursor, Cline: …
GitHub Copilot (Agent mode)
**โมเดล:** …
gpt-6 luna
**แพ็กเกจที่ใช้:** เช่น Copilot Student, Copilot Free, Gemini รุ่นฟรี: …
Copilot Student
บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | GitHub Copilot (Agent mode) / gpt-6 luna | `Follow prompts/1-recover-architecture.md`; `run: A` | `system/online-boutique/protos/demo.proto`| ใช่ | 3 | |
| 2 | 1-B | GitHub Copilot (Agent mode) / gpt-6 luna | `Follow prompts/1-recover-architecture.md`; `run: B` | `system/online-boutique/protos/demo.proto`; `system/online-boutique/kubernetes-manifests/` | ใช่ | 5 | ทำให้เห็น service addresses, Redis, LoadBalancer และ dependency ที่ deployment ระบุ |
| 3 | 2-a | GitHub Copilot (Agent mode) / gpt-6 luna | `Follow prompts/2-write-adr.md`; `run: a`; `context: none` | `docs/adr/template.md` | ใช่ | 4 | Zero-shot; ใช้ requirement R-12 ที่ส่งใน prompt แต่ไม่มี context files |
| 4 | 2-b | GitHub Copilot (Agent mode) / gpt-6 luna | `Follow prompts/2-write-adr.md`; `run: b`; มี context | `docs/adr/template.md`; `docs/scenario.md`; `docs/adr/0001-use-redis-for-cart-storage.md`; `system/online-boutique/src/checkoutservice/main.go` | ใช่ | 5 | มี scenario และบริบทระบบ จึงอธิบายข้อจำกัดและ current flow ได้ละเอียดกว่า 2-a |
| 5 | 3 | GitHub Copilot (Agent mode) / gpt-6 luna | `Follow prompts/3-review-adr.md`; `adr: docs/adr/0002-<your-title>.md` | ADR ฉบับสุดท้ายที่ระบุใน prompt; `docs/scenario.md` | ใช่ | 5 | ได้ ATAM review พร้อม quality-attribute scenarios, findings, risks, trade-offs และ verdict |
