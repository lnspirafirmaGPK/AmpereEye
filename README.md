# 🔋 AmpereEye — The Battery Truth

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Platform](https://img.shields.io/badge/platform-Android-blue.svg)](https://www.android.com)
[![Kotlin](https://img.shields.io/badge/kotlin-2.0.0-purple.svg)](https://kotlinlang.org)

AmpereEye คือแอปพลิเคชันตรวจสอบแบตเตอรี่ที่วัดค่าจากกระแสไฟจริง (Real Current mAh) ผ่านฮาร์ดแวร์ Fuel Gauge IC ไม่ใช้การคาดเดาจากโปรไฟล์พลังงาน มาพร้อม AI Governance และระบบวิเคราะห์เชิงลึก ช่วยให้คุณเข้าใจพฤติกรรมแบตเตอรี่และยืดอายุการใช้งานได้อย่างแท้จริง

![AmpereEye Banner](assets/banner.png) <!-- หากมีรูป -->

---

## ✨ คุณสมบัติหลัก

- **วัดกระแสไฟจริง** – อ่านค่า mA/mAh จาก `/sys/class/power_supply/battery/current_now` และ `charge_counter` โดยตรง ด้วย native code (Go/C) เพื่อความแม่นยำสูงสุด
- **สุขภาพแบตเตอรี่** – คำนวณความจุปัจจุบันเทียบกับดีไซน์ (Design Capacity) และแสดงเปอร์เซ็นต์การเสื่อมสภาพ
- **ผู้พิทักษ์การชาร์จ** – แจ้งเตือนเมื่อชาร์จถึงระดับที่ตั้ง (80%, 90% ฯลฯ) ป้องกัน overcharge
- **วิเคราะห์ Deep Sleep** – ตรวจจับ Wakelock ที่ผิดปกติด้วยโมเดล LightGBM บนเครื่อง
- **รายงานแยกตามแอป** – ดูปริมาณการใช้ mAh ของแต่ละแอปแบบเรียลไทม์ (รวมทั้ง background)
- **ประวัติการชาร์จ/คายประจุ** – เก็บสถิติทุกรอบ (cycle) ในฐานข้อมูล local พร้อมกราฟ
- **สรุปด้วย AI** – On-device LLM (Gemini Nano หรือ TinyLlama quantized) อ่านข้อมูลและให้คำแนะนำเป็นภาษาธรรมชาติ
- **ซิงก์คลาวด์แบบไม่ระบุตัวตน (Optional)** – ผู้ใช้สามารถเลือกส่งข้อมูลแบบ anonymized เพื่อช่วยพัฒนาโมเดล AI สำหรับอุปกรณ์รุ่นต่างๆ

---

## 🏗️ สถาปัตยกรรมระบบ (System Architecture)

AmpereEye ออกแบบตามหลัก **4‑Layer Architecture** ที่เน้นความแม่นยำและประหยัดพลังงาน:

```mermaid
flowchart TD
    subgraph A [Layer 1: ระบบประสาทสัมผัสระดับจุลภาค]
        direction TB
        K[Kernel Space<br/>Go/C Native Daemon] -->|อ่านค่า mA/µAh| FG[Fuel Gauge IC]
        K -->|Micro‑batch Wakeups| SM[AlarmManager / WorkManager]
    end

    subgraph B [Layer 2: ผู้พิทักษ์และผู้คุมกฎ]
        direction TB
        GE[Governance Enforcer] -->|ตรวจจับ Wakelock| PS[/proc/wakelocks]
        GE -->|แจ้งเตือน 80%| NO[Notification Service]
        GE -->|Anomaly Detection| ML[LightGBM Model]
    end

    subgraph C [Layer 3: ฮิปโปแคมปัสของระบบ]
        direction TB
        DB[(Local DB<br/>SQLite / Realm)] -->|Snapshot & Cycle| TS[Time‑series Data]
        DB -->|App Usage Stats| AU[App Statistics]
    end

    subgraph D [Layer 4: นักสื่อสารจิตวิญญาณ]
        direction TB
        LLM[On‑device LLM] -->|Summarize| INS[User Insights / Notifications]
        UI[UI Layer] -->|Query| DB
    end

    A -->|Raw Data| B
    B -->|Filtered/Annotated Data| C
    C -->|Historical Context| D
    D -->|Display| UI

    C -.->|Optional Anonymized Sync| Cloud[(Cloud Analytics<br/>BigQuery / Vertex AI)]
