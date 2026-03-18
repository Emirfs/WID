# Embedded Systems Master Agent — Design Spec
**Date:** 2026-03-18
**Platform:** Claude Code CLI (terminal)
**Scope:** STM32 odaklı, gömülü sistemler için master + alt ajan mimarisi

---

## 1. Genel Bakış

Gömülü sistem geliştirme süreçlerini destekleyen, Claude Code ekosistemi içinde tamamen çalışan, Pro/Max aboneliği ile kullanılabilen (ek API maliyeti yok) bir master ajan sistemi. Master ajan orkestratör rolündedir; görevi analiz eder, uygun alt ajana yönlendirir, kullanıcıya her adımı açıklar ve proje hafızasını yönetir.

---

## 2. Mimari

### Katmanlar

```
Kullanıcı (Terminal)
       │
       ▼
Master Agent  ← ~/.claude/commands/embedded-master.md (skill)
  • Görev sınıflandırma
  • Alt ajan seçimi + açıklama
  • Hafıza okuma/yazma
  • Başarısızlık takibi → kullanıcı danışma
       │  (Claude Code Agent tool ile çağrı)
       ▼
Alt Ajanlar (~/.claude/agents/*.md)
  firmware-coder | hardware-reviewer | debugger
  security-analyst | doc-generator | test-engineer
  rtos-specialist | linker-expert | power-management | hardware-bringup
       │
       ▼
Hafıza Katmanı (proje bazlı .md dosyaları)
  PROJECT.md | STATE.md | REQUIREMENTS.md | ROADMAP.md
  PRPs/ | project_memory/
```

### Üç Katmanlı Bağlam Mimarisi (Context-Rot Önleme)

| Katman | Dosyalar | Amaç |
|--------|----------|-------|
| Global | `~/.claude/CLAUDE.md` | Claude Code global davranış kuralları |
| Proje | `<proje>/CLAUDE.md`, `PROJECT.md`, `STATE.md`, `REQUIREMENTS.md`, `ROADMAP.md` | STM32 kodlama standartları + donanım platformu + geçmiş kararlar |
| Görev | `PRPs/<görev>.md` | Tek peripheral/feature için blueprint + validation gates |

Her görev tamamlandığında sonuç STATE.md'ye sıkıştırılır — ham konuşma geçmişi birikmez.

---

## 3. Proje Yapısı

### Global (~/.claude/)

```
~/.claude/
  CLAUDE.md                  # Global Claude Code kuralları (genel)
  agents/
    firmware-coder.md
    hardware-reviewer.md
    debugger.md
    security-analyst.md
    doc-generator.md
    test-engineer.md
    rtos-specialist.md
    linker-expert.md
    power-management.md
    hardware-bringup.md
  commands/
    embedded-master.md       # /embedded-master — master ajan skill giriş noktası
    generate-prp.md          # /generate-prp komutu
    execute-prp.md           # /execute-prp komutu
    new-project.md           # /new-project komutu
    switch-project.md        # /switch-project komutu
    verify-hardware.md       # /verify-hardware — ertelenmiş Gate 5 tamamlama
```

### Proje Klasörü

```
<proje_klasörü>/
  CLAUDE.md                  # STM32 kodlama standartları (proje bazlı, /new-project ile oluşturulur)
  PROJECT.md                 # Donanım platformu, MCU, hedefler
  STATE.md                   # Kararlar, blokajlar, başarısızlık sayacı, mevcut durum
  REQUIREMENTS.md            # Firmware gereksinimleri
  ROADMAP.md                 # Faz takibi + tamamlanma kriterleri
  PRPs/
    <görev-adı>.md           # Görev bazlı blueprint (schema aşağıda)
  project_memory/
    hardware.md              # MCU modeli, pinout, saat ağacı, donanım kısıtları
    decisions.md             # Mimari kararlar + gerekçeleri
    bugs.md                  # Karşılaşılan buglar + çözümler
    progress.md              # Tamamlananlar / yapılacaklar
```

