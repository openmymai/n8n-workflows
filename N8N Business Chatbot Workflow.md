
# N8N Business Chatbot Workflow (Conceptual Outline)

โครงสร้าง Workflow:

*   **Trigger:**
    *   `Webhook`: รับข้อความจาก LINE, Web Chat, etc.
    *   `Email Trigger`: หากต้องการรับทาง Email

*   **Initial Setup:**
    *   `Set` (Extract User Input): ดึงข้อความจาก Trigger (`$json.body.events[0].message.text` สำหรับ LINE Webhook).
    *   `Set` (Load Chat History - *Optional but Recommended*): ถ้าต้องการให้ AI จำบทสนทนาได้ ต้องมีการโหลด/บันทึก History อาจใช้ Data Store หรือ Database เก็บตาม User ID.

*   **Core AI Interaction:**
    *   `[Your Chosen LLM Chat Node]` (e.g., Google Vertex AI, OpenAI Chat Model):
        *   **System Prompt:** ใส่ Prompt ที่สร้างไว้ด้านบน.
        *   **User Message:** ใส่ข้อความจากผู้ใช้ (`{{ $node["Set (Extract User Input)"].json.userInput }}`).
        *   **Chat History:** ใส่ History ที่โหลดมา (ถ้ามี).
        *   **(Optional) Function Calling:** หาก LLM และ Node รองรับ อาจใช้ Function Calling แทนการให้ AI ส่ง JSON `next_action`.
    *   `Set` (Parse AI Response): แยกข้อมูลจาก JSON output ของ AI (`$json.choices[0].message.content` หรือคล้ายกัน แล้ว Parse JSON). เก็บ `conversation_status`, `next_action`, `response_message` ไว้.

*   **Action Handling (`Switch` Node):**
    *   ใช้ `Switch` Node แยกการทำงานตามค่า `$node["Set (Parse AI Response)"].json.next_action.action_type`.
    *   **Output 1: `query_products`**
        *   `Set` (Prepare DB Query): สร้าง SQL query โดยใช้ Parameters จาก `$node["Set (Parse AI Response)"].json.next_action.parameters`. เช่น:
            ```sql
            SELECT product_id, name, description, price, image_url, catalog_link
            FROM products
            WHERE type = '{{ $node["Set (Parse AI Response)"].json.next_action.parameters.type }}'
              AND occasion = '{{ $node["Set (Parse AI Response)"].json.next_action.parameters.occasion }}'
              AND price <= {{ $node["Set (Parse AI Response)"].json.next_action.parameters.budget_max }}
            LIMIT 3;
            ```
        *   `PostgreSQL` (Execute Query): รัน Query ที่สร้าง.
        *   `Set` (Format DB Results): จัดรูปแบบผลลัพธ์ DB ให้อ่านง่ายสำหรับ AI (เช่น "Product List: 1. C-012 (190 THB) Desc: ... Link: ... 2. S-105 ...") เก็บไว้ในตัวแปร เช่น `dbResultsContext`.
        *   **Loop Back:** เชื่อมกลับไปยัง `LLM Chat Model` Node อีกครั้ง โดยส่ง `dbResultsContext` เพิ่มเข้าไปใน Prompt/Context เพื่อให้ AI สร้างข้อความแนะนำสินค้า.
    *   **Output 2: `get_price`**
        *   `Set` (Prepare Price Query): สร้าง SQL query ดึงราคาจาก `product_id`.
            ```sql
            SELECT price FROM products WHERE product_id = '{{ $node["Set (Parse AI Response)"].json.next_action.parameters.product_id }}';
            ```
        *   `PostgreSQL` (Execute Query): รัน Query.
        *   `Code` / `Set` (Calculate Total): ดึงราคาต่อหน่วยจากผลลัพธ์ DB (`$node["PostgreSQL (Get Price)"].json[0].price`) คูณกับ Quantity (`$node["Set (Parse AI Response)"].json.next_action.parameters.quantity`). เก็บราคาต่อหน่วยและราคารวมไว้.
        *   **Loop Back:** เชื่อมกลับไปยัง `LLM Chat Model` Node โดยส่ง Unit Price และ Total Price เข้าไปใน Context เพื่อให้ AI สร้างข้อความสรุปยอด.
    *   **Output 3: `log_lead`**
        *   `Set` (Prepare Sheet Data): เตรียมข้อมูลสำหรับ Google Sheets จาก `$node["Set (Parse AI Response)"].json.next_action.parameters.summary` และข้อมูลอื่นๆ ที่อาจเก็บไว้ (เช่น User ID).
        *   `Google Sheets` (Append Row): เพิ่มข้อมูลลูกค้าลงใน Sheet ที่กำหนด.
        *   *อาจจะเชื่อมต่อไปยังการส่งข้อความสุดท้ายด้านล่าง*
    *   **Output 4: `send_message` (Default/No Action)**
        *   *ไม่ต้องทำอะไรพิเศษ แค่เตรียมส่งข้อความ*

