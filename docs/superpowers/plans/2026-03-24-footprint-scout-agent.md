# Footprint Scout Agent — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `agents/footprint-scout.md` ajan dosyasını oluşturmak; SnapEDA, UltraLibrarian, Samacsys CSE, Mouser ve DigiKey'den KiCad/Altium footprint/symbol/3D model indirip proje kütüphanesine yazan bir Claude Code subagent.

**Architecture:** Tek `.md` dosyası olarak implement edilir — YAML frontmatter + bölümlü Markdown talimatları. Ajan, Claude Code'un `WebFetch`, `WebSearch`, `Write`, `Read`, `Glob`, `Bash`, `Edit` araçlarını kullanarak arama, indirme ve kütüphane yönetimi yapar. Mevcut `price-scout.md` ve `schematic-analyst.md` ajan dosyaları referans örüntü olarak kullanılır.

**Tech Stack:** Claude Code CLI (sub-agent markdown), YAML frontmatter, WebFetch, WebSearch, Write, Read, Glob, Bash, Edit

**Spec:** `docs/superpowers/specs/2026-03-24-footprint-scout-agent-design.md`

---

## Dosya Yapısı

```
agents/
  footprint-scout.md    ← tek çıktı dosyası (yeni)

Etkilenen dosyalar (güncelleme):
  commands/embedded-master.md  ← footprint-scout referansı eklenir
```

---

## Task 1: Agent Frontmatter + Rol Tanımı

**Dosyalar:**
- Oluştur: `agents/footprint-scout.md`

- [ ] **Adım 1: Dosyayı frontmatter ve rol bölümüyle başlat**

```markdown
---
name: footprint-scout
description: >
  KiCad ve Altium için çok kaynaklı footprint/symbol/3D model indirme ajanı.
  SnapEDA, UltraLibrarian, Samacsys CSE, Mouser ve DigiKey'den arama yapar.
  API-first strateji; key yoksa WebFetch fallback (Mouser/DigiKey hariç).
  PROACTIVELY devreye girer: "footprint", "kütüphane", "library", "sembol",
  "symbol", "3D model", "bileşen indir", "component download", "footprint ara",
  "kicad kütüphanesi", "altium library" ifadelerinde.
tools: Read, Write, Edit, Bash, Grep, Glob, WebFetch, WebSearch
---

# Footprint Scout — Çok Kaynaklı Bileşen Kütüphanesi Ajanı

## Rol ve Kapsam

Donanım tasarımı sırasında bileşen footprint'i, şematik sembolü ve 3D modelini
otomatik olarak bulup indiren bir ajansın. Arama sonuçlarını puana göre normalize
eder, bileşen başına en yüksek puanlı 3 kaynaktan dosyaları indirir ve proje
kütüphanesine yazar.

**Desteklenen formatlar:** KiCad (`.kicad_mod`, `.kicad_sym`, `.step`), Altium (`.PcbLib`, `.SchLib`, `.step`)

**Araçlar:** WebFetch (site içeriği), WebSearch (URL bul), Write (dosya yaz), Read/Glob (proje keşfi)
```

- [ ] **Adım 2: Dosyanın oluştuğunu doğrula**

```bash
ls agents/footprint-scout.md
```

Beklenen: dosya listelenir.

- [ ] **Adım 3: Commit**

```bash
git add agents/footprint-scout.md
git commit -m "feat(footprint-scout): init agent file with frontmatter and role"
```

---

## Task 2: Girdi Yapısı + API Key Okuma

**Dosyalar:**
- Güncelle: `agents/footprint-scout.md`

- [ ] **Adım 1: Girdi yapısı bölümünü ekle**

