# DEPLOY_GUIDE — Guideline สำหรับเพิ่ม Docker + CI/CD ให้โปรเจกต์ของตัวเองด้วย AI

> เอกสารนี้เป็น **guideline** ให้ทั้งนิสิตและ AI coding agent (Claude Code / Codex) อ่าน
> บอก **หลักการ กฎ และผลลัพธ์ที่ต้องได้** — วิธีทำให้ AI วิเคราะห์และออกแบบตามสภาพแวดล้อมจริงของโปรเจกต์
> ใช้ **หลังจากทำ Lab Docker + CI/CD ส่วนที่ 1–6 เสร็จแล้ว**

---

## วิธีใช้ (นิสิต)

1. ดาวน์โหลดไฟล์นี้ไปไว้ที่ root ของโปรเจกต์ตัวเอง

   ```bash
   curl -fsSL -o DEPLOY_GUIDE.md https://raw.githubusercontent.com/suriyapi/nextjs-crud-starter/main/docs/DEPLOY_GUIDE.md
   ```

2. สั่ง AI ในโฟลเดอร์โปรเจกต์

   ```text
   อ่าน DEPLOY_GUIDE.md แล้ววิเคราะห์โปรเจกต์และสภาพแวดล้อมของฉัน
   วางแผนการเพิ่ม Docker + CI/CD ตาม guideline นี้ ให้ฉันยืนยันแผนก่อนลงมือ
   ทำตามกฎอย่างเคร่งครัด และอธิบายทุกไฟล์ที่สร้างหรือแก้เป็นภาษาไทย
   ```

3. ตรวจแผนและไฟล์ที่ AI ทำ แล้วทำ **ขั้นที่นิสิตต้องทำเอง** (หัวข้อ 6) ด้วยตัวเอง

---

## 1. กฎสำหรับ AI

1. **แก้ได้เฉพาะไฟล์ใน repo** — ห้าม SSH หรือรันคำสั่งบน VPS, ห้ามแก้ GitHub settings, ห้าม `git push`
2. **ห้ามแตะความลับ** — ห้ามเปิด/แสดง/commit `.env` (ยกเว้น `.env.example`), ห้ามขอ token / private key / รหัสผ่านจากนิสิต, ห้ามเขียนค่าความลับลงไฟล์
3. **ห้ามทำให้ข้อมูลหาย** — ห้าม `docker compose down -v`, ห้ามลบ volume หรือ migration เดิม
4. **แก้ business logic ให้น้อยที่สุด** — แก้เฉพาะที่จำเป็นต่อการ deploy
5. **วิเคราะห์ก่อน เสนอแผน แล้วรอนิสิตยืนยัน** — โดยเฉพาะการตัดสินใจใหญ่ (แยก/รวม service, เปลี่ยน DB, เปลี่ยนโครงสร้าง)
6. **ทดสอบในเครื่องได้** — build, `docker compose up`, ดู log, `docker compose down` (ไม่มี `-v`)
7. **จบด้วยสรุปภาษาไทย** — ไฟล์ที่สร้าง/แก้และเหตุผล, ผลการทดสอบ, ขั้นตอนที่นิสิตต้องทำเองพร้อมค่าจริงของโปรเจกต์ และคำสั่งที่ตรงกับ shell ของนิสิต

---

## 2. สิ่งที่ AI ต้องวิเคราะห์ก่อน

- **เครื่องของนิสิต** — OS และ shell (PowerShell / CMD / Git Bash / macOS / Linux)
- **โปรเจกต์** — stack, เวอร์ชัน, package manager, โครงสร้าง (monorepo / frontend + backend แยก), port, ไฟล์ Docker/CI ที่มีอยู่แล้ว (ปรับต่อ ไม่สร้างทับ)
- **ฐานข้อมูล** — ชนิด, ORM / migration tool, seed
- **ตัวแปรที่ฝังตอน build** ของ frontend (`VITE_*`, `NEXT_PUBLIC_*`, `REACT_APP_*`)
- **test / lint ที่มีอยู่**
- **ถามนิสิต**: GitHub username/repo, โดเมนหรือ subdomain, และสิ่งที่รันอยู่บน VPS แล้ว (port / โฟลเดอร์ / โดเมน ที่ห้ามซ้ำ — แอป Lab ใช้ `127.0.0.1:3300`, `~/app`)

---

## 3. สถาปัตยกรรมเป้าหมาย (เหมือนใน Lab)

```text
git push (main) → CI (test + docker build) → Deploy: build image → GHCR (Private, tag latest + sha)
                                                    → VPS: docker compose pull + up -d → health check
VPS: Nginx + HTTPS (certbot) → 127.0.0.1:<APP_PORT> → container app → db (volume)
```

---

