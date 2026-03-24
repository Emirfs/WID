# Footprint Scout Agent — Tasarım Dökümanı

**Tarih:** 2026-03-24
**Durum:** Onaylandı
**Kapsam:** KiCad ve Altium için çok kaynaklı footprint/symbol/3D model indirme ajanı

---

## 1. Genel Bakış

`footprint-scout` ajanı, donanım tasarımı sırasında bileşen footprint'i, şematik sembolü ve 3D modelini otomatik olarak bulup indiren bir Claude Code subagent'tır. Mevcut `schematic-analyst` ve `hardware-reviewer` ajanlarıyla entegre çalışır.

**Agent frontmatter (agents/footprint-scout.md için):**
```yaml
name: footprint-scout
description: >
  KiCad ve Altium için çok kaynaklı footprint/symbol/3D model indirme ajanı.
  SnapEDA, UltraLibrarian, Samacsys CSE, Mouser ve DigiKey'den arama yapar.
  PROACTIVELY devreye girer: "footprint", "kütüphane", "library", "sembol",
  "symbol", "3D model", "bileşen indir", "component download" ifadelerinde.
tools: Read, Write, Edit, Bash, Grep, Glob, WebFetch
```

---

## 2. Gereksinimler

- KiCad ve Altium formatlarını destekler; format seçimi çalışma zamanında kullanıcıya sorulur
- 5 kaynakta paralel arama: SnapEDA, UltraLibrarian, Samacsys CSE, Mouser, DigiKey
- **Bileşen başına** en yüksek puanlı 3 sonucu otomatik indirir (10 bileşen → 30 indirme)
- Her bileşen için footprint + şematik sembol + 3D model (STEP) indirilir; 3D model yoksa uyarı verilir hata verilmez
- Toplu arama: virgülle ayrılmış birden fazla bileşen desteklenir; bileşenler sıralı işlenir
- Zaman aşımı: kaynak başına 10 saniye; başarısız olursa kaynak atlanır

---

## 3. Mimari

### 3.1 Bileşenler

```
footprint-scout agent
│
├── Giriş Aşaması
│   ├── Bileşen adı / part number al
│   └── Format sor → KiCad veya Altium
│
├── Arama Motoru (5 kaynak, paralel, kaynak başına 10s timeout)
│   ├── SnapEDA       → API önce, WebFetch fallback
│   ├── UltraLibrarian → API önce, WebFetch fallback
│   ├── Samacsys CSE  → API önce, WebFetch fallback
│   ├── Mouser        → Yalnızca API (key zorunlu, WebFetch fallback YOK)
│   └── DigiKey       → Yalnızca API (key zorunlu, WebFetch fallback YOK)
│
├── Sonuç Birleştirici
│   ├── Normalize et → tüm puanları 0-5 skalasına çevir
│   ├── Puana göre sırala
│   └── Bileşen başına en yüksek puanlı 3 seç
│
├── İndirici
│   ├── Footprint + Symbol + 3D model
│   └── Seçilen formata göre filtrele
│
└── Kütüphane Yöneticisi
    ├── Proje kökünü bul (STATE.md'ye yukarı doğru traversal)
    ├── Bileşen tipini sınıflandır
    ├── <proje>/footprints/<tip>/<format>/ klasörüne yaz
    ├── Duplicate kontrolü → varsa atla, --force ile üzerine yaz
    └── library_index.json güncelle
```

### 3.2 Kaynak Stratejisi

| Kaynak | API Endpoint | WebFetch Fallback | Rating Kaynağı | Ham Skala |
|--------|-------------|-------------------|----------------|-----------|
| SnapEDA | `api.snapeda.com/v1/parts` | snapeda.com/parts/search | SnapEDA puanı | 0–5 |
| UltraLibrarian | `app.ultralibrarian.com/api` | ultralibrarian.com/search | İndirme sayısı | 0–∞ |
| Samacsys CSE | `componentsearchengine.com/api` | componentsearchengine.com/search | Kullanıcı puanı | 0–5 |
| Mouser | `api.mouser.com/api/v1` | **YOK** — key olmadan kaynak atlanır | Stok skoru | 0–100 |
| DigiKey | `api.digikey.com/products/v4` | **YOK** — key olmadan kaynak atlanır | DigiKey puanı | 0–5 |

**Önemli:** Mouser ve DigiKey için WebFetch fallback uygulanmaz. API key eksikse bu kaynaklar doğrudan atlanır.

