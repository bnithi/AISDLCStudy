# ถอดรหัส Monorepo: ทำไมบริษัทยักษ์ใหญ่และ AI Lab ระดับโลกถึงยอมทิ้ง Git?
### เจาะลึกโครงสร้าง Pain Point ➔ Solution ผ่านบทเรียนของ Google (Piper) และ Anthropic (Scion)

---

## 📌 บทนำ: คำสัญญาของ Monorepo กับความเป็นจริงที่ Git แบกไม่ไหว

ในโลกวิศวกรรมซอฟต์แวร์ ข้อถกเถียงระหว่าง **Monorepo (รวมทุกโค้ดไว้ในที่เดียว)** กับ **Polyrepo (แยก repo ตาม service)** เป็นเรื่องที่คุยกันไม่รู้จบ:

* **Polyrepo:** คล่องตัวในระดับทีมย่อย แต่แลกมาด้วย **"Dependency Hell"** (เวอร์ชัน API ไม่ตรงกัน, Library ตีกัน, การแก้งานข้ามโปรเจกต์ต้องเปิด PR กระจายหลายสิบที่)
* **Monorepo:** มอบ **Single Source of Truth**, Shared Code ที่แท้จริง, และความสามารถในการทำ **Atomic Refactoring** (แก้จุดเดียว อัปเดตทั่วทั้งองค์กรพร้อมกัน)

ทว่าปัญหาใหญ่ที่สุดของ Monorepo คือ **"Scale"** — ยิ่งโค้ดมหาศาล ยิ่งมีนักพัฒนาหรือ AI Agent เข้ามาทำงานพร้อมกัน เครื่องมือมาตรฐานของโลกอย่าง **Git จะเริ่มพังทลายลง** 

บทความนี้สรุปโครงสร้างปัญหา (Pain Points) และวิวัฒนาการของการแก้ปัญหา (Solutions) ผ่านเอกสารประวัติศาสตร์ 2 ฉบับ:
1. **Google (2016):** *"Why Google Stores Billions of Lines of Code in a Single Repository"* (CACM)
2. **Anthropic (2026):** *"Redesigning source control at Anthropic"* (การเปิดตัวระบบ VCS ภายในชื่อ **Scion**)

---

## 🛑 1. Core Pain Points: ทำไม Git ถึงตันเมื่อเจอ Monorepo ระดับมหาศาล?

หลักการทำงานของ Git ถูกออกแบบมาตั้งแต่ปี 2005 ภายใต้สมมติฐานแบบ **Distributed Version Control (DVCS)** เพื่อการพัฒนา Linux Kernel:

| สมมติฐานของ Git | ความเป็นจริงใน Monorepo ยุคใหม่ | ปัญหาที่เกิดขึ้น (Pain Points) |
| :--- | :--- | :--- |
| **Clients are Offline** (ทุกคนต้อง clone ประวัติทั้งหมดมาไว้ในเครื่อง) | โค้ดมีขนาดหลายสิบถึงร้อย GB (Working copy > 25 GB, History > 100 GB) | Disk เครื่อง Dev/CI เต็ม และใช้เวลา Clone นานหลายสิบนาที |
| **Symmetric Architecture** (Client ทุกเครื่องมีความสามารถเท่า Server) | นักพัฒนาและ CI ต้องการเพียงไฟล์ไม่กี่ไฟล์ที่ตนเองแตะต้อง | สิ้นเปลืองทั้ง Network I/O, CPU, และ Memory ในการคำนวณ packfile |
| **Local Commit + Push & Lock** | มีการ merge หลายหมื่นครั้งต่อวัน (Anthropic พุ่งแตะ **30,000+ merges/วัน**) | Server รับโหลด lock/rebase ไม่ไหว เกิด merge conflict มหาศาล |
| **Checkouts are Mandatory** | รัน CI และ Agent รันงานสั้นๆ นับสิบล้านครั้งต่อวัน | เครื่องต้อง materialize ไฟล์ทั้งหมดลง disk ก่อน แค่เพื่ออ่าน diff บรรทัดเดียว |