---

## 4. Master Agent — Tanım ve Aktivasyon

### Aktivasyon

Master ajan bir **Claude Code custom command (skill)** olarak tanımlanır:
- Dosya: `~/.claude/commands/embedded-master.md`
- Kullanıcı `/embedded-master` veya `/em` ile başlatır

**Not:** Proje klasöründeki `CLAUDE.md` yalnızca STM32 kodlama standartlarını içerir; master ajanı otomatik başlatmaz. Oturum başlangıcında master ajanın devreye girmesi için kullanıcının `/embedded-master` komutunu çalıştırması gerekir. `CLAUDE.md` içindeki "Oturum Başlangıcı" bölümü, master ajan aktif olduğunda izlemesi gereken prosedürü tanımlar — aktivasyon mekanizması değil.

### embedded-master.md Yapısı

```yaml
---
name: embedded-master
description: >
  STM32 ve gömülü sistem geliştirme master ajanı. Tüm firmware,
  donanım, debug, güvenlik ve RTOS görevlerini koordine eder.
  Her oturumda project_memory/ ve STATE.md'yi okur.
tools: Read, Write, Edit, Bash, Grep, Glob, Agent, Task
---

[Master ajan system prompt:
 - Oturum başlangıç prosedürü
 - Görev sınıflandırma mantığı
 - Alt ajan seçim kuralları
 - Başarısızlık yönetimi
 - Hafıza güncelleme kuralları]
```

### Oturum Başlangıç Prosedürü (Canonical)

```
1. <proje>/CLAUDE.md oku (yoksa ~/.claude/CLAUDE.md)
2. STATE.md oku (aktif proje, başarısızlık sayacı, blokajlar)
3. project_memory/*.md oku
4. Kullanıcıya bildir:
   "Aktif proje: <proje_adı> | MCU: <mcu> | Son durum: <STATE.md özeti>"
```

---

## 5. Alt Ajanlar

Her alt ajan `~/.claude/agents/<isim>.md` formatında tanımlanır:

```yaml
---
name: <isim>
description: >
  [Uzmanlık alanı. PROACTIVELY tetikleyici koşullar.]
tools: Read, Write, Edit, Bash, Grep, Glob
---
[Uzman system prompt]
```

### Alt Ajan Listesi

| Ajan | Uzmanlık | PROACTIVE Tetikleyici |
|------|----------|-----------------------|
| `firmware-coder` | STM32 HAL/LL/CMSIS kod üretimi, CubeMX config | `.c`/`.h` dosyalarında HAL API kullanımı |
| `hardware-reviewer` | Şematik analiz, PCB kural kontrolü, devre hesabı | Şematik/PCB dosyası referansı |
| `debugger` | HardFault, peripheral yanıt vermeme, DMA hataları, stack overflow | Hata mesajları, "çalışmıyor" ifadeleri |
| `security-analyst` | Firmware şifreleme, bootloader güvenliği, saldırı yüzeyi | Güvenlik/bootloader kelimeleri |
| `doc-generator` | Doxygen, datasheet özeti, teknik dokümantasyon | "belgele", "dokümante et" komutları |
| `test-engineer` | Unit test, HIL senaryoları, boundary analizi | "test yaz", "doğrula" komutları |
| `rtos-specialist` | FreeRTOS görev tasarımı, priority inversion, deadlock | `FreeRTOS.h` veya RTOS primitifleri |
| `linker-expert` | `.ld` dosyası, flash/RAM overflow, memory map | `.ld` dosya düzenleme |
| `power-management` | Low-power mode, sleep, clock gating | Güç tüketimi, uyku modu konuları |
| `hardware-bringup` | Yeni kart bring-up, BSP geliştirme | Yeni kart/proje başlangıcı |

**Yeni ajan eklemek:** `~/.claude/agents/<isim>.md` oluşturmak yeterli.