**Kaynak öncelik sırası (SnapEDA, UltraLibrarian, Samacsys için):**
1. API key tanımlıysa → REST API çağrısı (10s timeout)
2. Key yoksa veya API başarısız olursa → WebFetch ile HTML parse (10s timeout)
3. Her ikisi de başarısız → kaynağı atla, log yaz

**Kaynak öncelik sırası (Mouser, DigiKey için):**
1. API key tanımlıysa → REST API çağrısı (10s timeout)
2. Key yoksa veya API başarısız olursa → doğrudan atla, log yaz

### 3.3 Puan Normalizasyonu

Tüm kaynak puanları karşılaştırılabilir 0–5 skalasına dönüştürülür:

| Kaynak | Normalizasyon Formülü |
|--------|----------------------|
| SnapEDA | `ham_puan` (zaten 0–5) |
| UltraLibrarian | `min(5, log10(indirme_sayisi + 1) / log10(CAP) * 5)` |
| Samacsys CSE | `ham_puan` (zaten 0–5) |
| Mouser | `ham_puan / 20` (0–100 → 0–5) |
| DigiKey | `ham_puan` (zaten 0–5) |

**UltraLibrarian CAP değeri:** Varsayılan `10000`. Kütüphane sitelerinde yaygın bileşenlerin tipik indirme aralığı (100–10.000) bu sınırı kapsayacak şekilde seçilmiştir; üzerindeki her değer `5.0` alır. `STATE.md`'de `ULTRALIB_DOWNLOAD_CAP: <değer>` satırıyla geçersiz kılınabilir.

Rating bilgisi API'den gelmiyorsa o kaynak için normalize puan `2.5` (orta değer) atanır.

---

## 4. Proje Kökü Tespiti

Ajan çalışma dizininden (`cwd`) başlayarak yukarı doğru `STATE.md` dosyasını arar:

```
cwd/ → STATE.md var mı? → evet → proje kökü burada
     → hayır → üst dizine git → tekrar dene
     → kök dizine ulaşıldı ve STATE.md yoksa → cwd'yi proje kökü say
```

`footprints/` klasörü proje köküne göre oluşturulur.

---

## 5. Bileşen Tipi Sınıflandırması

Bileşen tipi şu sırayla belirlenir:

1. **API kategori alanı:** Top-3 kaynaklardan dönen `category`/`type` alanları çoğunluk oylamasıyla belirlenir (2/3 oy alan kategori kazanır). Eşitlik halinde en yüksek normalize puanlı kaynağın kategorisi seçilir. Hiçbir kaynak kategori döndürmüyorsa 2. adıma geç.
2. **Part number anahtar kelimesi eşleşmesi:**

| Klasör | Eşleşen kelimeler |
|--------|------------------|
| `resistors` | R, RES, ohm, resistor |
| `capacitors` | C, CAP, farad, capacitor |
| `inductors` | L, IND, henry, inductor, ferrite |
| `transistors` | Q, BJT, MOSFET, FET, transistor |
| `diodes` | D, LED, TVS, zener, schottky, diode |
| `ics` | IC, MCU, STM, ESP, NRF, op-amp, regulator, U |
| `connectors` | J, CON, connector, header, USB, RJ |
| `crystals` | Y, XTAL, crystal, oscillator |
| `misc` | hiçbiri eşleşmezse |

---

## 6. API Key Yönetimi

API key'ler proje `STATE.md` dosyasında aşağıdaki formatta saklanır:

```markdown
## API Keys

- SNAPEDA_API_KEY: <değer>
- ULTRALIBRARIAN_API_KEY: <değer>
- SAMACSYS_API_KEY: <değer>
- MOUSER_API_KEY: <değer>
- DIGIKEY_CLIENT_ID: <değer>
- DIGIKEY_CLIENT_SECRET: <değer>
```

İlk çalıştırmada ajan `STATE.md`'deki eksik key'leri tespit eder ve kullanıcıya sorar:
- Key girilirse → `STATE.md`'ye kaydeder
- Key girilmezse → o kaynağı atlar (Mouser/DigiKey için WebFetch yok; diğerleri için WebFetch dener)

---

## 7. Kütüphane Yapısı

```
<proje_kök>/
└── footprints/
    ├── resistors/
    │   ├── kicad/      ← .kicad_mod, .kicad_sym, .step
    │   └── altium/     ← .PcbLib, .SchLib, .step
    ├── capacitors/
    ├── ics/
    ├── connectors/
    ├── inductors/
    ├── transistors/
    ├── diodes/
    ├── crystals/
    ├── misc/
    └── library_index.json
```

### 7.1 library_index.json Formatı