## 4. ผลลัพธ์ที่ต้องได้

### โค้ด
- มี **health endpoint** ที่ตรวจการเชื่อมต่อ DB จริง — ตอบ 200 เมื่อพร้อม และ **503 เมื่อ DB ล่ม โดยแอปต้องไม่ crash**
- แอปอ่าน config จาก env (DB, port) และฟังที่ `0.0.0.0` ภายใน container
- **build ได้โดยไม่ต้องต่อ DB** และไม่ต้องใช้ความลับ
- มี `.env.example` (ค่าตัวอย่างเท่านั้น), `.gitignore` กัน `.env`

### Docker
- `Dockerfile` แบบ multi-stage — image สำหรับรันเล็ก, **รันด้วย user ที่ไม่ใช่ root**, มี `HEALTHCHECK`
- ถ้ามี migration: แยก image/stage สำหรับ migrate ที่รันแล้วจบ (seed ต้องรันซ้ำได้)
- `.dockerignore` กัน `.env*`
- `docker-compose.yml` ใช้ได้ทั้งในเครื่องและบน VPS: DB มี healthcheck + named volume + **ไม่เปิด port ออกนอก**, ลำดับเริ่ม db → migrate → app, ชื่อ image อ่านจาก env (`APP_IMAGE`, `MIGRATOR_IMAGE`), port บนเครื่องอ่านจาก `APP_PORT`
- port DB สำหรับ dev ให้อยู่ใน `docker-compose.override.yml`

### CI/CD
- `ci.yml` (ชื่อ workflow `CI`) — ตรวจโค้ดเท่าที่โปรเจกต์มี + ทดลอง `docker build`
- `deploy.yml` — รันหลัง CI ผ่านบน `main` (และสั่งเองได้), build + push image ไป GHCR ด้วย `GITHUB_TOKEN` (tag `latest` และ `sha-xxxxxxx`), deploy ผ่าน SSH ด้วย Secrets (`VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`) ไปที่โฟลเดอร์ของแอปนี้, ตรวจ health หลัง deploy, **job deploy ข้ามได้ถ้ายังไม่ได้ตั้งค่า VPS**
- ตัวแปรที่ฝังตอน build ส่งเป็น build-arg

### ไฟล์สำหรับ VPS
- วิธีสร้าง `.env` บน VPS ที่สุ่มรหัสผ่านและไม่ต้องพิมพ์ความลับเอง, `APP_PORT` bind ที่ `127.0.0.1`
- config Nginx (reverse proxy) สำหรับโดเมนของแอปนี้

### ตัวอย่างอ้างอิง (ใช้ดูแนวทาง ไม่ต้องลอก)
ไฟล์ใน repo Lab: https://github.com/suriyapi/nextjs-crud-starter
— `Dockerfile`, `docker-compose.yml`, `docker-compose.override.yml`, `.dockerignore`, `.github/workflows/ci.yml.disabled`, `.github/workflows/deploy.yml.disabled`, `deploy/make-env.sh`, `deploy/nginx.conf`

---

## 5. เช็กลิสต์ (AI ตรวจก่อนส่งงาน)

- [ ] `docker compose up --build -d` ผ่าน — app/db healthy (migrate จบด้วย exit 0 ถ้ามี)
- [ ] health ตอบ 200 และตอบ 503 เมื่อ `docker compose stop db` แล้วกลับมา 200 หลัง `start db`
- [ ] `.env` ไม่อยู่ใน git และไม่อยู่ใน image
- [ ] container รันด้วย user ที่ไม่ใช่ root
- [ ] port / โฟลเดอร์ / โดเมนบน VPS ไม่ชนกับแอปอื่น
- [ ] ไม่มีความลับในไฟล์ใดๆ ที่ commit

---

## 6. ขั้นที่นิสิตต้องทำเอง (AI สรุปค่าจริงให้)

1. ตรวจไฟล์ แล้ว commit + push → ดูแท็บ Actions
2. DNS: ชี้โดเมน/subdomain ของแอปมาที่ IP ของ VPS
3. บน VPS: สร้างโฟลเดอร์ของแอป, copy ไฟล์ที่จำเป็น, สร้าง `.env`, `docker compose pull && up -d` (ถ้า `unauthorized` ให้ `docker login ghcr.io` ด้วย token `read:packages`)
4. บน VPS (root): ติดตั้ง config Nginx ของโดเมนนี้ → `nginx -t` → reload → `certbot --nginx -d <โดเมน> --redirect`
5. GitHub ของ repo นี้: ตั้ง Secrets ของ VPS และค่าที่ workflow ต้องใช้ → Run workflow → ตรวจเว็บจริง

อ้างอิงวิธีทำแต่ละข้อได้จากคู่มือ Lab ส่วนที่ 4–6
