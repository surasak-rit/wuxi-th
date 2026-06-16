# 10 — Data Schemas & Reference (สำหรับทีมพัฒนา)

> ออกแบบเกมแบบ **data-driven** — วิชา/ศัตรู/ไอเทม/เควส เก็บเป็นข้อมูล แก้สมดุลได้โดยไม่แตะโค้ด
> สคีมาด้านล่างเป็นแนวทางตั้งต้น (ปรับตาม stack จริง) สอดคล้องสูตรใน docs/01–03

---

## 1. Skill Schema
```jsonc
{
  "id": "cloud_sword_basic",
  "name": "กระบี่พื้นฐานเมฆขาว",
  "type": "active",                 // active | passive
  "category": "sword",              // sword|palm|poison|qi|light
  "grade": "common",                // common(70)|fine(80)|superb(90)|supreme(100)
  "element": "blade",
  "base_mult": 1.1,
  "mp_cost": 6,
  "max_level": 70,                  // ระดับสูงสุดตาม grade (สเกล 1–100)
  "targeting": "single",            // single|aoe|self|ally|all_allies
  "shape": { "type": "single" },    // ทรงพื้นที่ AoE → ดูเต็มใน docs/12-skill-shapes.md
  "range": {                        // สูตรระยะฐานร่วมทุกวิชา (docs/12 §3.3) — growth/cap เท่ากันทั้งเกม
    "base_range": 3,                // ชั้นระยะ: น้อย=2 / กลาง=3 / มาก=4 (ต่างกันแค่ ±1)
    "range_growth": 0.04, "range_cap_level": 100
  },
  "effects": [
    { "kind": "damage" },
    { "kind": "status", "status": null, "chance": 0.0, "duration": 0 },
    { "kind": "lifesteal", "pct": 0.0 },
    { "kind": "mpsteal", "pct": 0.0 }
  ],
  "level_unlocks": {                 // ผลพิเศษตามระดับ (มาตรฐาน docs/02 §2)
    "40": "effect_chance+0.05",
    "60": "self_action_spd+0.10",
    "80": "effect_chance+0.05",
    "100": "supreme_passive"         // เฉพาะ grade=supreme
  },
  "requirements": {                  // ปลดล็อกเรียน — ไม่อิงเลเวลตัวละคร (docs/02 §1)
    "attributes": { "arm": 8 },
    "comprehension": 8,
    "fame": 1,
    "sect": "cloud_peak",
    "prereq": []                     // [{ "skill": "...", "level": 50 }]
  }
}
```

## 2. Enemy Schema (ดู docs/08 §2)
```jsonc
{
  "id": "bandit_leader_01",
  "name": "หัวหน้าโจรเขาเป่ย",
  "tier": "elite",                  // common|elite|region_boss|world_boss
  "PI": 520,
  "stats": { "HP":4200,"MP":300,"ATK":180,"DEF":95,"ACC":130,"EVA":35,"SPD":60,"CRIT":0.08 },
  "armor_type": "leather",          // leather|plate|cloth (มีผลกับ element, docs/01 §5)
  "status_res": 0.15,
  "ai_script": [
    { "if": "enemies>=2", "do": "skill:sweep_aoe" },
    { "if": "self_hp<0.5", "do": "skill:battle_roar" },
    { "else": true,        "do": "skill:double_slash" }
  ],
  "phases": [],                     // บอสเท่านั้น (ดู §3)
  "loot_table": "bandit_elite_drops",
  "base_exp": 380,
  "respawn_sec": 120
}
```

## 3. Boss Phase Schema (ดู docs/08 §3)
```jsonc
{
  "phases": [
    { "hp_above": 0.70, "ai_script": [/* ... */] },
    { "hp_above": 0.35, "ai_script": [/* ... */], "on_enter": "buff:summon_minions" },
    { "hp_above": 0.00, "ai_script": [/* ... */], "on_enter": "buff:atk+0.5" }
  ],
  "enrage_round": 30,
  "enrage_effect": "boss_damage_x3"
}
```

## 4. Item Schema (ดู docs/05)
```jsonc
{
  "id": "azure_longsword",
  "name": "กระบี่ฟ้าคราม",
  "slot": "weapon",                 // weapon|armor|head|boots|accessory|charm|consumable|material|manual
  "weapon_type": "sword",
  "rarity": "blue",                 // white|green|blue|purple|orange
  "ilvl": 45,
  "bind": false,
  "base_stats": { "weapon_atk": 60, "CRIT": 0.03 },
  "sockets": 1,
  "enhance_level": 0,               // +0..+10 (docs/05 §5)
  "element": "blade",
  "elemental_imprint": {            // ตราธาตุจากช่างสายธาตุ (docs/14 §5.2) — null ถ้าไม่มี
    "element": null,                // sun | star | moon
    "elem_power": 0,                // ค่าโจมตีพิเศษธาตุ
    "proc_chance": 0.0
  },
  "requirements": { "attributes": { "arm": 30 }, "fame": 2 }
}
```

