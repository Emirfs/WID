# ESP32 Agent Sistemi — Design Spec
**Tarih:** 2026-03-18
**Platform:** Claude Code CLI (terminal)
**Kapsam:** ESP32 uzman ajan sistemi — mevcut STM32 embedded-master'a entegrasyon

---

## 1. Genel Bakış

Mevcut STM32 odaklı `embedded-master` sistemine ESP32 desteği eklenir. Yaklaşım: tek komut (`/embedded-master`), platform-aware routing. `STATE.md`'deki `platform` alanına göre STM32, ESP32 veya Mixed mod otomatik seçilir. 7 yeni ESP32 alt ajanı eklenir; özellikle `esp32-bridge` ajanı STM32↔ESP32 ortak projeleri için kritik değer katar.

---

## 2. Mimari

```
embedded-master (tek komut, platform-aware)
│
├── Platform Algılama
│   STATE.md → platform: stm32 | esp32 | mixed
│
├── STM32 Modu (mevcut — dokunulmaz)
│   firmware-coder | hardware-reviewer | debugger
│   security-analyst | doc-generator | test-engineer
│   rtos-specialist | linker-expert | power-management
│   hardware-bringup | schematic-analyst | readme-writer | github-agent
│
├── ESP32 Modu (yeni)
│   esp32-firmware-coder   ← ESP-IDF + Arduino, peripheral sürücüler
│   esp32-wifi             ← Wi-Fi, HTTP, MQTT, WebSocket
│   esp32-ble              ← BLE + Bluetooth Classic, GATT
│   esp32-rtos             ← FreeRTOS (ESP-IDF çift çekirdek farkları dahil)
│   esp32-debugger         ← Core dump, JTAG/OpenOCD, panic analizi
│   esp32-ota              ← OTA, partition table, secure boot
│   esp32-bridge           ← STM32↔ESP32 UART/SPI/I2C protokol tasarımı
│
└── Mixed Mod (yeni)
    Her iki platform ajanına erişim açık
    esp32-bridge her zaman aktif
```

---

## 3. ESP32 Alt Ajan Detayları

Her ajan için frontmatter şeması:
```yaml
---
name: esp32-<isim>
description: >
  [Tek satır açıklama]. PROACTIVELY devreye girer: [tetikleyici keyword'ler].
tools: Read, Write, Edit, Bash, Grep, Glob
---
```
`Bash` tüm ESP32 ajanlarında zorunludur (`idf.py` komutları için). `WebFetch` isteğe bağlıdır.

### `esp32-firmware-coder`
- **Kapsam:** ESP-IDF ve Arduino framework'lerinde kod üretimi
- **Framework seçimi:** `STATE.md`'deki `esp32_framework: idf | arduino` alanından okunur
- **Peripheral şablonları:** GPIO, ADC, DAC, PWM (LEDC), I2S, SPI, I2C, UART
- **Kodlama standartları:** Explicit-width integer'lar (`uint8_t`, `uint32_t`), dönüş değeri kontrolü, ISR'da malloc yasak — STM32 ajanıyla paralel
- **Tetikleyiciler (ESP32 modunda):** `esp32`, `arduino`, `ledc`, `esp_gpio`, `idf`, "yaz", "sürücü", "implement"
- **Not:** "yaz" / "sürücü" / "implement" genel fiilleri; STM32 modunda `firmware-coder`'a, ESP32 modunda `esp32-firmware-coder`'a gider. Mixed modda `esp32_` prefix'li API kelimeleri önceliklidir (bkz. Bölüm 4 Çatışma Çözümü).

### `esp32-wifi`
- **Kapsam:** Wi-Fi bağlantı yönetimi, uygulama katmanı protokolleri
- **API'ler:** ESP-IDF `esp_wifi` + Arduino `WiFi` kütüphanesi
- **Protokoller:** HTTP/HTTPS (`esp_http_client`), MQTT (`esp-mqtt`), WebSocket
- **Özellikler:** TLS sertifika yönetimi, bağlantı kopma/yeniden bağlanma stratejileri
- **Tetikleyiciler:** `esp_wifi`, `wifi`, `mqtt`, `http`, `websocket`, `TLS`

