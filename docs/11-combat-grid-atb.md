# 11 — ระบบต่อสู้บนกริดแบบ Real-time ATB (Grid Real-Time ATB Combat)

> **ระบบต่อสู้หลักของเกม (Real-time Action)** — ทุกหน่วยมี **หลอดคูลดาวน์ (Action Gauge)**
> อิสระที่เติมตามเวลาจริง, เคลื่อนที่บนกริดพร้อมกันทุกตัวโดยไม่สลับเทิร์น, เมื่อหลอดเต็ม
> ใช้สกิล **AoE รอบตัว (Radial)** ใส่ฝูงศัตรู และแสดง **Floating Damage Text** เหนือหัวศัตรู
> ทุกตัวพร้อมกัน
>
> 📌 **ความสัมพันธ์กับ docs/01:** ระบบนี้ใช้ **ค่าสถานะ/สูตรดาเมจ/สถานะผิดปกติชุดเดียวกับ
> docs/01 §3–4, §7** (ฐานคณิตที่ใช้ร่วมกัน) ส่วนกลไก "turn/รอบ" ใน docs/01 ถูกแทนที่ด้วย
> real-time ATB ในเอกสารนี้ — progression/วิชา/ไอเทม/EXP ทั้งหมดไม่เปลี่ยน

---

## 1. ภาพรวมการออกแบบ
| มิติ | การออกแบบ |
|---|---|
| เข้าสู่ฉาก | **ตัดเข้าฉากต่อสู้ (encounter transition)** เมื่อปะทะศัตรูบนแผนที่โลก |
| สนามรบ | กริด 2D ขนาดปรับตาม encounter — **รองรับทั้งสี่เหลี่ยม (square) และหกเหลี่ยม (hex)** |
| เวลา | **Real-time** จำลองด้วย fixed tick (server-authoritative) |
| จังหวะการกระทำ | **ATB อิสระต่อหน่วย** — หลอดเติมตาม SPD, เต็มแล้วลงมือทันที |
| การเคลื่อนที่ | ต่อเนื่องบนกริด ทุกหน่วยพร้อมกัน (ไม่มีเทิร์น) |
| สกิลเด่น | **Radial AoE** รอบตัว โดนศัตรูที่ล้อมอยู่ทุกตัวในรัศมีพร้อมกัน |
| ฟีดแบ็ก | Floating Damage Text หลายตัวเลขพร้อมกันใน frame เดียว |

> **Core fantasy ของโหมดนี้:** "จอมยุทธ์ยืนกลางวงล้อม สะบัดท่าเดียวกระจายศัตรูทั้งวง" —
> เน้นความมันของการล้างฝูง (horde clearing) + การวางตำแหน่งให้ศัตรูกระจุกเข้ารัศมี

---

## 2. การตัดเข้าฉากต่อสู้ (Encounter Transition)
เมื่อผู้เล่นปะทะศัตรูบนแผนที่โลก (เดินชน/ถูกลากเข้า aggro/เริ่มภารกิจรบ) **ระบบตัดเข้าสู่
ฉากต่อสู้** แล้วสร้างสนามกริดสำหรับการปะทะนั้น
```
ทริกเกอร์ → transition (เฟด/ซูม ~0.5–1s) → สร้างสนามกริด → จัดวางหน่วยเริ่มต้น → เริ่มจำลอง
```
- **เลือกเทมเพลตสนาม** ตามภูมิประเทศจุดปะทะ (ป่า/เมือง/ถ้ำ) + ขนาดตามจำนวนศัตรู
- **จัดวางเริ่มต้น (spawn):** ฝ่ายผู้เล่นโซนหนึ่ง ศัตรูอีกโซน; ถ้าถูก "ลอบโจมตี" ศัตรูอาจ
  ล้อมรอบ (ได้เปรียบ Radial!) ถ้าผู้เล่น "เปิดฉากก่อน" ได้บัฟ Action Gauge เริ่มต้น