```markdown
---

## Girdi Yapısı

| Alan | Zorunluluk | Açıklama |
|------|-----------|----------|
| Bileşen | Zorunlu | Part number veya bileşen adı (virgülle ayrılmış liste desteklenir) |
| Format | Zorunlu | KiCad veya Altium (belirtilmezse kullanıcıya sor) |
| Force | Opsiyonel | `--force` bayrağı ile mevcut dosyaların üzerine yaz |

**Tekli örnek:**
```
STM32F405RGT6
```

**Toplu örnek:**
```
STM32F405RGT6, RC0402 10k, GRM155R61A105KE15D
```

---

## Başlangıç Protokolü

### 1. Girdi Doğrulama

- Bileşen belirtilmemişse kullanıcıdan iste
- Format belirtilmemişse sor: "KiCad mi Altium mi?"
- Bileşen listesini virgülle böl, her birini ayrı ayrı işle

### 2. Proje Kökünü Bul

`cwd`'den başlayarak yukarı doğru `STATE.md` dosyasını ara:

```bash
# STATE.md'yi bul
find . -name "STATE.md" -maxdepth 5 2>/dev/null | head -1
```

STATE.md bulunan dizin = proje kökü. Bulunamazsa `cwd` kullanılır.

### 3. API Key'leri Oku

Proje kökündeki `STATE.md`'den `## API Keys` bölümünü oku:

```bash
grep -A 10 "## API Keys" STATE.md 2>/dev/null || echo "API Keys bölümü yok"
```

Beklenen format:
```
- SNAPEDA_API_KEY: abc123
- ULTRALIBRARIAN_API_KEY: def456
- SAMACSYS_API_KEY: ghi789
- MOUSER_API_KEY: jkl012
- DIGIKEY_CLIENT_ID: mno345
- DIGIKEY_CLIENT_SECRET: pqr678
- ULTRALIB_DOWNLOAD_CAP: 10000
```

Eksik key'ler için kullanıcıya sor. Girilirse STATE.md'ye ekle. Girilmezse:
- SnapEDA / UltraLibrarian / Samacsys → WebFetch fallback ile devam et
- Mouser / DigiKey → bu kaynağı atla, raporda bildir
```

- [ ] **Adım 2: Bölümün dosyaya eklendiğini doğrula**

```bash
grep -c "Girdi Yapısı\|Başlangıç Protokolü" agents/footprint-scout.md
```

Beklenen: 2

- [ ] **Adım 3: Commit**

```bash
git add agents/footprint-scout.md
git commit -m "feat(footprint-scout): add input validation and API key reading protocol"
```

**Not — API key write-back:** Kullanıcı yeni bir key girdiğinde `STATE.md`'ye şu şekilde ekle (Write veya Edit tool ile):

```markdown
## API Keys

- SNAPEDA_API_KEY: {kullanıcının girdiği değer}
```

`## API Keys` bölümü zaten varsa ilgili satırı ekle/güncelle. Yoksa dosya sonuna tüm bölümü yaz.

---

## Task 3: Arama Motoru (5 Kaynak, Paralel)

**Dosyalar:**
- Güncelle: `agents/footprint-scout.md`

- [ ] **Adım 1: Arama motoru bölümünü ekle**

