# ARUN — Solar ROI Calculator (สูตรและการคำนวณ)

เว็บแอปพลิเคชันคำนวณความคุ้มทุน ผลประหยัด และวิเคราะห์จุดคุ้มทุนการติดตั้ง Solar Rooftop สำหรับที่อยู่อาศัยในประเทศไทย (รองรับเงื่อนไขโครงการโซลาร์ภาคประชาชน, เงินสนับสนุนภาครัฐ 50,000 บาท, สินเชื่อดอกเบี้ยต่ำ และแบตเตอรี่สำรอง)

---

## 1. สูตรและวิธีการคำนวณทั้งหมดในระบบ

### 1.1 การคำนวณหน่วยไฟฟ้าฐานและการแปลงย้อนกลับ (Invert Bill)

ค่าไฟฐานของไทยคิดตามอัตราก้าวหน้า (Progressive Rate) ประเภทบ้านอยู่อาศัยเกิน 150 หน่วย/เดือน (อัตรา 1.2):

$$\text{Base Cost}(u) = \begin{cases} 
u \times 3.2484 & , u \le 150 \\
(150 \times 3.2484) + (u - 150) \times 4.2218 & , 150 < u \le 400 \\
(150 \times 3.2484) + (250 \times 4.2218) + (u - 400) \times 4.4217 & , u > 400 
\end{cases}$$

$$\text{Bill}(u) = \Big( \text{Base Cost}(u) + (u \times Ft) + \text{Service Charge} \Big) \times 1.07 \quad (\text{รวม VAT 7\%})$$

- **ค่า Ft เริ่มต้น:** $0.1623$ บาท/หน่วย
- **ค่าบริการรายเดือน:** $24.62$ บาท/เดือน
- **การประมาณหน่วยไฟฟ้าเดิม ($u$):** ใช้เทคนิค Binary Search (60 รอบ ความแม่นยำสูง) จากยอดบิลค่าไฟที่ผู้ใช้กรอก

---

### 1.2 การผลิตไฟฟ้าจากพลังงานแสงอาทิตย์ (Solar Generation)

$$\text{Gen}_{\text{year}} (\text{kWh}) = \text{kW} \times \text{Yield} \times 365 \times (1 - \text{Shade})$$

- $\text{kW}$: กำลังติดตั้งของระบบ (สเต็ปปรับทีละ 1 kW ในช่วง 1–30 kW)
- $\text{Yield}$: ผลผลิตเฉลี่ยต่อวัน ตั้งต้นที่ $4.2$ kWh/kW/วัน
- $\text{Shade}$: อัตราการสูญเสียจากเงาบังและทิศทางหลังคา (เช่น ปลอดโปร่ง = 0%, บังปานกลาง = 10%)

---

### 1.3 การจัดสรรพลังงานไฟฟ้า (Self-Consumption & Battery)

1. **การใช้ไฟช่วงกลางวัน (Daytime Consumption):**
   $$\text{Demand}_{\text{day}} = u \times 12 \times \text{DayRatio}$$
   $$\text{Self}_{\text{day}} = \min(\text{Gen}_{\text{year}} \times \text{DayRatio},\, \text{Demand}_{\text{day}})$$
   $$\text{Surplus}_{\text{day}} = \max(0,\, \text{Gen}_{\text{year}} - \text{Self}_{\text{day}})$$

2. **ระบบแบตเตอรี่สำรอง (Battery Storage - ถ้าเปิดใช้งาน):**
   - ความจุสุทธิต่อปีที่จ่ายได้:
     $$\text{Bat}_{\text{annual}} = \text{Qty} \times \text{Capacity} \times 365 \times \text{DoD} \times \eta_{\text{rt}}$$
     *(DoD = 90%, Round-trip Efficiency $\eta_{\text{rt}} = 90\%$)*
   - พลังงานชาร์จเข้าแบตเตอรี่:
     $$\text{Bat}_{\text{charged}} = \min(\text{Surplus}_{\text{day}},\, \text{Bat}_{\text{annual}})$$
   - ทดแทนไฟช่วงค่ำ/กลางคืน:
     $$\text{Self}_{\text{night}} = \min(\text{Bat}_{\text{charged}},\, \text{Demand}_{\text{night}})$$
   - ไฟส่วนเกินที่เหลือขายคืนเข้ากริด:
     $$\text{ExportUnits} = \max\left(0,\, \text{Surplus}_{\text{day}} - \frac{\text{Bat}_{\text{charged}}}{\eta_{\text{rt}}}\right)$$

3. **หน่วยไฟที่ลดได้ต่อเดือน & ค่าไฟใหม่:**
   $$\text{Self}_{\text{monthly}} = \frac{\text{Self}_{\text{day}} + \text{Self}_{\text{night}}}{12}$$
   $$\text{Bill}_{\text{after}} = \text{Bill}\Big(\max(0,\, u - \text{Self}_{\text{monthly}})\Big)$$
   $$\text{Saving}_{\text{monthly}} = \max(0,\, \text{Bill}_{\text{orig}} - \text{Bill}_{\text{after}})$$

---

### 1.4 รายได้จากการขายไฟฟ้าส่วนเกิน (Net Billing / Solar Citizen)

$$\text{SellIncome}_{\text{monthly}} = \frac{\text{ExportUnits} \times \text{SellPrice}}{12}$$

- **ราคารับซื้อไฟคืน:** $2.20$ บาท/หน่วย (โครงการ 20 ปี)

---

### 1.5 เงินลงทุนสุทธิ และการคำนวณค่างวดสินเชื่อ (Loan Financing)