### `esp32-ble`
- **Kapsam:** Bluetooth Low Energy ve Classic Bluetooth
- **Stack'ler:** NimBLE ve Bluedroid
- **Özellikler:** GATT server/client tasarımı, characteristic tanımı, pairing/bonding
- **Senaryo:** STM32 veya telefon ile BLE köprüsü kurma
- **Tetikleyiciler:** `ble`, `bluetooth`, `gatt`, `nimble`, `bluedroid`, `pairing`

### `esp32-rtos`
- **Kapsam:** ESP-IDF FreeRTOS'a özgü görev yönetimi
- **Farklar:** Çift çekirdek mimarisi, `xTaskCreatePinnedToCore`, çift çekirdek mutex
- **Özellikler:** Task-to-core pin stratejisi, inter-task iletişim, watchdog yönetimi
- **Tetikleyiciler:** `xTaskCreatePinnedToCore`, `pinnedToCore`, `esp-idf freertos`, `esp_task_wdt`
- **Not:** Genel "task" / "görev" kelimesi Mixed modda çatışır (bkz. Bölüm 4). ESP32-özgü API prefix'i olmadan bu ajan seçilmez.

### `esp32-debugger`
- **Kapsam:** ESP32'ye özgü hata ayıklama
- **Araçlar:** Panic handler analizi, core dump (`idf.py coredump-info`), JTAG/OpenOCD
- **Özellikler:** Stack trace yorumlama, Arduino Serial debug, ESP-IDF log seviyesi yönetimi
- **Tetikleyiciler:** `core dump`, `panic`, `idf.py`, `openocd`, `esp_log`, `ets_printf`
- **Not:** "donuyor" / "crash" gibi genel kelimeler Mixed modda çatışır (bkz. Bölüm 4). ESP32 özgü prefix olmadan STM32 `debugger` önceliklidir.

### `esp32-ota`
- **Kapsam:** Over-the-air firmware güncelleme
- **Özellikler:** Partition table tasarımı (factory + ota_0 + ota_1), `esp_ota_ops` API
- **Güvenlik:** HTTPS OTA, rollback mekanizması, secure boot + flash encryption entegrasyonu
- **Güvenlik sahipliği:** ESP32 security konuları `esp32-ota` ajanına aittir. STM32 `security-analyst` ESP32 projelerinde çağrılmaz.
- **Tetikleyiciler:** `esp_ota`, `partition table`, `ota update`, `esp_https_ota`, `rollback`

### `esp32-bridge`
- **Kapsam:** STM32↔ESP32 iletişim protokolü tasarımı ve kod üretimi
- **Geçerli platformlar:** Mixed modda her zaman aktif; ESP32-only veya STM32-only modda da çağrılabilir (köprü protokolü tasarlama ihtiyacı platform bağımsız olabilir)
- **Protokoller:** UART framing (başlık + CRC), SPI master/slave rol ataması, I2C adres yönetimi
- **Çıktı:** Her iki taraf için örnek kod — STM32 HAL tarafı + ESP-IDF/Arduino tarafı
- **Hafıza:** Kararlar `project_memory/bridge.md`'ye kaydedilir
- **bridge.md yoksa:** Ajan `project_memory/bridge.md`'yi `templates/bridge-memory.md` şablonundan otomatik oluşturur (tüm platform modlarında geçerli)
- **`templates/bridge-memory.md` yoksa:** Ajan kullanıcıya hata verir: "bridge-memory.md şablonu bulunamadı. Lütfen WID repo'sunu güncelleyin veya `templates/bridge-memory.md` dosyasını manuel oluşturun."
- **Tetikleyiciler:** `bridge`, `stm32+esp32`, `uart protocol`, `spi bridge`, "köprü", "haberleşme"