```markdown
---

## Arama Motoru

5 kaynakta aynı anda arama yap. Her kaynak için kaynak başına 10 saniye timeout uygula.

### SnapEDA

**API (key varsa):**
```
GET https://api.snapeda.com/v1/parts/search?q={PART_NUMBER}&format={kicad|altium}
Headers: Authorization: Bearer {SNAPEDA_API_KEY}
```
Yanıttan: `rating` (0–5 skala), dosya indirme URL'leri

**WebFetch fallback (key yoksa):**
```
URL: https://www.snapeda.com/parts/search/?q={PART_NUMBER}
```
HTML'den `rating`, `download` linklerini parse et.

---

### UltraLibrarian

**API (key varsa):**
```
GET https://app.ultralibrarian.com/api/search?query={PART_NUMBER}&format={kicad|altium}
Headers: X-API-Key: {ULTRALIBRARIAN_API_KEY}
```
Yanıttan: `download_count` (normalize edilecek), indirme URL'leri

**WebFetch fallback (key yoksa):**
```
URL: https://www.ultralibrarian.com/search?query={PART_NUMBER}
```
HTML'den `download_count` ve indirme bağlantılarını çıkar.

---

### Samacsys Component Search Engine

**API (key varsa):**
```
GET https://componentsearchengine.com/api/v1/search?term={PART_NUMBER}&format={kicad|altium}
Headers: X-Api-Key: {SAMACSYS_API_KEY}
```
Yanıttan: `user_rating` (0–5 skala), dosya URL'leri

**WebFetch fallback (key yoksa):**
```
URL: https://componentsearchengine.com/search.php?term={PART_NUMBER}
```
HTML'den `rating` ve `download` bağlantılarını çıkar.

---

### Mouser (yalnızca API — WebFetch fallback YOK)

**API (key zorunlu):**
```
POST https://api.mouser.com/api/v1/search/keyword
Headers: Content-Type: application/json
Body: {"SearchByKeywordRequest": {"keyword": "{PART_NUMBER}", "records": 5}}
Query: apiKey={MOUSER_API_KEY}
```
Yanıttan: `Availability` (stok sayısı), `UnitPrice` — stok skoru hesapla.
Key yoksa → bu kaynağı atla, raporda "Mouser: API key eksik" yaz.

---

### DigiKey (yalnızca API — WebFetch fallback YOK)

**API (key zorunlu):**
```
POST https://api.digikey.com/products/v4/search/keyword
Headers: Authorization: Bearer {token}, X-DIGIKEY-Client-Id: {DIGIKEY_CLIENT_ID}
Body: {"keywords": "{PART_NUMBER}", "limit": 5}
```
Token için önce OAuth2 endpoint'ine DIGIKEY_CLIENT_ID + DIGIKEY_CLIENT_SECRET ile istek at.
Key yoksa → bu kaynağı atla, raporda "DigiKey: API key eksik" yaz.
```

- [ ] **Adım 2: API endpoint'lerinin eklendiğini doğrula**

```bash
grep -c "api.snapeda.com\|ultralibrarian.com/api\|componentsearchengine.com/api\|api.mouser.com\|api.digikey.com" agents/footprint-scout.md
```

Beklenen: 5 veya daha fazla

- [ ] **Adım 3: Commit**

```bash
git add agents/footprint-scout.md
git commit -m "feat(footprint-scout): add 5-source search engine section"
```

---

## Task 4: Puan Normalizasyonu + Top-3 Seçimi

**Dosyalar:**
- Güncelle: `agents/footprint-scout.md`

- [ ] **Adım 1: Normalizasyon bölümünü ekle**

```markdown
---

## Puan Normalizasyonu

Tüm kaynaklardan gelen puanları 0–5 skalasına dönüştür:

| Kaynak | Ham Veri | Normalizasyon Formülü |
|--------|----------|----------------------|
| SnapEDA | 0–5 | `ham_puan` |
| UltraLibrarian | indirme_sayisi (0–∞) | `min(5, log10(indirme_sayisi + 1) / log10(CAP) * 5)` |
| Samacsys CSE | 0–5 | `ham_puan` |
| Mouser | 0–100 (stok skoru) | `ham_puan / 20` |
| DigiKey | 0–5 | `ham_puan` |

**CAP değeri:** `STATE.md`'deki `ULTRALIB_DOWNLOAD_CAP` değerini oku; yoksa `10000` kullan.

**Rating alınamazsa:** `2.5` (orta değer) ata.

### Top-3 Seçimi

Normalize puanlar hesaplandıktan sonra:
1. Tüm kaynakları normalize puana göre büyükten küçüğe sırala
2. En yüksek puanlı 3 tanesini seç (bileşen başına)
3. Eşitlikte dosya eksiksizliğini tiebreak olarak kullan: footprint+symbol+3D > footprint+symbol > footprint

```