*   **Send Response:**
    *   `IF` (Check if message exists): ตรวจสอบว่า `$node["Set (Parse AI Response)"].json.response_message` ไม่ใช่ null หรือ empty.
    *   **True:**
        *   `[LINE Send Message / Email / Webhook Response]` : ส่ง `response_message` กลับไปหาลูกค้าตามช่องทางที่ Trigger เข้ามา.
    *   **False:** (อาจจะเกิดตอน log_lead แล้วไม่มีข้อความตอบกลับ หรือตอน user ขอหยุด) -> `NoOp` (Do Nothing) หรือจบ Workflow.

*   **Update Chat History (Optional but Recommended):**
    *   ก่อนจบ Workflow (หรือก่อนส่ง Response) ให้บันทึกข้อความล่าสุดของ User และ AI กลับไปที่ Data Store/Database สำหรับ User ID นั้นๆ.

**การเตรียมการ:**

1.  **Database (PostgreSQL):** สร้างตาราง `products` ที่มีคอลัมน์อย่างน้อย: `product_id` (text, primary key), `name` (text), `description` (text), `price` (numeric), `type` (text, e.g., 'สำเร็จรูป', 'สั่งทำ'), `occasion` (text, e.g., 'โรงเรียน', 'กีฬา'), `image_url` (text, optional), `catalog_link` (text, optional).
2.  **Google Sheets:** เตรียม Sheet สำหรับบันทึกข้อมูลลูกค้า กำหนดคอลัมน์ที่ต้องการ (เช่น Timestamp, UserID/Platform, RequestType, Occasion, Quantity, Budget, SelectedProduct, QuoteAmount, ContactInfo, Status).
3.  **N8N Credentials:** ตั้งค่า Credentials สำหรับ PostgreSQL, Google Sheets, LINE/Email, และ LLM API Key (Google AI / OpenAI).
4.  **LLM Node:** เลือก Node สำหรับ LLM ที่คุณต้องการใช้ (Google Vertex AI, OpenAI, etc.) และใส่ API Key.

**ข้อควรจำ:**

*   Workflow นี้เป็นโครงสร้าง คุณต้องปรับแก้ Expression, SQL Query, และการเชื่อมต่อ Node ให้ถูกต้อง.
*   การจัดการ Chat History เป็นสิ่งสำคัญเพื่อให้ AI สนทนาได้อย่างต่อเนื่อง.
*   ทดสอบ Workflow อย่างละเอียดกับหลายๆ สถานการณ์.
*   พิจารณาเรื่อง Error Handling (เช่น ถ้า Query DB ไม่เจอข้อมูล, ถ้า AI ตอบผิดรูปแบบ).
*   Prompt อาจจะต้องปรับจูนเพื่อให้ได้ผลลัพธ์และการควบคุม Flow ที่ดีที่สุดกับ LLM ที่คุณเลือกใช้.

หวังว่านี่จะเป็นจุดเริ่มต้นที่ดีในการสร้างระบบอัตโนมัติสำหรับธุรกิจถ้วยรางวัลของคุณนะครับ!