- จบฉาก: ชนะ/แพ้/หนี → คำนวณผล (EXP/drop ตาม docs/03,08) → เฟดกลับแผนที่โลก
> รูปแบบ "ตัดเข้าฉาก" แยกสนามรบออกจากแผนที่โลก ทำให้คุมขนาดกริด/จำนวนศัตรู/สมดุล
> ได้ชัด และรองรับทั้ง random encounter และบอสที่ออกแบบสนามเฉพาะ

## 3. สนามรบกริด (Battlefield Grid) — Square / Hex
```
- ระบบพิกัด: ตาราง 2D, 1 cell ≈ 1 เมตร (conceptual)
- ขนาดมาตรฐาน: 16×16 (ปกติ) ... สูงสุด 24×24 (ศึกฝูงใหญ่/raid)
- พิกัดหน่วย: ตำแหน่งต่อเนื่อง (float) ไม่สแนปกลางช่อง → เคลื่อนที่ลื่น (กริดใช้คิดระยะ/วาง)
- footprint: หน่วยปกติ 1 ช่อง, บอส 2×2 หรือ 3×3
- terrain flags ต่อช่อง: walkable / blocked / hazard(พื้นพิษ-ไฟ) / high-ground(+ระยะ/ดาเมจ)
```
**รองรับ 2 โทโพโลยี (เลือกต่อโปรเจกต์/ต่อสนาม) — กริดใช้ทั้งเคลื่อนที่และกำหนดระยะโจมตี:**
| | สี่เหลี่ยม (Square) | หกเหลี่ยม (Hex) |
|---|---|---|
| เพื่อนบ้านต่อช่อง | 4 (ตั้งฉาก) หรือ 8 (รวมทแยง) | 6 (สม่ำเสมอทุกทิศ) |
| สูตรระยะ (range) | Chebyshev `max(|dx|,|dy|)` หรือ Euclidean | Cube/Axial: `(|dq|+|dr|+|dq+dr|)/2` |
| ข้อดี | สร้าง/วาง terrain ง่าย, UI คุ้นตา | ระยะ/รัศมีสมมาตร "วงกลม" สวย เหมาะ Radial AoE |
| ข้อควรระวัง | ทแยงเพี้ยนระยะ (ต้องเลือก metric ชัด) | คณิต/พิกัดซับซ้อนกว่า, art ต้องออกแบบเฉพาะ |
```python
# ระยะที่ใช้ทั้งการเคลื่อนที่และ "ระยะการโจมตี/รัศมี AoE"
def grid_dist(a, b, mode):
    if mode == "square_8":  return max(abs(a.x-b.x), abs(a.y-b.y))     # Chebyshev
    if mode == "square_4":  return abs(a.x-b.x) + abs(a.y-b.y)         # Manhattan
    if mode == "hex":       return (abs(a.q-b.q)+abs(a.r-b.r)+abs(a.q+a.r-b.q-b.r))//2
# Radial AoE: เป้าที่ grid_dist(caster, e) <= skill.radius (หน่วยเป็น "ช่อง")
# ระยะโจมตีสกิล (range): grid_dist(caster, target) <= skill.range
```
> **ค่าเริ่มต้นของโปรเจกต์ = กริดสี่เหลี่ยม (square, 8 ทิศ)** — เลือกเพราะสร้าง terrain/UI
> เร็วและคุ้นตาผู้เล่น เหมาะกับการผลิตคอนเทนต์จำนวนมาก (เมือง/ป่า/ถ้ำ) ใช้ระยะแบบ
> **Chebyshev** เพื่อให้รัศมี Radial AoE เป็น "วงสี่เหลี่ยม" ที่ทแยงไม่เพี้ยน
> (hex ยังคงเปิดเป็นตัวเลือกผ่าน config สำหรับสนามพิเศษ/บอสที่อยากได้รัศมีวงกลมสมมาตร —
> logic เลือกเป้าทั้งหมดอ้าง `grid_dist()` ตัวเดียว จึงสลับ mode ได้โดยไม่แตะโค้ดสกิล)
- **Collision:** soft collision — หน่วยไม่ทับช่องศูนย์กลางกัน ใช้ steering ดันแยกเบาๆ; ศัตรู
  body-block ได้ (ใช้ตั้งแนว tank ขวางทาง)