---

## 🏗️ 2. Solution ยุคบุกเบิก: บทเรียนจาก Google (CACM 2016)

Google มีขนาดโค้ดระดับ **หลายพันล้านบรรทัด (Billions of Lines of Code)** และวิศวกรหลายหมื่นคนทำงานบน Trunk เดียว (Trunk-Based Development)

### แนวคิดหลักในการแก้ปัญหาของ Google:
1. **ทิ้ง Git แล้วสร้าง Piper:**
   * Google ไม่ใช้ Git แต่สร้าง **Piper** ซึ่งเป็น Centralized VCS ที่จัดเก็บข้อมูลบน Bigtable / Cloud Spanner infrastructure
   * ไม่มีการ clone ประวัติทั้งหมดลงเครื่องเด็ดขาด
2. **CitC (Clients in the Cloud) — Virtual File System:**
   * ใช้ FUSE (Filesystem in Userspace) ทำ mount repository เสมือนลงบนเครื่องนักพัฒนา
   * **Lazy Loading:** ไฟล์ใน repo ไม่ได้ถูกดาวน์โหลดลงเครื่องจริงๆ จนกว่าจะมีคำสั่งเปิดอ่านหรือแก้ไข Local disk จะเก็บเฉพาะไฟล์ที่มีการแก้ไข (Delta) เท่านั้น
3. **Hermetic & Distributed Build (Blaze / Bazel):**
   * โค้ดทั้งหมดต้องระบุ dependency ชัดเจน (Build targets)
   * ระบบแคชระดับบิลด์ทั่วโลก (Build Cache) ทำให้ไม่ต้องคอมไพล์โค้ดซ้ำ
4. **Automated Refactoring & Analysis (Rosie & Tricorder):**
   * เมื่อต้องการเปลี่ยน API ภายใน มีระบบอัตโนมัติสร้าง Patch และแยกส่ง Code Review ไปยังเจ้าของแต่ละส่วนพร้อมกันนับร้อยชิ้น

---

## ⚡ 3. Solution ยุค Agentic Era: บทเรียนจาก Anthropic (Scion, 2026)

หาก Google คือตัวแทนของยุคที่ "คนนับหมื่น" เขียนโค้ด **Anthropic คือตัวแทนของยุคที่ "AI Agents นับหมื่นตัว" เขียนโค้ดและรันงานพร้อมกัน**

Anthropic เผชิญอัตราการเติบโตของ merge จาก 2,500 ครั้ง/วัน สู่ **30,000+ ครั้ง/วัน** ภายใน 4 เดือน จน Git Mirror หลายร้อยตัวรับไม่ไหว ทีมวิศวกร 7 คน (ร่วมกับโมเดล Claude) จึงสร้างระบบ VCS ใหม่ชื่อ **Scion** ภายในเวลา 5 เดือน

### 6 สถาปัตยกรรมหัวใจของ Anthropic Scion:
1. **Always-Online Clients:** สลัดสมมติฐานเรื่อง Offline ทิ้ง เพราะ 99% ของ clients คือ Cloud Agents และ CI Jobs ข้อมูลจะถูกดึงผ่าน Network แบบ On-demand เสมอ
2. **Client-Server Asymmetry:** 
   * ย้ายงานหนัก (History, Blame, Diff, Merge, Rebase) ไปทำบน Server ทั้งหมด
   * Client มีหน้าที่แค่ Bounded Cache และจัดการ Local File Interface
3. **Optional & Virtualized Checkout:**
   * **Sparse Checkout โดยดีไซน์:** CI หรือ Tool ไม่จำเป็นต้องดาวน์โหลดโค้ดทั้ง repo 
   * **Materialize on demand:** เสียค่า I/O เฉพาะไฟล์ที่อ่านจริง
   * หาก CI แค่อยากรู้ diff เพื่อตรวจ linting ก็ยิงขอผ่าน API ได้โดยไม่ต้อง checkout สักไฟล์เดียว