```json
{
  "RC0402FR-0710KL": {
    "type": "resistor",
    "manufacturer": "Yageo",
    "description": "10kΩ 1% 0402",
    "sources": ["snapeda", "ultralibrarian"],
    "normalized_rating": 4.8,
    "format": "kicad",
    "downloaded_at": "2026-03-24T10:00:00Z",
    "files": [
      "footprints/resistors/kicad/RC0402FR-0710KL_snapeda.kicad_mod",
      "footprints/resistors/kicad/RC0402FR-0710KL_snapeda.kicad_sym",
      "footprints/resistors/kicad/RC0402FR-0710KL_snapeda.step"
    ]
  }
}
```

### 7.2 Duplicate Politikası

- Part number `library_index.json`'da zaten varsa → indirme atlanır, kullanıcıya bildirilir
- `--force` bayrağı verilirse → mevcut dosyalar üzerine yazılır, index güncellenir

---

## 8. Ajan Davranış Akışı

```
[1] Bileşen al          → "STM32F405RGT6" (tekli veya virgülle ayrılmış liste)
[2] Format sor          → "KiCad mi Altium mi?"
[3] Proje kökünü bul    → STATE.md traversal
[4] API key'leri oku    → STATE.md [API Keys] bölümü
[5] Her bileşen için:
    [5a] Paralel ara    → 5 kaynak eş zamanlı (10s timeout)
    [5b] Normalize et   → tüm puanları 0-5'e çevir
    [5c] Top 3 seç      → en yüksek normalize puanlı 3 kaynak
    [5d] Duplicate?     → varsa atla; --force ile devam
    [5e] İndir          → footprint + symbol + 3D model
    [5f] Sınıflandır    → bileşen tipini belirle
    [5g] Kütüphaneye yaz → footprints/<tip>/<format>/
    [5h] Index güncelle  → library_index.json
[6] Özet rapor         → tüm bileşenler için sonuçları listele
```

**Toplu arama örneği:**
```
"STM32F405RGT6, RC0402 10k, GRM155R61A105KE15D"
→ 3 bileşen → her biri 5 kaynakta aranır → her biri için top 3 → 9 bileşen indirme
```

---

## 9. Durum Tanımları

| Durum | Koşul |
|-------|-------|
| `BAŞARILI` | Tüm bileşenler için en az 1 dosya indirildi |
| `KISMEN_TAMAMLANDI` | En az 1 bileşen indirildi, en az 1 bileşen başarısız |
| `BAŞARISIZ` | Hiçbir bileşen indirilemedi |

---

## 10. Çıktı Raporu Formatı

```markdown
## Footprint Scout Sonucu

### İndirilen Bileşenler
| Bileşen | Kaynak | Normalize Puan | Dosyalar |
|---------|--------|----------------|----------|
| STM32F405RGT6 | SnapEDA | ★4.9 | .kicad_mod, .kicad_sym, .step |
| STM32F405RGT6 | Samacsys | ★4.7 | .kicad_mod, .kicad_sym, .step |
| STM32F405RGT6 | UltraLib | ★4.5 | .kicad_mod, .step (3D model yok) |

### Atlanan / Uyarılar
- Mouser: API key eksik → atlandı
- DigiKey: API key eksik → atlandı
- STM32F405RGT6 UltraLib: 3D model bulunamadı

## Durum
BAŞARILI
```

---

## 11. Entegrasyon Kontratı

`library_index.json`'dan okuma yapan ajanlar şu alanları kullanabilir:

| Alan | Tip | Açıklama |
|------|-----|----------|
| `type` | string | Bileşen tipi (resistor, ic, connector...) |
| `files` | string[] | Proje köküne göreli dosya yolları |
| `format` | string | `kicad` veya `altium` |
| `manufacturer` | string | Üretici adı |
| `description` | string | İnsan okunabilir bileşen açıklaması |

- `schematic-analyst`: BOM doğrulamasında `files` listesine bakarak footprint'in mevcut olup olmadığını kontrol eder
- `hardware-reviewer`: `type` ve `description` ile BOM içindeki bileşeni doğrular

---

## 12. Sınırlamalar

- Mouser ve DigiKey yalnızca API üzerinden çalışır; key olmadan bu kaynaklar atlanır
- Bazı siteler indirme için oturum gerektirebilir; bu durumda ajan kullanıcıyı bilgilendirir
- 3D model her kaynakta bulunmayabilir; eksikse uyarı verilir, hata verilmez
- UltraLibrarian indirme sayısı 0 olan bileşenler `normalized_rating: 0.0` alır
