# n8n AI Agent vs Basic LLM Chain

**ทำไมถึงเกิดความแตกต่าง?**

1.  **Basic LLM Chain Node:**
    *   Node นี้ทำงานตรงไปตรงมา คือส่ง Prompt และรับคำตอบกลับจาก LLM เป็นหลัก
    *   เมื่อคุณใส่ JSON Schema ที่ซับซ้อนเข้าไป (ไม่ว่าจะในช่อง Prompt หลัก หรือช่องเฉพาะสำหรับ Output Format ถ้ามี), Node นี้อาจจะไม่ได้ส่ง Schema นั้นไปให้ Gemini API ในฐานะ "ข้อกำหนดโครงสร้าง Output ที่ต้องทำตามเป๊ะๆ" (ซึ่งต้องใช้ parameter เฉพาะของ API เช่น `response_mime_type` หรือ `response_schema`) หรือในฐานะ "Tool/Function Declaration"
    *   แต่ LLM (Gemini) อาจจะตีความ Schema ที่เห็นใน Prompt นั้นเป็น **ส่วนหนึ่งของคำสั่งที่เป็นข้อความธรรมดา (text instruction)** และพยายามทำตามคำสั่งนั้นอย่างดีที่สุดในการสร้าง Output ให้คล้ายกับ Schema ที่ให้มา มันจึง "ดูเหมือน" ทำงานได้ เพราะ LLM แค่ทำตามคำสั่งใน Prompt แต่ไม่ได้ถูกบังคับด้วยข้อกำหนดทางเทคนิคของ API อย่างเข้มงวด
2.  **AI Agent Node:**
    *   Node นี้ถูกออกแบบมาให้ซับซ้อนกว่า รองรับการทำงานเป็น Agent, การใช้ Tools (Function Calling), และการจัดการ Memory
    *   เมื่อคุณใส่ JSON Schema เข้าไปในส่วนที่เกี่ยวข้องกับ Tools/Functions (ซึ่ง Error Message `tools[0].function_declarations...` บ่งชี้ชัดเจนว่าคุณกำลังทำสิ่งนี้), Node นี้จะพยายามส่ง Schema นั้นไปให้ Gemini API ในฐานะ **การประกาศฟังก์ชัน (Function Declaration)** ซึ่งต้องเป็นไปตาม **OpenAPI Schema ที่ถูกต้อง** ไม่ใช่ Schema สำหรับ Output ทั่วไป
    *   API ของ Google จึงปฏิเสธเพราะโครงสร้าง Schema ที่คุณส่งไปในส่วน `tools` นั้นผิดรูปแบบ (`Unknown name "type" at ...`) มันไม่เข้าใจว่า `{ "type": "object" }` หรือ `{ "type": ["string", "null"] }` คืออะไรในบริบทของการประกาศ *พารามิเตอร์* ของฟังก์ชัน
    *   **Key Point:** AI Agent node พยายามใช้ฟีเจอร์ขั้นสูง (Tools/Functions) ของ Gemini API ซึ่งมีการตรวจสอบความถูกต้องของ Input (Schema) ที่เข้มงวดกว่ามาก

**แนวทางการสร้าง Workflow (ผสมผสาน Agent และ Basic LLM Chain):**

เนื่องจากข้อจำกัดที่คุณระบุ (Agent ต่อ Memory ได้, Basic LLM ต่อไม่ได้) และปัญหาเรื่อง Schema กับ Agent Node เราสามารถออกแบบ Workflow แบบผสมผสานได้ดังนี้:

**หลักการ:**

*   ใช้ **AI Agent Node** เป็นตัวหลักในการ **จัดการบทสนทนา (Conversation Flow)**, **ตัดสินใจว่าต้องทำอะไรต่อ (Decision Making)**, และ **เชื่อมต่อกับ Memory** เพื่อรักษาบริบท
*   เมื่อ AI Agent ตัดสินใจว่าต้องการ *สร้าง Output ที่มีโครงสร้าง JSON ซับซ้อนตามที่คุณต้องการ*, ให้มันส่งสัญญาณและข้อมูลที่จำเป็นไปยัง **Basic LLM Chain Node** เพื่อทำหน้าที่ **สร้าง JSON Output นั้นโดยเฉพาะ** (โดยอาศัยความสามารถของ LLM ในการทำตามคำสั่ง Schema ที่ให้ไปใน Prompt แบบข้อความ)

