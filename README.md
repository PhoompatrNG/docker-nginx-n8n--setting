# การตั้งค่า n8n ด้วย Nginx Docker

โปรเจกต์นี้เป็นการตั้งค่า Docker สำหรับการรัน n8n instance โดยมี Nginx ทำหน้าที่เป็น reverse proxy

## สิ่งที่ต้องมี

*   Docker
*   Docker Compose

## โครงสร้างไดเรกทอรี

```
.
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf
│   └── conf.d/
│       └── default.conf
└── README.md
```

## การกำหนดค่า

### `docker-compose.yml`

ไฟล์นี้กำหนดสอง services หลัก:

1.  **`n8n`**:
    *   ใช้ image `n8nio/n8n` อย่างเป็นทางการ
    *   เปิด port `5678` ภายใน ซึ่ง Nginx จะทำการ proxy ไปหา
    *   ใช้ volume ชื่อ `n8n_data` สำหรับเก็บข้อมูลของ n8n แบบถาวร
    *   มีการตั้งค่า Environment variables สำหรับการกำหนดค่าพื้นฐานของ n8n รวมถึง:
        *   `N8N_HOST`: ชื่อ hostname ที่ n8n คิดว่าตัวเองกำลังรันอยู่
        *   `N8N_PORT`: port ภายในที่ n8n ใช้ฟัง
        *   `N8N_PROTOCOL`: protocol ที่ n8n คาดหวัง
        *   `WEBHOOK_URL`: URL พื้นฐานสำหรับ webhooks
        *   `N8N_BASIC_AUTH_ACTIVE`, `N8N_BASIC_AUTH_USER`, `N8N_BASIC_AUTH_PASSWORD`: สำหรับการรักษาความปลอดภัย n8n ด้วย basic authentication **ขอแนะนำอย่างยิ่งให้ตั้งค่าข้อมูลประจำตัวที่ไม่ซ้ำกันและรัดกุมสำหรับสิ่งเหล่านี้ในไฟล์ `.env` หรือโดยตรงใน `docker-compose.yml`**
    *   เชื่อมต่อกับ `n8n_network`

2.  **`nginx`**:
    *   ใช้ image `nginx:latest` อย่างเป็นทางการ
    *   เปิด port `80` (HTTP) และ `443` (HTTPS) ไปยังเครื่อง host
    *   Mount ไฟล์การกำหนดค่า Nginx:
        *   `./nginx/nginx.conf` ไปยัง `/etc/nginx/nginx.conf`
        *   `./nginx/conf.d` ไปยัง `/etc/nginx/conf.d`
    *   ขึ้นอยู่กับ `n8n` service
    *   เชื่อมต่อกับ `n8n_network`

### การกำหนดค่า Nginx (`nginx/conf.d/default.conf`)

ไฟล์ `default.conf` กำหนดค่า Nginx:

*   ฟังบน port `80`
*   ตั้งค่า `server_name` เป็น `localhost` (สามารถเปลี่ยนเป็นโดเมนของคุณได้)
*   `location /` block จะ proxy ทุก requests ไปยัง `n8n` service ที่ `http://n8n:5678`
*   ตั้งค่า proxy headers ที่จำเป็นเพื่อให้ n8n ทำงานได้อย่างถูกต้อง รวมถึง headers สำหรับ WebSocket connections
*   มีส่วนที่ comment ไว้สำหรับการเปิดใช้งาน HTTPS ซึ่งจะต้องใช้ SSL certificates และการกำหนดค่าเพิ่มเติม

## วิธีการรัน

1.  **ตั้งค่า Environment Variables (แนะนำ):**
    สร้างไฟล์ `.env` ในไดเรกทอรี `c:\Work\docker-n8n` เพื่อเก็บข้อมูลที่ละเอียดอ่อน เช่น ข้อมูล basic auth:
    ```env
    N8N_BASIC_AUTH_USER=your_username
    N8N_BASIC_AUTH_PASSWORD=your_strong_password
    # เพิ่ม environment variables อื่นๆ ตามต้องการ
    ```
    ไฟล์ `docker-compose.yml` ถูกตั้งค่าให้ใช้ตัวแปรเหล่านี้หากมีไฟล์ `.env` อยู่