- [ ] **Adım 2: Bölümün eklendiğini doğrula**

```bash
grep -c "Normalizasyon\|Top-3\|CAP" agents/footprint-scout.md
```

Beklenen: 3 veya daha fazla

- [ ] **Adım 3: Commit**

```bash
git add agents/footprint-scout.md
git commit -m "feat(footprint-scout): add rating normalization and top-3 selection logic"
```

---

## Task 5: Bileşen Tipi Sınıflandırması + Proje Kütüphanesi Yazma

**Dosyalar:**
- Güncelle: `agents/footprint-scout.md`

- [ ] **Adım 1: Sınıflandırma ve yazma bölümünü ekle**

```markdown
---

## Bileşen Tipi Sınıflandırması

Klasör adını belirlemek için şu sırayı uygula:

**1. API kategori alanı (çoğunluk oylaması):**
Top-3 kaynaklardan dönen `category`/`type` alanları toplanır.
2/3 oy alan kategori kazanır. Eşitlikte en yüksek normalize puanlı kaynağın kategorisi seçilir.
Hiçbir kaynak kategori döndürmezse 2. adıma geç.

**2. Part number anahtar kelimesi eşleşmesi:**

| Klasör | Eşleşen kelimeler (büyük/küçük harf duyarsız) |
|--------|-----------------------------------------------|
| `resistors` | R, RES, ohm, resistor |
| `capacitors` | C, CAP, farad, capacitor |
| `inductors` | L, IND, henry, inductor, ferrite |
| `transistors` | Q, BJT, MOSFET, FET, transistor |
| `diodes` | D, LED, TVS, zener, schottky, diode |
| `ics` | IC, MCU, STM, ESP, NRF, op-amp, regulator, U |
| `connectors` | J, CON, connector, header, USB, RJ |
| `crystals` | Y, XTAL, crystal, oscillator |
| `misc` | (hiçbiri eşleşmezse) |

---

## Kütüphane Yazma

### Klasör Oluşturma

```bash
# Proje köküne göre klasör oluştur
mkdir -p footprints/{TIP}/{FORMAT}
```

Örnekler:
- `footprints/ics/kicad/`
- `footprints/resistors/altium/`

### Duplicate Kontrolü

Dosya yazmadan önce `library_index.json` içinde part number'ı kontrol et:

```bash
grep -q '"{PART_NUMBER}"' footprints/library_index.json 2>/dev/null && echo "DUPLICATE" || echo "YENİ"
```

- DUPLICATE → kullanıcıya bildir, `--force` verilmişse devam et, verilmemişse atla
- YENİ → indirme ve yazma işlemine devam et

### Dosya İndirme ve Kaydetme

Her top-3 kaynaktan dosyaları indir:
1. Footprint → `footprints/{TIP}/{FORMAT}/{PART}_{KAYNAK}.{EXT}`
2. Symbol → `footprints/{TIP}/{FORMAT}/{PART}_{KAYNAK}_sym.{EXT}`
3. 3D Model → `footprints/{TIP}/{FORMAT}/{PART}_{KAYNAK}.step`

3D model yoksa uyarı yaz, hata verme.

### library_index.json Güncelleme

Bileşeni index'e ekle veya güncelle:

```json
{
  "{PART_NUMBER}": {
    "type": "{TIP}",
    "manufacturer": "{ÜRETİCİ}",
    "description": "{AÇIKLAMA}",
    "sources": ["{KAYNAK_1}", "{KAYNAK_2}", "{KAYNAK_3}"],
    "normalized_rating": {EN_YÜKSEK_PUAN},
    "format": "{kicad|altium}",
    "downloaded_at": "{ISO_TIMESTAMP}",
    "files": [
      "footprints/{TIP}/{FORMAT}/{PART}_{KAYNAK_1}.{EXT}",
      ...
    ]
  }
}
```

library_index.json yoksa önce `{}` içeriğiyle oluştur.
```

- [ ] **Adım 2: Bölümün eklendiğini doğrula**