### Alt Ajan Çağrı Kontratı

Master ajan bir alt ajanı Claude Code `Agent` tool ile çağırır. Her çağrıda aşağıdaki payload iletilir:

```
CONTEXT PAYLOAD (her alt ajan çağrısında):
  - STATE.md içeriği (proje durumu)
  - PRPs/<görev>.md içeriği (aktif görev)
  - İlgili HAL header excerpt (varsa, kullanıcıdan onay alınarak)
  - Kullanıcı mesajı

BEKLENEN ÇIKTI FORMATI (Markdown):
  ## Sonuç
  [Ne yapıldı / ne bulundu]

  ## Durum
  BAŞARILI | BAŞARISIZ | KISMEN_TAMAMLANDI

  ## STATE.md Güncellemesi (varsa)
  [STATE.md'ye eklenecek bilgi]

  ## Kullanıcı İçin Notlar
  [Kullanıcıya iletilecek özet]
```

---

## 6. Master Agent Karar Mekanizması

```
Kullanıcı mesajı gelir
        │
        ▼
1. Hafıza yükle
   project_memory/*.md + STATE.md oku (başarısızlık sayacını dahil et)
        │
        ▼
2. Görevi sınıflandır
   "Bu bir debug görevi mi? Kod yazımı mı? Şematik mi?"
        │
        ▼
3. Alt ajan seç + kullanıcıya açıkla
   "Bu bir DMA transfer hatası. debugger ajanını devreye alıyorum."
   [PRP varsa: ajan seçimi PRP'deki 'assigned_agent' alanından]
   [PRP yoksa: görev sınıflandırmasından]
        │
        ▼
4. Taze bağlam hazırla + kullanıcı onayı (gerekirse)
   STATE.md + PRP + ilgili dosyalar → context payload hazırla
   Dosya erişimi gerekiyorsa kullanıcıya sor
        │
        ▼
5. Alt ajanı çağır (Agent tool)
        │
        ▼
6. Çıktıyı değerlendir
   BAŞARILI   → kullanıcıya sun + STATE.md güncelle + sayacı sıfırla
   BAŞARISIZ  → hata kategorisine göre işle (aşağıya bak)
```

### Hata Kategorileri ve Yanıtları

| Kategori | Tanım | Yanıt |
|----------|-------|-------|
| `COMPILE_ERROR` | Gate 1/4 başarısız | firmware-coder ile tekrar dene, farklı yaklaşım |
| `LOGIC_ERROR` | Kod derlenir ama yanlış davranır | debugger devreye al |
| `HARDWARE_UNAVAILABLE` | Gate 5 atlandı | Kısmi doğrulama, STATE.md'ye kaydet, kullanıcıya bildir |
| `AGENT_FAILURE` | Alt ajan geçersiz çıktı | Sayacı artır, farklı prompt ile tekrar dene |
| `UNKNOWN` | Sınıflandırılamayan hata | Sayacı artır |

**Başarısızlık sayacı STATE.md'de `failure_count` alanında saklanır.**

```
sayaç ≥ 2?
├─ Evet → "Bu yaklaşım 2 kez işe yaramadı.
│          Farklı bir yol deneyelim mi?" (kullanıcıya danış, sayacı sıfırla)
└─ Hayır → farklı stratejiyle tekrar dene
```

### Kullanıcı Onayı Gereken Kritik Noktalar
- Dosya okuma/yazma öncesi
- Birden fazla alt ajan zinciri gerektiğinde
- Hafızaya yeni bilgi eklenirken
- Yaklaşım değişikliği gerektiğinde

### Rutin — Otomatik (Açıklamayla)
- Görev sınıflandırma
- Alt ajan seçimi
- Validation gates çalıştırma

---

## 7. PRP Dosya Şeması

`PRPs/<görev-adı>.md` formatı:

```markdown
# PRP: <görev adı>
**Oluşturulma:** YYYY-MM-DD
**Assigned Agent:** <firmware-coder | debugger | ...>
**Validation Budget:** flash=<KB>KB, ram=<KB>KB

## Hedef
[Ne yapılacak, başarı kriterleri]

## Bağlam
- İlgili HAL API referansları
- Mevcut driver pattern'ları
- Donanım kısıtları (saat frekansı, pin, DMA kanal)
- Bilinen errata

## Uygulama Planı (Wave Yapısı)
### Wave 1 (paralel)
- [ ] Görev 1a: ...
- [ ] Görev 1b: ...

### Wave 2 (paralel, Wave 1'e bağlı)
- [ ] Görev 2a: ...

### Wave N (serial)
- [ ] Görev Na: ...

## Validation Gates
- [ ] Gate 1 — Syntax: `arm-none-eabi-gcc -Wall -Werror`
- [ ] Gate 2 — Size: `arm-none-eabi-size` (flash < budget, RAM < budget)
- [ ] Gate 3 — Static: `cppcheck src/`
- [ ] Gate 4 — Build: `make clean && make`
- [ ] Gate 5 — Hardware: `openocd -f board.cfg -c "program firmware.elf verify reset"` [OPTIONAL]
- [ ] Gate 6 — RTOS: Stack watermark kontrol [RTOS varsa]

## Anti-patterns
[Ne yapılmamalı]
```

---

## 8. İş Akışı (PRP Pipeline)

### /new-project
```
→ Kullanıcıya sorular sor: MCU modeli, proje hedefi, donanım özellikleri
→ Şunları oluştur:
   PROJECT.md       (MCU, clock, hedef)
   STATE.md         (başlangıç: failure_count=0, active_prp=null)
   REQUIREMENTS.md  (kullanıcının verdiği gereksinimler)
   ROADMAP.md       (faz listesi + her faz için tamamlanma kriteri)
   CLAUDE.md        (STM32 kodlama standartları — ~/.claude/templates/claude-stm32.md kopyası)
   project_memory/hardware.md  (MCU detayları)
```

### /switch-project \<proje_yolu\>
```
→ Mevcut STATE.md kaydet (unsaved work uyarısı ver)
→ Hedef proje klasörünü doğrula (CLAUDE.md + PROJECT.md varlığı)
→ Hedef projenin STATE.md + project_memory/*.md oku
→ Kullanıcıya bildir: "Proje değiştirildi: <yeni_proje> | MCU: <mcu>"
```

### /generate-prp "\<görev açıklaması\>"
```
→ HAL dokümantasyonu araştır (kullanıcıdan onay alarak)
→ Mevcut driver pattern'larını tara
→ assigned_agent belirle (görev sınıflandırmasına göre)
→ PRPs/<görev-adı>.md oluştur (yukarıdaki şemaya göre)
→ Kullanıcıya sun, onay iste
```

### /execute-prp \<prp_yolu\>
```
→ PRP'yi oku, assigned_agent'ı belirle
→ "firmware-coder ajanını devreye alıyorum — Wave 1 başlıyor" (açıkla)
→ Wave bazlı uygulama (aşağıya bak)
→ Tüm validation gates çalıştır
→ STATE.md güncelle (failure_count sıfırla, active_prp=null)
→ ROADMAP.md ilgili fazı işaretle
→ project_memory/progress.md güncelle (tamamlanan PRP'yi ekle)
```

### Wave Execution Modeli

Wave'ler Claude Code `Task` tool ile paralel çalıştırılır. Her Task tamamlandığında master ajan çıktıyı okur; Wave N+1 ancak Wave N'deki tüm Task'lar `BAŞARILI` döndükten sonra başlar.

```
Wave 1 (Task paralel): Clock init, GPIO init, NVIC config
Wave 2 (Task paralel): UART, SPI, I2C driver'ları          [Wave 1 tamamlanınca]
Wave 3 (Task paralel): Sensör, aktüatör driver'ları         [Wave 2 tamamlanınca]
Wave 4 (Task serial):  RTOS görev oluşturma + scheduler     [Wave 3 tamamlanınca]
Wave 5 (Task paralel): Uygulama modülleri                   [Wave 4 tamamlanınca]
```