**โครงสร้าง Workflow:**

1.  **Trigger:** (Webhook, LINE, Email, etc.) - เหมือนเดิม
2.  **Memory Load:** (เช่น Vector Store Retrieve, Database Lookup) - โหลดประวัติแชท/ข้อมูลบริบทสำหรับ User ID นี้
3.  **AI Agent Node (ตัวจัดการหลัก):**
    *   **Input:** User Message (จาก Trigger), Loaded Memory/Chat History
    *   **Prompt:** ออกแบบ Prompt ให้เน้นการ **ทำความเข้าใจ** ข้อความล่าสุดของลูกค้าในบริบทของ Memory, **ตัดสินใจ** ว่าขั้นตอนต่อไปควรเป็นอะไร (เช่น ถามคำถามต่อ, ต้องการข้อมูลสินค้า, ต้องการคำนวณราคา, ต้องการบันทึก Lead, ต้องการสร้าง JSON สรุป) และ **ส่งข้อความตอบกลับง่ายๆ** หรือ **ส่งสัญญาณ** บอกขั้นตอนต่อไป
        *   **สำคัญ:** **ห้ามใส่ Output JSON Schema ที่ซับซ้อนของคุณเข้าไปในส่วน Tools/Function Declaration ของ Node นี้เด็ดขาด**
        *   ถ้าต้องการให้ Agent เรียกใช้ Tools จริงๆ (เช่น Query DB) ให้ประกาศ Tools เหล่านั้นด้วย **OpenAPI Schema ที่ถูกต้อง** สำหรับพารามิเตอร์ (ตามที่แนะนำในคำตอบก่อนหน้า)
        *   *ทางเลือก:* อาจจะให้ Agent ตอบกลับเป็น JSON ง่ายๆ ที่บอกแค่ว่าต้องทำอะไรต่อ เช่น `{"next_step": "generate_final_json", "data_for_json": {...}}` หรือ `{"next_step": "ask_question", "message": "คำถาม..."}`
    *   **Memory:** เชื่อมต่อ Memory ที่โหลดมา และตั้งค่าให้บันทึกกลับ
    *   **Tools/Functions:** กำหนด Tools เท่าที่จำเป็นจริงๆ ด้วย Schema ที่ถูกต้อง หรือ *ไม่ต้องกำหนดเลย* ถ้า Agent มีหน้าที่แค่จัดการบทสนทนาและตัดสินใจ
    *   **Output:** ผลลัพธ์จาก Agent (อาจจะเป็นข้อความตอบกลับง่ายๆ หรือสัญญาณบอกขั้นตอนถัดไป)
4.  **Switch Node (ตัวแยกงานตาม Agent Decision):**
    *   แยกการทำงานตาม Output ของ AI Agent Node (เช่น ตามค่า `next_step` ที่ Agent ส่งมา)
    *   **Case 1: "ask_question" / Simple Reply:** ส่ง Output `message` จาก Agent ไปยัง Node ปลายทาง (LINE, Email) โดยตรง
    *   **Case 2: "query_products" / "get_price":** ส่งต่อไปยัง Node PostgreSQL เพื่อดึงข้อมูล -> จากนั้นอาจจะส่งข้อมูลกลับไปให้ AI Agent อีกรอบ (ถ้า Agent ต้องใช้ข้อมูลนั้นตัดสินใจต่อ) หรือส่งต่อไปยังขั้นตอนสร้าง JSON (ถ้าข้อมูลพร้อมแล้ว)
    *   **Case 3: "log_lead":** ส่งต่อไปยัง Node Google Sheets
    *   **Case 4: "generate_final_json":** **ส่งต่อไปยัง Basic LLM Chain Node** พร้อมข้อมูลที่จำเป็น (`data_for_json`)