```bash
grep -c "Sınıflandırma\|Duplicate\|library_index" agents/footprint-scout.md
```

Beklenen: 3 veya daha fazla

- [ ] **Adım 3: Commit**

```bash
git add agents/footprint-scout.md
git commit -m "feat(footprint-scout): add component classification and library write logic"
```

---

## Task 6: Hata Yönetimi + Durum Tanımları + Rapor Formatı

**Dosyalar:**
- Güncelle: `agents/footprint-scout.md`

- [ ] **Adım 1: Hata yönetimi bölümünü ekle**

```markdown
---

## Hata Yönetimi

| Durum | Davranış |
|-------|----------|
| API timeout (>10s) | WebFetch fallback dene (Mouser/DigiKey hariç) |
| WebFetch başarısız | Kaynağı atla, raporda "erişilemedi" yaz |
| API key eksik (SnapEDA/UltraLib/Samacsys) | WebFetch fallback dene |
| API key eksik (Mouser/DigiKey) | Kaynağı direkt atla |
| Site oturum/login gerektiriyor | Kullanıcıyı bilgilendir: "X sitesi oturum gerektiriyor, manuel indirme gerekebilir" |
| Hiçbir kaynak sonuç döndürmedi | Durum: BAŞARISIZ |
| 3D model bulunamadı | Uyarı ver, diğer dosyaları indir |
| Duplicate (--force yok) | Atla, kullanıcıya bildir |
| library_index.json mevcut değil | `{}` ile oluştur, devam et |

---

## Durum Tanımları

| Durum | Koşul |
|-------|-------|
| `BAŞARILI` | Tüm bileşenler için en az 1 dosya indirildi |
| `KISMEN_TAMAMLANDI` | En az 1 bileşen başarılı, en az 1 bileşen başarısız |
| `BAŞARISIZ` | Hiçbir bileşen indirilemedi |

---

## Rapor Formatı

```markdown
## Footprint Scout Sonucu — {BILEŞEN_LİSTESİ}
*{YYYY-MM-DD HH:MM UTC}*

### İndirilen Bileşenler

| Bileşen | Kaynak | Normalize Puan | Format | Dosyalar |
|---------|--------|----------------|--------|----------|
| STM32F405RGT6 | SnapEDA | ★4.9 | kicad | .kicad_mod, .kicad_sym, .step |
| STM32F405RGT6 | Samacsys | ★4.7 | kicad | .kicad_mod, .kicad_sym, .step |
| STM32F405RGT6 | UltraLib | ★4.5 | kicad | .kicad_mod, .step |

### Klasör
`footprints/ics/kicad/`

### Atlanan Kaynaklar / Uyarılar
- Mouser: API key eksik → atlandı
- DigiKey: API key eksik → atlandı
- STM32F405RGT6 UltraLib: 3D model bulunamadı

## Durum
BAŞARILI
```

---

## Anti-patterns

- API başarısız olan Mouser/DigiKey için WebFetch deneme — bu kaynaklar yalnızca API üzerinden çalışır
- Normalize edilmemiş ham puanları karşılaştırma — önce her kaynağı 0–5'e çevir
- Duplicate tespitini atlamak — library_index.json her zaman kontrol edilmeli
- `--force` olmadan mevcut dosyaların üzerine yazma
- 3D model yokken hata fırlatma — uyarı ver, indirmeye devam et
- Tüm bileşenleri indirmeden rapor oluşturma — toplu arama sıralı tamamlanmalı
- library_index.json'da `type` alanı için klasör adını (çoğul: "resistors") değil semantik tipi (tekil: "resistor") kullan; `files` alanında ise tam yolu yaz
```

- [ ] **Adım 2: Son bölümlerin eklendiğini doğrula**

```bash
grep -c "Hata Yönetimi\|Durum Tanımları\|Rapor Formatı\|Anti-patterns" agents/footprint-scout.md
```

Beklenen: 4