---

## 9. Hafıza Sistemi

### Dosya Sorumlulukları (Çakışmasız)

| Dosya | İçerik | Kim Yazar |
|-------|--------|-----------|
| `STATE.md` | Aktif PRP, `failure_count`, son oturum özeti, blokajlar | Master ajan (her görev sonunda) |
| `project_memory/hardware.md` | MCU, pinout, clock tree, donanım kısıtları | `/new-project` + hardware-reviewer |
| `project_memory/decisions.md` | Mimari kararlar + gerekçe ("Neden LL yerine HAL?") | Master ajan (kullanıcı onayı sonrası) |
| `project_memory/bugs.md` | Bug + çözüm çiftleri (tarihli) | Master ajan (debugger çıktısından) |
| `project_memory/progress.md` | Tamamlananlar, yapılacaklar | Master ajan — `/execute-prp` tamamlandığında (tüm gates geçince) ve `/new-project` sırasında |
| `REQUIREMENTS.md` | Değişmez gereksinimler | `/new-project`, kullanıcı talepleri |
| `ROADMAP.md` | Faz listesi + tamamlanma kriterleri | `/new-project` + faz güncellemeleri |

**Kural:** Bir görev hem bug fix hem de yeni karar üretirse → bug `bugs.md`'ye, karar `decisions.md`'ye, özet `STATE.md`'ye yazılır.