- **Pathfinding:** A* บนกริด (square/hex graph) + local avoidance สำหรับหลบหน่วยอื่น/hazard

---

## 4. โมเดลเวลา & ลูปจำลอง (Simulation Model)
```
- SIM_TICK = 20 Hz  → dt = 0.05s  (server จำลองและตัดสินผลทั้งหมด)
- Client: เรนเดอร์ 60 fps, interpolate ตำแหน่งระหว่าง tick (ลื่นไหล)
- Server-authoritative: ตำแหน่ง/หลอด/ดาเมจ/ตาย ตัดสินที่ server เท่านั้น (docs/09 §8)
- Determinism: ใช้ seeded RNG ต่อ encounter เพื่อ replay/ตรวจสอบ/กันโกง
```
**ลูปต่อ 1 tick:**
```
for each tick (dt = 0.05s):
  1. รับ input/คำสั่ง (เคลื่อนที่, สั่งสกิล, retarget) ของผู้เล่น + ตัดสิน AI ศัตรู
  2. อัปเดตการเคลื่อนที่ทุกหน่วยพร้อมกัน (pathfollow + avoidance + collision)
  3. เติม Action Gauge ทุกหน่วย: gauge += fill_per_sec * dt   (§5)
  4. resolve actions: หน่วยที่ gauge เต็ม + มีคำสั่ง/สคริปต์พร้อม → execute (§7)
  5. tick เอฟเฟกต์ต่อเนื่อง (DoT พิษ/ไฟ, พื้น hazard, บัฟ/ดีบัฟ หมดอายุ)
  6. ตรวจการตาย/เงื่อนไขจบ → ส่ง event ออก (รวม batched damage events ของ tick นี้)
```

---

## 5. หลอดคูลดาวน์ / Action Gauge (ATB)
หัวใจของระบบ — แต่ละหน่วย (ผู้เล่น + ศัตรู) มีหลอดของตัวเองที่เติม **อิสระ** ตามเวลาจริง
```python
GAUGE_CAP   = 1000        # หลอดเต็มที่ 1000
BASE_CT     = 3.0         # เวลาชาร์จอ้างอิง (วินาที) ที่ SPD = SPD_REF
SPD_REF     = 100         # ค่า SPD อ้างอิง (ระดับกลางเกม)
CT_MIN, CT_MAX = 0.8, 5.0 # ขอบเขตเวลาชาร์จ (กัน fast เกิน/ช้าเกิน)
OVERFLOW_CAP = 1500       # ชาร์จล้นเก็บได้ถึง 1.5 หลอด (ไม่เสียเวลาตอนกำลังเดิน/ติดสถานะ)

def charge_time(SPD):
    # diminishing returns: SPD สูงเร็วขึ้นแต่ไม่ทบเป็นเส้นตรง
    return clamp(BASE_CT * SPD_REF / (SPD + SPD_REF), CT_MIN, CT_MAX)

def fill_per_sec(unit):
    base = GAUGE_CAP / charge_time(unit.SPD)
    haste = product(unit.haste_multipliers)   # บัฟเร่ง/ดีบัฟหน่วง (เช่น ตรึง=0)
    return base * haste

# ทุก tick:
unit.gauge = min(OVERFLOW_CAP, unit.gauge + fill_per_sec(unit) * dt)
unit.ready = unit.gauge >= GAUGE_CAP
```
**ตารางจังหวะตาม SPD** (SPD = AGI*2 + FOC*0.5, ดู docs/01 §3):
| SPD | charge | actions/นาที | ช่วงตัวละคร |
|---|---|---|---|
| 12.5 | 2.67s | ~22 | เริ่มเกม |
| 60 | 1.88s | ~32 | ต้น-กลาง |
| 100 | 1.50s | ~40 | กลาง |
| 175 | 1.09s | ~55 | ปลาย |
| 237.5 | 0.89s | ~67 | เกือบสุด |