---

## 4. `embedded-master.md` Değişiklikleri

### Platform Algılama Bloğu (oturum başına eklenir)
```
STATE.md'den platform alanını oku:
- platform: stm32  → Yalnızca STM32 routing aktif (ESP32 routing tablosu devre dışı)
- platform: esp32  → Yalnızca ESP32 routing aktif (STM32 routing tablosu devre dışı)
- platform: mixed  → Her iki routing aktif (çatışma çözümü kuralları geçerli)
- alan yoksa       → STM32 modu varsayılır (platform: stm32 olarak işle,
                     STATE.md'ye yaz, kullanıcıya bildir)
```

**Mevcut STM32 projeleri için geriye uyumluluk:** `platform` alanı olmayan STATE.md → STM32 modu. Kullanıcıya prompt gösterilmez; sadece "platform: stm32 varsayıldı" notu eklenir.

**STM32 routing tablosu ESP32 modunda tamamen devre dışıdır.** Bu mod geçişi keyword-level değil, tablo-level'dır: ESP32 modunda `firmware-coder`, `rtos-specialist`, `debugger` gibi STM32 ajanlarına routing yapılmaz; kullanıcı bu ajanları adıyla açıkça istemediği sürece.

**"STATE.md bulunamadı" mesajı güncellemesi:** Mevcut mesaj ("Bu dizinde STM32 projesi bulunamadı...") şu şekilde değiştirilir: "Bu dizinde gömülü sistem projesi bulunamadı. `/new-project` ile yeni proje oluşturun."

### Mixed Mod Çatışma Çözümü (keyword öncelik sırası)
Mixed modda aynı keyword birden fazla ajana eşleşirse:
1. **ESP32-özgü API prefix'i olan keyword** önceliklidir (`esp_`, `esp32_`, `xTaskCreate`, `idf.py` vb.)
2. **STM32-özgü API prefix'i olan keyword** önceliklidir (`HAL_`, `LL_`, `CMSIS`)
3. **Genel fiil / belirsiz keyword** → kullanıcıya platform sorusu sorulur: "Bu görev STM32 mi ESP32 için mi?"

Örnekler:
- "task yazalım" → belirsiz → kullanıcıya sor
- "FreeRTOS task oluştur" → belirsiz → kullanıcıya sor ("STM32 için mi ESP32 için mi?")
- "xTaskCreatePinnedToCore kullan" → `esp32-rtos` (ESP32-özgü API)
- "HAL_UART_Transmit kullan" → `firmware-coder` (STM32-özgü API)
- "donuyor" → belirsiz → kullanıcıya sor
- "core dump analiz et" → `esp32-debugger` (ESP32-özgü terim)