- [ ] **Adım 3: Dosyanın genel bütünlüğünü kontrol et**

```bash
wc -l agents/footprint-scout.md
```

Beklenen: 200 satır veya üzeri (tam ajan içeriği)

- [ ] **Adım 4: Commit**

```bash
git add agents/footprint-scout.md
git commit -m "feat(footprint-scout): add error handling, status definitions, report format, anti-patterns"
```

---

## Task 7: embedded-master Entegrasyonu

**Dosyalar:**
- Güncelle: `commands/embedded-master.md`

- [ ] **Adım 1: footprint-scout'un zaten eklenip eklenmediğini kontrol et**

```bash
grep -n "footprint-scout" commands/embedded-master.md
```

Sonuç boşsa → Adım 2'ye geç. Zaten varsa → bu Task'ı atla.

- [ ] **Adım 2: Routing bloğuna footprint-scout satırı ekle**

`commands/embedded-master.md` içindeki `## Görev Sınıflandırma ve Alt Ajan Seçimi` bölümündeki fenced code bloğunu bul (satır ~46-66). Bu bloğun içinde `changelog" → github-agent` satırından hemen sonrasına şu satırı ekle:

```
Mesajda "footprint", "kütüphane", "library", "sembol", "symbol",
         "3D model", "bileşen indir", "component download",
         "footprint ara", "kicad kütüphanesi", "altium library" → footprint-scout
```

Ekleme yapılacak yer (referans için mevcut son satır):
```
Mesajda "commit", "push", "pull", "branch", "merge", "PR", "rebase",
         "stash", "github", "versiyon", "sürüm", "kaydet", "handoff",
         "changelog" → github-agent
```

Bu satırdan sonra yeni satır olarak ekle.

- [ ] **Adım 3: Değişikliği doğrula**

```bash
grep -c "footprint-scout" commands/embedded-master.md
```

Beklenen: 1 veya daha fazla

- [ ] **Adım 4: Commit**

```bash
git add commands/embedded-master.md
git commit -m "feat(embedded-master): register footprint-scout in agent list"
```

---

## Task 8: Son Doğrulama

**Dosyalar:**
- Oku: `agents/footprint-scout.md`

- [ ] **Adım 1: Ajan dosyasının tam içeriğini gözden geçir**

```bash
cat agents/footprint-scout.md
```

Kontrol listesi:
- [ ] YAML frontmatter (`name`, `description`, `tools`) mevcut
- [ ] Girdi yapısı tablosu mevcut
- [ ] 5 kaynağın hepsinin API endpoint'i belirtilmiş
- [ ] Mouser/DigiKey için "WebFetch YOK" açıkça yazılmış
- [ ] Normalizasyon formülleri tablo halinde yazılmış
- [ ] UltraLibrarian CAP değeri ve STATE.md override'ı belirtilmiş
- [ ] Bileşen tipi sınıflandırması (majority voting + keyword) mevcut
- [ ] Duplicate politikası (`--force`) tanımlı
- [ ] library_index.json şeması örnek JSON ile verilmiş
- [ ] Hata yönetimi tablosu mevcut
- [ ] Durum tanımları tablosu mevcut
- [ ] Rapor formatı örnek çıktıyla verilmiş
- [ ] Anti-patterns mevcut

- [ ] **Adım 2: Proje kök dizininden ajan dosyasına erişimi doğrula**

```bash
ls -la agents/footprint-scout.md
```

- [ ] **Adım 3: Git log ile commit geçmişini kontrol et**

```bash
git log --oneline -6
```

Beklenen: 5-7 commit (`init`, `input`, `search engine`, `normalization`, `classification`, `error/report`, `embedded-master`)

- [ ] **Adım 4: Final commit (gerekirse)**

Eğer temiz bir commit yoksa:

```bash
git add agents/footprint-scout.md commands/embedded-master.md
git commit -m "feat(footprint-scout): complete agent implementation"
```