**การลงมือ (เมื่อ gauge เต็ม):**
- ผู้เล่น: ทำตามคำสั่งที่สั่งค้างไว้ หรือ **แผนการรบ/สคริปต์** (docs/01 §6) ถ้า auto;
  ถ้ายังไม่สั่ง หลอดค้างที่เต็ม (สูงสุด OVERFLOW_CAP) รอจังหวะ
- ลงมือแล้ว: `gauge -= action.gauge_cost` (ปกติ = GAUGE_CAP; สกิลเบาคอสต์น้อย, ท่าหนักมาก)
- **cast_time / recovery:** สกิลหนักมี windup (เล่นอนิเมชัน, ขยับช้าลง) + recovery lock สั้นๆ
  หลังออกท่า → เปิดช่องให้ศัตรูสวน (counter-play)

> **เหตุผลเชิงดีไซน์:** หลอดอิสระ + diminishing ของ SPD ทำให้สาย "เร็ว" (ลมวสันต์) ออกท่า
> ถี่กว่าชัดเจน แต่ไม่ถึงกับครองเกม; overflow buffer ให้สาย SPD สูงไม่เสียเปล่าตอนต้องเดิน
> เข้าหา/หลบ — ตำแหน่ง (positioning) จึงมีค่าเท่ากับ raw speed

---

## 6. การเคลื่อนที่ (Movement) — พร้อมกันทุกหน่วย
การเคลื่อนที่ **แยกอิสระจาก Action Gauge** — เดินได้ตลอดแม้หลอดยังไม่เต็ม
```python
BASE_MOVE = 2.0          # tiles/sec ฐาน
def move_speed(unit):
    return clamp(BASE_MOVE * (1 + unit.AGI/120), 1.5, 6.0)   # tiles/sec
```
| AGI | ความเร็ว (tiles/s) |
|---|---|
| 5 | 2.08 |
| 40 | 2.67 |
| 100 | 3.67 |
| 240 | 6.00 |

- **ระหว่าง cast ท่าหนัก:** ความเร็วลดลง (เช่น ×0.3) หรือหยุด ตาม skill flag `move_while_cast`
- **คำสั่งผู้เล่น (mobile-first UX):** แตะพื้น = เดินไปจุด, ลากนิ้ว = เดินต่อเนื่อง,
  ปุ่มสกิล = ออกท่า ณ ตำแหน่งปัจจุบัน; รองรับ auto-move เข้าหากระจุกศัตรู
- **AI ศัตรู:** เดินเข้าหาเป้า/รักษาระยะ (ranged)/ล้อมวง (flanker) — ดู §8

> **เหตุผล:** แยกเดินออกจากหลอด ทำให้เกิด gameplay "วิ่งจัดตำแหน่งให้ศัตรูกระจุกเข้ารัศมี
> ก่อนหลอดเต็ม แล้วปล่อย Radial ทีเดียว" — เป็นแกนความสนุกของโหมดนี้

---