### ESP32 Keyword Routing (mevcut tabloya eklenir)
```
esp_wifi, wifi, mqtt, http, websocket, TLS                   → esp32-wifi
ble, bluetooth, gatt, nimble, bluedroid, pairing             → esp32-ble
esp_ota, partition table, ota update, esp_https_ota, rollback → esp32-ota
core dump, panic, idf.py, openocd, esp_log, ets_printf        → esp32-debugger
xTaskCreatePinnedToCore, pinnedToCore, esp_task_wdt          → esp32-rtos
bridge, stm32+esp32, uart protocol, spi bridge, köprü        → esp32-bridge
esp32, arduino, ledc, esp_gpio, esp_idf_, esp_err_t          → esp32-firmware-coder
```
**Not:** `idf.py` sadece `esp32-debugger` tetikler. `esp_idf_` (alt çizgiyle biten prefix) `esp32-firmware-coder` tetikler. İkisi çakışmaz.
```

### Oturum Başı Mesajı — Üç Varyant

**STM32 modu (mevcut, değişmez):**
```
🔧 Embedded Master Agent Aktif
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Proje    : <PROJECT.md'den>
MCU      : <hardware.md'den>
Son durum: <STATE.md özeti>
Aktif PRP: <active_prp veya "yok">
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**ESP32 modu (yeni):**
```
🔧 Embedded Master Agent Aktif
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Proje    : <PROJECT.md'den>
Platform : ESP32
SoC      : <STATE.md → esp32_model>
Framework: <STATE.md → esp32_framework>
Son durum: <STATE.md özeti>
Aktif PRP: <active_prp veya "yok">
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Mixed mod (yeni):**
```
🔧 Embedded Master Agent Aktif
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Proje    : <PROJECT.md'den>
Platform : STM32 + ESP32 (Mixed)
MCU/SoC  : <STATE.md → stm32_model veya "N/A"> + <STATE.md → esp32_model veya "N/A">
Framework: <STATE.md → esp32_framework>
Son durum: <STATE.md özeti>
Aktif PRP: <active_prp veya "yok">
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
MCU/SoC alanları için kaynak: önce `STATE.md`, yoksa `project_memory/hardware.md`, ikisi de yoksa "N/A".

---

## 5. `STATE.md` Yeni Alanları

```markdown
platform: mixed          # stm32 | esp32 | mixed (yoksa stm32 varsayılır)
stm32_model: STM32F407   # Mixed projeler için; hardware.md'ye de yazılır
esp32_model: ESP32-S3    # ESP32 | ESP32-S2 | ESP32-S3 | ESP32-C3
esp32_framework: idf     # idf | arduino
esp32_idf_version: 5.1
esp32_clock: 240         # MHz; yoksa: ESP32/S2/S3 → 240, C3 → 160
```

**`platform` alanı yoksa ve `esp32_model` gibi ESP32-özgü alanlar mevcutsa:** Yine de STM32 modu varsayılır. Kullanıcıya bildirilir: "platform alanı bulunamadı, STM32 modu kullanılıyor. ESP32 için STATE.md'ye platform ekleyin."

**`esp32_clock` yoksa varsayılan:** ESP32 / ESP32-S2 / ESP32-S3 → 240 MHz; ESP32-C3 → 160 MHz. `esp32_model` bilinmiyorsa 240 MHz kullanılır.

---

## 6. Proje Hafızası Değişiklikleri

- ESP32 projeleri mevcut `project_memory/` yapısını kullanır
- Mixed projeler için ek dosya: `project_memory/bridge.md`
  - `/new-project` Mixed seçildiğinde `templates/bridge-memory.md`'den otomatik oluşturulur
  - `esp32-bridge` ajanı çağrıldığında dosya yoksa aynı şablondan oluşturulur
  - İçerik şeması (`templates/bridge-memory.md` tarafından tanımlanır, bkz. Bölüm 8)

---

## 7. `/new-project` Komutu Değişikliği

Platform sorusu eklenir:
```
Platform seçin:
1. STM32
2. ESP32
3. Mixed (STM32 + ESP32)
```

**ESP32 seçilirse ek sorular:**
- `esp32_model` sorulur (ESP32 / ESP32-S2 / ESP32-S3 / ESP32-C3)
- `esp32_framework` sorulur (idf / arduino)
- `esp32_idf_version` sorulur (varsayılan: 5.1)

**Oluşturulan STATE.md şablonu (ESP32):**
```markdown
platform: esp32
esp32_model: <kullanıcıdan>
esp32_framework: <kullanıcıdan>
esp32_idf_version: <kullanıcıdan, varsayılan 5.1>
esp32_clock:               # boş, ajan ilk kullanımda sorar
active_prp: null
failure_count: 0
gate5_pending: false
gate5_prp: null
Son Güncelleme: <tarih>
Son Oturum Özeti:
```