### Kurallar
- Master ajan her oturum başında STATE.md + project_memory/*.md okur
- Her yazma işlemi öncesi kullanıcıya sorulur
- `bugs.md`: Ajan tekrar aynı hatayı görürse önce burayı kontrol eder
- Ham konuşma geçmişi birikmez → STATE.md'ye sıkıştırılır

---

## 10. CLAUDE.md — STM32 Kodlama Standartları (Proje Bazlı)

Proje klasöründe yer alır. `/new-project` ile `~/.claude/templates/claude-stm32.md` şablonundan kopyalanır.

```markdown
# STM32 Gömülü Sistem Kodlama Standartları

## Zorunlu Kurallar
- Tüm integer'lar explicit-width type: uint8_t, uint32_t (int/long yasak)
- Tüm HAL çağrıları dönüş değeri kontrol edilir (HAL_OK assertion)
- ISR ve main arasında paylaşılan tüm değişkenler volatile zorunlu
- ISR içinde dinamik bellek tahsisi (malloc/free) yasak
- Include guard zorunlu: #ifndef (pragma once değil)
- Max 300 satır per .c dosyası — aşılırsa init/config/runtime/irq böl
- Cortex-M7 DMA transferleri: SCB_CleanDCacheByAddr() / SCB_InvalidateDCacheByAddr() zorunlu
- CubeMX üretilen referans konfigürasyonu ile karşılaştırma zorunlu
- Yapılandırılmış veriler (register map, pin config, clock tree) YAML formatında belgelenir

## Oturum Başlangıcı (Canonical — Bkz. Bölüm 4)
1. Bu CLAUDE.md dosyasını oku
2. STATE.md oku (failure_count, active_prp, son durum)
3. project_memory/*.md oku
4. Kullanıcıya bildir: "Aktif proje: <proje> | MCU: <mcu> | Durum: <özet>"
```

---

## 11. Validation Gates

Her `/execute-prp` sonunda sırasıyla çalışır:

| Gate | Komut | Geçme Koşulu | Başarısız Olursa |
|------|-------|--------------|------------------|
| 1 — Syntax | `arm-none-eabi-gcc -Wall -Werror` | Sıfır hata/uyarı | `COMPILE_ERROR` → firmware-coder |
| 2 — Size | `arm-none-eabi-size` | Flash/RAM bütçe içinde | Boyut raporla, kullanıcıya danış |
| 3 — Static | `cppcheck src/` | Embedded ruleset temiz | Uyarı listesi kullanıcıya sun |
| 4 — Build | `make clean && make` | Başarılı tam derleme | `COMPILE_ERROR` |
| 5 — Hardware | OpenOCD flash + GDB smoke test | Temel peripheral yanıt verir | `HARDWARE_UNAVAILABLE`: kısmi doğrulama olarak kaydet, ROADMAP'ta işaretle |
| 6 — RTOS | `uxTaskGetStackHighWaterMark()` | Watermark > min_threshold | RTOS uzmanı uyar |

**Gate 5 atlandığında:** STATE.md'ye `gate5_pending: true` + `gate5_prp: <prp_yolu>` yazılır. ROADMAP'taki ilgili faz "kısmi" olarak işaretlenir.

**Ertelenmiş Gate 5 Tamamlama (`/verify-hardware`):**
```
/verify-hardware
→ STATE.md'de gate5_pending=true olan tüm PRP'leri listele
→ Kullanıcıya "Donanım bağlı mı?" sor
→ Evet → gate5_prp'deki OpenOCD komutunu çalıştır
        → Başarılı: gate5_pending=false, ROADMAP fazını "tam" olarak güncelle
        → Başarısız: LOGIC_ERROR kategorisine düşür, debugger devreye al
→ Hayır → "gate5_pending PRP'ler: <liste>" bildir, oturumu sonlandır
```

---

## 12. Tasarım Kararları

| Karar | Seçilen | Neden |
|-------|---------|-------|
| Platform | Claude Code CLI only (Faz 1) | Pro/Max aboneliği yeterli, ek API maliyeti yok |
| VS Code Extension | Ertelenmiş (Faz 2) | Temel mimariyi bozmadan sonradan eklenebilir |
| Alt ajan iletişimi | Claude Code Agent tool | Aboneliğe dahil, taze bağlam per görev |
| Hafıza formatı | .md dosyaları | Okunabilir, version control uyumlu, Claude Code native |
| Başarısızlık yanıtı | Kullanıcıya danış (≥2 başarısız) | Şeffaflık öncelikli, kullanıcı döngüde |
| Yapılandırılmış veri formatı | YAML (donanım konfigürasyonları için) | LLM sembolik işleme devrelerini etkinleştirir, insan okunabilir |
| Paralel görev yönetimi | Claude Code Task tool | Wave bağımlılık zincirini doğal olarak destekler |
| CLAUDE.md mimarisi | Global (genel) + Proje bazlı (STM32 standartları) | Proje bağımsızlığı, standart şablon kopyası |

---

## 13. Faz 1 Tamamlanma Kriterleri

Faz 1 (terminal/CLI) şu koşullar sağlandığında tamamlanmış sayılır:
- [ ] Tüm 10 alt ajan `~/.claude/agents/` altında tanımlı ve çalışıyor
- [ ] `/new-project`, `/switch-project`, `/generate-prp`, `/execute-prp` komutları çalışıyor
- [ ] Bir gerçek STM32 projesinde end-to-end PRP pipeline testi başarılı
- [ ] Hafıza sistemi (STATE.md + project_memory/) doğru yazma/okuma yapıyor
- [ ] Başarısızlık sayacı STATE.md'de doğru güncelleniyor

---

## 14. Gelecek Aşamalar

| Faz | Kapsam | Ön Koşul |
|-----|--------|----------|
| Faz 2 | VS Code Extension (TypeScript) + Python backend | Faz 1 tamamlandı |
| Faz 3 | KiCad şematik entegrasyonu | Faz 2 tamamlandı |
| Faz 4 | OpenOCD/GDB doğrudan entegrasyonu | Faz 1 tamamlandı |
| Faz 5 | STM32CubeIDE plugin | Faz 2 tamamlandı |