## 7. การกระทำ & สกิล AoE รอบตัว (Actions / Radial AoE)
เมื่อหลอดเต็ม หน่วยเลือก action 1 อย่าง — **Radial AoE คือท่าหลักสำหรับล้างฝูง**
```python
# โครงสกิล Radial (ต่อยอดจาก skill schema docs/10)
{
  "shape": "radial",          # radial | line | cone | single | self
  "radius": 2.5,              # รัศมีเป็น tile (วัดจากจุดศูนย์กลางผู้ร่าย)
  "max_targets": 12,          # เพดานจำนวนเป้า (เลือกใกล้สุดก่อน) — สมดุล+performance
  "falloff": {"inner": 1.5, "outer_mult": 0.7},  # ในรัศมี inner เต็ม, นอกออก -30%
  "gauge_cost": 1000,
  "mp_cost": 22,
  "cast_time": 0.25, "recovery": 0.35,
  "base_mult": 1.3, "element": "blade"
}
```
**ขั้นตอน resolve Radial AoE (เกิดใน tick เดียว → ดาเมจพร้อมกัน):**
```python
def resolve_radial(caster, skill):
    if caster.mp < skill.mp_cost: return fallback_basic(caster)
    caster.mp -= skill.mp_cost
    # 1) หาเป้าทุกตัวในรัศมี
    targets = [e for e in enemies_of(caster)
               if dist(caster.pos, e.pos) <= skill.radius and e.alive]
    targets.sort(key=lambda e: dist(caster.pos, e.pos))
    targets = targets[:skill.max_targets]
    # 2) คำนวณดาเมจต่อเป้า (ใช้สูตร docs/01 §4 ทั้งหมด)
    events = []
    for t in targets:
        if not roll_hit(caster, t):                  # ACC vs EVA
            events.append(dmg_event(t, kind="miss")); continue
        mult = skill.base_mult + caster.skill_star*0.10
        if dist(caster.pos,t.pos) > skill.falloff.inner:
            mult *= skill.falloff.outer_mult         # ดาเมจลดตามระยะ
        crit = roll_crit(caster, t)
        dmg  = damage_formula(caster, t, mult, crit) # raw=ATK*mult-DEF*0.5 ...
        apply_damage(t, dmg)
        events.append(dmg_event(t, dmg, crit=crit))
        if skill.status: try_apply_status(t, skill)  # พิษ/มึน ราย target
    caster.gauge -= skill.gauge_cost
    emit_batched(events)        # ⬅ ส่งทุก event พร้อมกัน → floating text simultaneous (§8)
    return events
```
**ประเภท shape อื่น (ใช้ซ้ำ engine เดียว):** `line` (ทะลุแนว), `cone` (กรวยหน้า),
`single` (เป้าเดียวแรง), `self` (บัฟ/ฟื้น) — ต่างกันที่ฟังก์ชันเลือกเป้า

> **เหตุผล:** `max_targets` + `falloff` คุมทั้งสมดุล (ไม่ให้ AoE ล้างทุกอย่างไร้ขีดจำกัด)
> และ performance (จำกัดงานต่อ frame); การ "ดึงศัตรูเข้า inner radius" ให้รางวัลดาเมจเต็ม
> = ทักษะการวางตำแหน่งมีผลต่อ output จริง

---

## 8. Floating Damage Text (ตัวเลขความเสียหายลอย) — พร้อมกัน
```python
# ดาเมจทุกเป้าจาก action เดียวถูก emit ใน batch เดียวกัน (timestamp = tick เดียวกัน)
DamageEvent = { target_id, amount, kind, world_pos, tick }
# kind: normal | crit | poison | bleed | heal | miss | immune
```
**สเปกการแสดงผล (client):**
- ทุกตัวเลขของ batch เด้งขึ้น **พร้อมกัน** (เฟรมเดียว) → สื่อความรู้สึก "ฟันทีเดียวโดนทั้งวง"
- สี/สไตล์ตามชนิด: ปกติขาว, **คริติคอลใหญ่+เหลือง/แดง สั่น**, พิษเขียว, ฟื้นเขียวอ่อน, "พลาด" เทา
- **กันตัวเลขซ้อนกัน:** เมื่อหลายตัวเลขใกล้กัน กระจายมุม/หน่วงเสี้ยววินาที (jitter 0–80ms)
  เชิงภาพ แต่ค่ายังคิดพร้อมกันใน tick เดียว
- ตัวเลขรวม (option): แสดง "ดาเมจรวมทั้งวง" สั้นๆ กลางจอเมื่อโดน ≥ N เป้า (ความมัน)
- performance: object pooling, รวม batch, culling นอกจอ; cap floating text ที่แสดงพร้อมกัน

---

## 9. AI ศัตรูบนกริด (Enemy AI)
ต่อยอด ai_script (docs/08 §2) + พฤติกรรมเชิงพื้นที่:
| archetype | พฤติกรรม |
|---|---|
| melee swarm | รุมเข้าประชิด — มักกระจุก → เป็นอาหารของ Radial (จงใจ) |
| ranged | รักษาระยะ ถอยเมื่อถูกประชิด — บีบผู้เล่นให้ไล่/เลือกเป้า |
| flanker | อ้อมตีด้านหลัง/ตัดเป้าหลัง (healer) |
| brute/บอส | footprint ใหญ่ body-block, ท่าหนักมี telegraph |
- **Telegraph:** ก่อนท่าใหญ่/AoE ศัตรู แสดง **เขตเตือนบนกริด** (สีแดง) ช่วงเวลาสั้น →
  ผู้เล่นมีโอกาส "เดินหลบ" (counter-play เชิงตำแหน่งแบบ real-time)