5.  **Basic LLM Chain Node (ตัวสร้าง JSON เฉพาะกิจ):**
    *   **Input:** รับข้อมูลที่จำเป็นมาจากขั้นตอนก่อนหน้า (เช่น `data_for_json` จาก Agent)
    *   **Prompt:** สร้าง Prompt ที่ **เฉพาะเจาะจง** สำหรับ Node นี้:
        ```text
        คุณคือผู้ช่วยที่ทำหน้าที่สร้าง JSON เท่านั้น

        ข้อมูลที่ได้รับ:
        {{ $json.data_for_json }} // หรือ Expression ที่ถูกต้องเพื่อดึงข้อมูล

        โปรดสร้าง JSON object ตาม Schema ต่อไปนี้ **เท่านั้น** โดยใช้ข้อมูลที่ได้รับ:

        ```json
        // <<< วาง Output JSON Schema ที่ซับซ้อนของคุณตรงนี้ (เป็นข้อความธรรมดา) >>>
        {
          "type": "object",
          "properties": {
            "conversation_status": { ... },
            "next_action": { ... },
            "response_message": { ... }
          },
          "required": ["conversation_status", "response_message"]
        }
        ```

        **สำคัญ:** ห้ามใส่ข้อความอธิบายหรือ markdown ใดๆ นำหน้าหรือต่อท้าย JSON object ที่สร้างขึ้น ส่งออกเฉพาะ JSON object เท่านั้น
        ```
    *   **Model:** เลือก Gemini
    *   **Output:** ควรจะเป็น String ที่มีเฉพาะ JSON ที่คุณต้องการ
6.  **Parse JSON Node:** (ใช้ `Set` หรือ `Code` node)
    *   แปลง String JSON ที่ได้จาก Basic LLM Chain ให้เป็น JSON Object จริงๆ (`JSON.parse(...)`)
7.  **Switch / Logic Nodes (หลังได้ JSON):**
    *   ใช้ข้อมูลจาก JSON ที่ Parse แล้ว (เช่น `next_action.action_type`) เพื่อทำงานต่อ เช่น:
        *   ถ้า `next_action` คือ `query_products` -> ไป Node PostgreSQL
        *   ถ้า `next_action` คือ `log_lead` -> ไป Node Google Sheets
        *   ดึง `response_message` เพื่อส่งกลับให้ลูกค้า
8.  **Memory Save:** (อาจจะทำก่อนส่ง Response สุดท้าย) - บันทึกสถานะล่าสุด (อาจจะรวมข้อมูลจาก Agent และ Basic LLM) กลับไปยัง Memory Store
9.  **Output Node:** (LINE Send, Email, etc.) - ส่ง `response_message` ที่ได้ (อาจจะมาจาก JSON สุดท้าย หรือจาก Agent โดยตรง) กลับไปหาลูกค้า

**ข้อดีของแนวทางนี้:**

*   ใช้ประโยชน์จาก Memory ของ AI Agent Node ได้
*   หลีกเลี่ยงปัญหา Error 400 จากการใส่ Schema ผิดที่ใน Agent Node
*   มอบหมายงานสร้าง JSON ที่ซับซ้อน (ซึ่ง Basic LLM Chain ดูเหมือนจะจัดการได้ผ่าน Prompt) ให้ Node ที่เหมาะสม

**ข้อควรพิจารณา:**

*   **ความเสถียรของ Basic LLM Chain:** การให้ LLM สร้าง JSON ตาม Schema ใน Prompt แบบข้อความ อาจไม่เสถียร 100% เท่ากับการใช้ฟีเจอร์บังคับ Output Format ของ API โดยตรง (ซึ่ง Agent Node *ควรจะ* ทำได้ถ้าตั้งค่าถูก) อาจมีบางครั้งที่ LLM สร้าง JSON ผิดรูปแบบ ต้องมี Error Handling เพิ่มเติม
*   **ความซับซ้อนของ Workflow:** Workflow จะมีขั้นตอนเพิ่มขึ้น
*   **การส่งข้อมูลระหว่าง Agent และ Basic LLM:** ต้องออกแบบข้อมูลที่ส่งต่อ (`data_for_json`) ให้ดี เพื่อให้ Basic LLM มีข้อมูลเพียงพอที่จะสร้าง JSON ได้ถูกต้อง
*   **ทางออกที่ดีที่สุด (Ideal Solution):** การแก้ไขให้ AI Agent Node ทำงานกับ Tools/Functions และ/หรือ Output Formatting ได้อย่างถูกต้องตาม Schema ที่ Gemini API กำหนด ยังคงเป็นทางออกที่ตรงไปตรงมาและมีประสิทธิภาพสูงสุดในระยะยาว หากเป็นไปได้ ลองตรวจสอบการตั้งค่า Tools/Functions ใน Agent Node อีกครั้งให้แน่ใจว่าใช้ OpenAPI Schema ที่ถูกต้องสำหรับการประกาศพารามิเตอร์จริงๆ