1. **เงินลงทุนสุทธิ:**
   $$\text{Net Investment} = \max\Big(0,\, \text{System Price} + \text{Battery Cost} - \text{Subsidy}\Big)$$
   - **เงินสนับสนุนภาครัฐ:** $50,000$ บาท (กรณีเปิดใช้งาน)

2. **ค่างวดสินเชื่อผ่อนชำระ (Flat Rate):**
   $$\text{Total Interest} = \text{Net Investment} \times \text{LoanRate} \times \text{Years}$$
   $$\text{Monthly Payment} = \frac{\text{Net Investment} + \text{Total Interest}}{\text{Years} \times 12}$$
   - ดอกเบี้ยมาตรฐาน: $2.5\%$ ต่อปี
   - ระยะเวลาผ่อนชำระ: $1–15$ ปี (ค่าตั้งต้น 7 ปี)

---

### 1.6 การเปรียบเทียบภาระจ่ายจริง 2 ช่วงเวลา (2-Phase Analysis)

- **ช่วงผ่อนชำระ (ปีที่ 1 ถึง ปีที่ผ่อน):**
  $$\text{Actual Outflow}_{\text{during}} = \text{Bill}_{\text{after}} + \text{Monthly Payment} - \text{SellIncome}_{\text{monthly}}$$
  $$\text{Diff From Baseline} = \text{Bill}_{\text{orig}} - \text{Actual Outflow}_{\text{during}}$$
  *(หาก Diff เป็นบวก แปลว่าจ่ายค่างวดรวมค่าไฟใหม่แล้ว ยังประหยัดกว่าค่าไฟเดิมทันที)*

- **ช่วงหลังปลดหนี้แล้ว (ปีที่หมดงวดผ่อนเป็นต้นไป):**
  $$\text{Actual Outflow}_{\text{after}} = \max(0,\, \text{Bill}_{\text{after}} - \text{SellIncome}_{\text{monthly}})$$
  $$\text{Monthly Net Saving} = \text{Bill}_{\text{orig}} - \text{Actual Outflow}_{\text{after}}$$

---

### 1.7 กระแสเงินสดสะสม 20 ปี และระยะเวลาคืนทุน (Cash Flow & Payback Period)

ในแต่ละปี $t$ (ตั้งแต่ปีที่ 1 ถึง 20):

$$\text{Degradation Factor}(t) = (1 - \text{Degradation Rate})^{t-1}$$
$$\text{Tariff Rise Factor}(t) = (1 + \text{Tariff Rise Rate})^{t-1}$$

$$\text{CashFlow}(t) = \Big(\text{Saving}_{\text{monthly}} \times 12 \times \text{Tariff Rise Factor}(t) \times \text{Degradation Factor}(t)\Big)$$
$$+ \Big(\text{ExportUnits} \times \text{SellPrice} \times \text{Degradation Factor}(t)\Big) - \text{Maintenance}$$

- **การเสื่อมของแผง (Degradation):** $0.5\%$ ต่อปี
- **การเพิ่มขึ้นของค่าไฟ (Tariff Rise):** $2.0\%$ ต่อปี
- **ค่าบำรุงรักษา:** $1,500$ บาท/ปี
- **กระแสเงินสดสะสม:**
  $$\text{Cumulative}(0) = -\text{Net Investment}$$
  $$\text{Cumulative}(t) = \text{Cumulative}(t-1) + \text{CashFlow}(t)$$

**ระยะเวลาคืนทุนแบบละเอียด (Payback Period):**
คำนวณจุดตัดที่เงินสะสมข้ามจากติดลบเป็นบวกด้วยสมการเชิงเส้น:
$$\text{Payback} = (t - 1) + \frac{|\text{Cumulative}(t-1)|}{\text{CashFlow}(t)} \quad (\text{หน่วย: ปี})$$

**ผลตอบแทนการลงทุนต่อปี (Simple Annual ROI %):**
$$\text{Annual ROI} = \frac{(\text{Saving}_{\text{monthly}} + \text{SellIncome}_{\text{monthly}}) \times 12}{\text{Net Investment}} \times 100\%$$

---

## 2. สรุปค่าตัวแปรตั้งต้นที่ใช้ในระบบ

| รายการ | ค่าเริ่มต้น | หน่วย | คำอธิบาย |
| :--- | :---: | :---: | :--- |
| ค่าไฟเฉลี่ยต่อเดือน | 2,000 | บาท | ค่าไฟเดิมของผู้ใช้ |
| กำลังติดตั้ง | 5 | kW | ปรับได้ทีละ 1 kW (1–30 kW) |
| ราคาประเมินรวมติดตั้ง | 150,000 | บาท | ปรับตามสูตรหรือแก้ไขเองได้ |
| เงินสนับสนุนภาครัฐ | 50,000 | บาท | หักลดเงินลงทุนทันที |
| ดอกเบี้ยสินเชื่อ | 2.5 | % ต่อปี | Flat Rate โครงการภาครัฐ |
| ระยะเวลาผ่อนชำระ | 7 | ปี | ปรับได้ 1–15 ปี |
| ราคารับซื้อไฟคืน | 2.20 | บาท/หน่วย | สัญญา 20 ปี โซลาร์ภาคประชาชน |
| ค่า Ft | 0.1623 | บาท/หน่วย | งวดปัจจุบัน |
| ประสิทธิภาพการผลิต | 4.2 | kWh/kW/วัน | ค่าเฉลี่ยแสงแดดในไทย |
| แบตเตอรี่สำรอง | 5 | kWh/ชุด | ราคา 100,000 บาท/ชุด |