- บอสมีเฟส (docs/08 §3) + รูปแบบ AoE บนพื้น (วงกลม/เส้น/วงแหวน) ที่ต้องหลบ

---

## 10. การใช้สูตร/ค่าร่วมกับเกมหลัก (Consistency)
ใช้ของเดิมทั้งหมด ไม่สร้างเลขใหม่ซ้อน:
- **ค่าสถานะ & สูตรดาเมจ/hit/crit/สถานะ:** docs/01 §3–4 และ §7 (status)
- **SPD** ขับ Action Gauge, **AGI** ขับความเร็วเดิน (ทั้งคู่จากค่าพื้นฐานเดิม)
- **วิชา/ดาว/MP/ความชำนาญ:** docs/02 (เพิ่มแค่ field เชิงพื้นที่: shape/radius/max_targets)
- **EXP/ชื่อเสียง/PI/drop:** docs/03, 08 ใช้ได้ทันที (โหมดนี้แค่เปลี่ยน "วิธีตี")

---

## 11. ค่าคงที่สำหรับจูน (Tuning Constants)
| ค่า | ดีฟอลต์ | ผลกระทบ |
|---|---|---|
| SIM_TICK | 20 Hz | ความละเอียด/โหลด server |
| GAUGE_CAP | 1000 | สเกลหลอด |
| BASE_CT / SPD_REF | 3.0s / 100 | จังหวะออกท่ารวม |
| CT_MIN / CT_MAX | 0.8 / 5.0s | เพดานเร็ว/ช้า |
| OVERFLOW_CAP | 1500 | บัฟเฟอร์หลอดล้น |
| BASE_MOVE / max | 2.0 / 6.0 tiles/s | ความเร็วเดิน |
| Radial radius | 2.0–3.0 tiles | ขนาดวง AoE |
| max_targets | 8–16 | เพดานเป้า/สมดุล+perf |
| grid size | 16×16 (≤24×24) | ขนาดสนาม |
| grid mode | **square_8** (ดีฟอลต์) / hex | โทโพโลยี + metric ระยะ |

---

## 12. ลำดับพัฒนาโหมดนี้ (ต่อจาก roadmap หลัก)
1. **Prototype แกน:** กริด + เคลื่อนที่ real-time + Action Gauge + Radial AoE + floating text (1 สำนัก)
2. AI archetype (swarm/ranged) + telegraph หลบ + collision/pathfinding
3. ครบ 5 สำนัก (map ท่าวิชาเดิม → shape เชิงพื้นที่) + บอสเฟสบนกริด
4. polish: ตัวเลขรวม, juice (สั่นจอ/เอฟเฟกต์), tuning pass, มือถือ UX

---

## 13. Open Questions (ตัดสินใจ/จูนรอบ playtest)
- [x] **ตัดสินแล้ว:** ระบบนี้เป็น **ระบบต่อสู้หลัก** ของเกม (แทน turn-based เดิม docs/01)
- [x] **ตัดสินแล้ว:** โทโพโลยีกริดเริ่มต้น = **สี่เหลี่ยม (square_8, Chebyshev)** — hex เป็นออปชัน
- [ ] PvP ใช้ real-time grid ด้วยไหม (sync/latency) หรือ PvP คงเป็น turn-based?
- [ ] ผู้เล่นคุมเอง real-time แค่ไหน vs auto-script (docs/01 §6) — สำคัญต่อ feel มือถือ
- [ ] Radial เป็น "instant รอบตัว" หรือมี cast windup ให้หลบได้ (กระทบ counter-play)
- [ ] max_targets/radius/ดาเมจ falloff — จูนให้ "ล้างฝูงมัน" แต่ไม่ทำ single-target ไร้ค่า