2.  **เริ่มการทำงานของ Services:**
    เปิด terminal ในไดเรกทอรี `c:\Work\docker-n8n` และรันคำสั่ง:
    ```powershell
    docker-compose up -d
    ```
    คำสั่งนี้จะ build images (หากยังไม่ได้ build) และเริ่มการทำงานของ containers ใน detached mode

## การเข้าถึง n8n

เมื่อ containers ทำงานแล้ว คุณควรจะสามารถเข้าถึง n8n instance ของคุณได้โดยไปที่:

*   `http://localhost` (หรือ `server_name` ที่คุณกำหนดค่าไว้หากมีการเปลี่ยนแปลง)

Nginx จะส่งต่อ request ของคุณไปยัง n8n container

## การหยุดการทำงานของ Services

ในการหยุด containers ให้รันคำสั่ง:
```powershell
docker-compose down
```

## การปรับแต่ง

### HTTPS/SSL

ไฟล์ `nginx/conf.d/default.conf` มีตัวอย่าง blocks ที่ comment ไว้สำหรับการตั้งค่า HTTPS ในการเปิดใช้งาน:
1.  รับ SSL certificates สำหรับโดเมนของคุณ
2.  Uncomment การ map port `443:443` ใน `docker-compose.yml` สำหรับ `nginx` service
3.  Uncomment และกำหนดค่า `server` block สำหรับ port `443` ใน `nginx/conf.d/default.conf` โดยระบุ path ที่ถูกต้องไปยัง SSL certificate และ key ของคุณ
4.  พิจารณาการ redirect HTTP ไป HTTPS โดยเพิ่ม redirect block ที่เกี่ยวข้องในการกำหนดค่า Nginx ของคุณ

### ฐานข้อมูล

`n8n` service ใน `docker-compose.yml` มีบรรทัดที่ comment ไว้สำหรับการกำหนดค่าฐานข้อมูล PostgreSQL ภายนอก สำหรับการใช้งานจริง ขอแนะนำให้ใช้ฐานข้อมูลเฉพาะแทน SQLite ที่เป็นค่าเริ่มต้น

### n8n Instances หลายตัว

การตั้งค่านี้สามารถปรับใช้เพื่อรัน n8n instances หลายตัวได้ ซึ่งโดยทั่วไปจะเกี่ยวข้องกับ:
1.  การกำหนด n8n services เพิ่มเติมใน `docker-compose.yml` (เช่น `n8n2`) โดยแต่ละ service จะมี:
    *   ชื่อ Volume ที่ไม่ซ้ำกัน (เช่น `n8n2_data`)
    *   การกำหนดค่า port ภายในที่ไม่ซ้ำกัน (เช่น `N8N_PORT=5679`)
    *   การ map host port หากมีการเปิดเผยโดยตรง (เช่น `"5679:5679"`)
    *   Environment variables โดยเฉพาะ `N8N_PATH` หากจะให้บริการภายใต้ subpaths ที่แตกต่างกัน
2.  การอัปเดตการกำหนดค่า Nginx (`default.conf`) เพื่อรวม `location` blocks แยกต่างหากสำหรับแต่ละ instance โดย proxy ไปยัง service และ port ที่ถูกต้อง (เช่น `location /n8n1/ { proxy_pass http://n8n1_service_name:port; ... }` และ `location /n8n2/ { proxy_pass http://n8n2_service_name:port; ... }`)
3.  ตรวจสอบให้แน่ใจว่าแต่ละ n8n instance ได้รับการกำหนดค่าด้วย `N8N_PATH` ที่ถูกต้อง (เช่น `/` หาก Nginx ทำการ rewrite subpath หรือ subpath เองหาก n8n จำเป็นต้องรับทราบ)