**Mixed seçilirse ek sorular:**
- `stm32_model` ve `esp32_model` ayrı ayrı sorulur
- `esp32_framework` sorulur (idf / arduino)
- `esp32_idf_version` sorulur (varsayılan: 5.1)

**Oluşturulan STATE.md şablonu (Mixed):**
```markdown
platform: mixed
stm32_model: <kullanıcıdan>
esp32_model: <kullanıcıdan>
esp32_framework: <kullanıcıdan>
esp32_idf_version: <kullanıcıdan, varsayılan 5.1>
esp32_clock:               # boş, ajan ilk kullanımda sorar
active_prp: null
failure_count: 0
gate5_pending: false
gate5_prp: null
Son Güncelleme: <tarih>
Son Oturum Özeti:
```

Proje kurulum onay mesajı Mixed için:
```
Oluşturulacak dosyalar:
  CLAUDE.md, PROJECT.md, STATE.md, REQUIREMENTS.md, ROADMAP.md
  project_memory/hardware.md
  project_memory/decisions.md
  project_memory/bugs.md
  project_memory/progress.md
  project_memory/bridge.md      ← Mixed'e özgü
Onaylıyor musunuz?
```

---

## 8. Dosya Listesi (Oluşturulacak/Değiştirilecek)

### Yeni Dosyalar
```
agents/esp32-firmware-coder.md
agents/esp32-wifi.md
agents/esp32-ble.md
agents/esp32-rtos.md
agents/esp32-debugger.md
agents/esp32-ota.md
agents/esp32-bridge.md
templates/bridge-memory.md
```

**`project_memory/bridge.md` runtime-generated bir dosyadır — repo'da önceden bulunmaz, proje kurulumunda veya `esp32-bridge` ilk çağrıldığında oluşturulur.**

### `templates/bridge-memory.md` İçerik Şeması
```markdown
# Bridge Protokol Hafızası

## Donanım Bağlantıları
- STM32 pin: [doldur]
- ESP32 pin: [doldur]
- Protokol: UART | SPI | I2C

## Frame Formatı
- Başlık: [doldur]
- Payload maks. boyut: [doldur]
- CRC: [doldur]

## Rol Ataması (SPI için)
- Master: STM32 | ESP32
- Slave: STM32 | ESP32

## Kararlar
| Tarih | Karar | Neden |
|-------|-------|-------|
|       |       |       |
```

### Değiştirilecek Dosyalar
```
commands/embedded-master.md   ← platform algılama + ESP32 routing + 3 banner varyantı
                                 + "STM32 projesi bulunamadı" mesaj güncellemesi
                                 + /new-project açıklaması "Yeni gömülü sistem projesi kur" olarak güncellenir
commands/new-project.md       ← platform sorusu + ESP32/Mixed akışı + STATE.md şablonları + bridge.md oluşturma
```

---

## 9. Tasarım Kararları

| Karar | Tercih | Neden |
|-------|--------|-------|
| Entegrasyon şekli | Mevcut master'a eklenti | Tek komut, tutarlı arayüz |
| ESP32 master | Yok | Gereksiz karmaşıklık |
| Bridge ajanı | Ayrı ajan | Her iki platform context'ini bilmesi gerekiyor |
| Framework desteği | ESP-IDF + Arduino | Her ikisi de yaygın kullanımda |
| STATE.md platform alanı | platform yoksa → stm32 varsayılır | Mevcut projelerle geriye uyumluluk |
| Mixed mod çatışma çözümü | API prefix önceliği, belirsizde kullanıcıya sor | Yanlış ajan seçimi riskini ortadan kaldırır |
| ESP32 security sahibi | esp32-ota | STM32 security-analyst ESP32'ye uygulanamaz |
| esp32_clock alanı | STATE.md'ye eklendi | firmware-coder ve rtos-specialist için gerekli context |
| bridge.md oluşturma | Yoksa ajan oluşturur | Mixed olmayan projelerde de bridge kullanılabilir |