## 5. Loot Table Schema (ดู docs/08 §6)
```jsonc
{
  "id": "bandit_elite_drops",
  "gold": { "min": 50, "max": 180 },
  "entries": [
    { "item": "azure_longsword", "chance": 0.04, "qty": 1 },
    { "item": "iron_ore",        "chance": 0.5,  "qty": [1,3] },
    { "item": "manual_cloud_adv","chance": 0.01, "qty": 1, "pity": 50 }
  ]
}
```

## 6. Character Save Schema (ผู้เล่น)
```jsonc
{
  "id": "player_uuid",
  "name": "เย่หลิง",
  "sect": "cloud_peak",
  "attributes": { "arm": 22, "bone": 16, "agi": 18, "courage": 12, "comprehension": 40, "recovery": 10, "fortune": 8 },  // 7 ค่าแบบ JY (docs/15)
  "combat_exp": 1820,               // สกุลเงินฝึกวิชา (มีเพดานตาม fame)
  "fame_level": 3,
  "karma": 0,
  "skills": [                       // ระดับ (1–100) + ความชำนาญ (docs/02 §3)
    { "id": "cloud_sword_basic", "level": 70, "prof": 120 },
    { "id": "cloud_nine_heavens", "level": 40, "prof": 30 }
  ],
  "equipped_active": ["cloud_nine_heavens","..."],   // ≤4
  "equipped_passive": ["cloud_inner_qi","..."],      // ≤2
  "skill_loadouts": { "pve": [...], "pvp": [...], "boss": [...] },
  "inventory": [/* item instances */],
  "equipment": { "weapon": {...}, "armor": {...} },
  "rested_pool": 0
}
```
> ทุกค่าที่กระทบสมดุล (combat_exp, fame, skills, attributes) ต้องตรวจสอบ/อัปเดต
> ฝั่ง **server-authoritative** เท่านั้น (docs/09 §8)

---

## 7. Reference: ลำดับคำนวณค่าสถานะสุดท้าย (Stat Pipeline)
```
1. base = สูตรจากค่าพื้นฐาน (docs/01 §3)
2. + flat จากอุปกรณ์ (weapon_atk, armor_def, +HP ฯลฯ) + หยกสลัก
3. * (1 + Σ passive_%บัฟ)        # 内功 แบบเปอร์เซ็นต์ (docs/02 §7)
4. + บัฟ/ดีบัฟ ชั่วคราวในการต่อสู้
5. clamp ค่าที่มีเพดาน (CRIT ≤ 0.50)
→ ค่าที่ใช้จริงในการคำนวณการปะทะ (docs/01 §4)
```

---

## 8. อภิธานศัพท์ (Glossary TH ↔ EN/term)
| ไทย | อังกฤษ/โค้ด | หมายเหตุ |
|---|---|---|
| ประสบการณ์ต่อสู้จริง | Combat EXP | สกุลเงินฝึกวิชา ไม่ใช่เลเวล |
| ค่าความเข้าใจ | Comprehension (COM) | เพดานระดับวิชาที่ฝึกได้ |
| ชื่อเสียง | Fame | เพดานสะสม EXP / ปลดวิชา/สิทธิ์ |
| ดัชนีพลังรวม | Power Index (PI) | ใช้แทนเลเวลทั้งเกม |
| ระดับวิชา | Skill Level | ระดับความแรงของวิชา 1–100 |
| ความชำนาญ | Proficiency (prof) | สะสมจากการใช้วิชา ลดต้นทุนฝึก |
| พลังภายใน/ลมปราณ | Inner Power / MP (内功/MP) | passive=内功, ทรัพยากร=MP |
| ฝีเท้า | Speed (SPD) | ลำดับออกท่า |
| ค่าสังหาร | Karma (正邪) | สถานะ PK |
| สำนัก (NPC) | Sect | สำนักวิชาในเกม |
| สำนักผู้เล่น | Guild/Player Sect | องค์กรผู้เล่น |
| นั่งสมาธิ/เดินลมปราณ | Meditate (运功) | ฟื้นฟู + rested |
| เพดานแต้ม | EXP Cap | ผูกกับ Fame |

---

## 9. Open Questions สำหรับรอบ playtest (ค่าที่ต้องจูน)
- [ ] ค่า `base_exp` ของมอนแต่ละ tier เทียบกับ train_cost (เป้า: วิชาสุดยอด ~หลายสัปดาห์)
- [ ] อัตรา `attrCost` แพงไป/ถูกไปไหม (เป้า: ค่าพื้นฐานโตช้ากว่าวิชาชัดเจน)
- [ ] เพดาน CRIT 0.50 และ crit_mult 1.5 สมดุลกับสาย Assassin หรือไม่
- [ ] ความรุนแรง element modifier (±10–15%) มากไป/น้อยไป
- [ ] diminishing ของ status resist กัน perma-lock ได้จริงใน PvP ไหม
- [ ] อัตรา faucet/sink ของเงิน — monitor เงินเฟ้อสัปดาห์แรกหลัง launch