4. **API-First Source Control:** เปิด endpoint ให้ระบบ Automation/AI ถามข้อมูล repo ได้โดยตรง (เช่น ใครแก้บรรทัดนี้, สอง commit นี้เกี่ยวข้องกันอย่างไร) ทำให้ pipeline การ bisect และ revert เกิดขึ้นได้ในระดับมิลลิวินาที
5. **Lock-Free Server-Side Commits:**
   * ตัดระบบ Local Branch/Ref ที่ต้อง sync สถานะทั่วโลกทิ้ง
   * ต้นทุนการ Commit แปรผันตาม "ขนาดของ Change" ไม่ใช่ "ขนาดของ Repo"
6. **Hierarchical Caching:**
   * มี `scion-cache` หลายระดับที่อัตรา Cache hit เกิน **99%**
   * ประหยัดขนาดโครงสร้างพื้นฐาน CPU/RAM ลงถึง **200 เท่า** เมื่อเทียบกับตอนใช้ Git Mirrors

---

## 📊 สรุปตารางเปรียบเทียบ: Git vs Google (Piper) vs Anthropic (Scion)

| มิติการเปรียบเทียบ | Traditional Git | Google (Piper + CitC) | Anthropic (Scion) |
| :--- | :--- | :--- | :--- |
| **รูปแบบสถาปัตยกรรม** | Distributed (P2P-like) | Centralized Cloud VCS | Asymmetric Cloud-Native VCS |
| **การเก็บข้อมูลในเครื่อง** | Full Copy (History + Files) | Virtual FUSE (Cloud-backed) | Sparse & Virtualized on-demand |
| **การจัดการ Merge/Rebase** | ฝั่ง Client คำนวณแล้ว Push | ฝั่ง Server / Trunk Global Lock | ฝั่ง Server แบบ Lock-free & Post-merge testing |
| **ผู้ใช้งานหลัก** | มนุษย์ (Offline/Online) | มนุษย์นับหมื่นคน | มนุษย์ + ฝูง Autonomous AI Agents |
| **จุดเด่น** | เป็นอิสระ ไม่ต้องพึ่งพา Server ตลอดเวลา | ขอบเขตสเกลใหญ่ระดับประวัติศาสตร์ | ความเร็วสูง ทำงานผ่าน API และ Cache 99%+ |

---

## 💡 บทเรียนสำหรับ Tech Leaders และ Software Engineers

1. **Monorepo ไม่ใช่แค่ "โฟลเดอร์รวมโค้ด":** 
   * ถ้าโค้ดยังไม่เยอะ Git + เครื่องมืออย่าง Turborepo, Nx, หรือ Bazel ยังคงเพียงพอ
   * แต่เมื่อเข้าสู่ระดับ Hyper-scale ปัญหาจะไม่ใช่เรื่องของ Code Quality แต่เป็นเรื่องของ **Filesystem I/O, Network Bandwidth, และ VCS Data Structures**
2. **การมาถึงของ AI-generated Code จะบีบให้ Tooling ต้องเปลี่ยน:**
   * เมื่อ Agent ทำงานได้เร็วกว่าคน อัตราการแตก Branch และ Merge จะพุ่งสูงขึ้นเป็นเท่าทวีคูณ
   * ระบบ Source Control ยุคถัดไปจะต้อง **API-First** และ **Compute on Server** เสมือนที่ Anthropic ทำให้เห็น
3. **เลือก Architecture ให้เหมาะกับบริบท:**
   * หากทีมของคุณไม่ได้แตะระดับ 10,000+ merges/วัน อย่าเพิ่งสร้างระบบ VCS เอง
   * แต่จงนำหลักคิดของพวกเขากลับมาประยุกต์ใช้: **Sparse checkout, Trunk-based development, และ Remote caching** เพื่อให้ระบบของคุณเติบโตได้อย่างยั่งยืน

---
**References:**
* Rachel Potvin & Josh Levenberg, *"Why Google Stores Billions of Lines of Code in a Single Repository"*, Communications of the ACM (2016)
* Anthropic Engineering, *"Redesigning source control at Anthropic"* (2026)