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

### `esp32-firmware-coder`
- **Kapsam:** ESP-IDF ve Arduino framework'lerinde kod üretimi
- **Framework seçimi:** `STATE.md`'deki `esp32_framework: idf | arduino` alanından okunur
- **Peripheral şablonları:** GPIO, ADC, DAC, PWM (LEDC), I2S, SPI, I2C, UART
- **Kodlama standartları:** Explicit-width integer'lar (`uint8_t`, `uint32_t`), dönüş değeri kontrolü, ISR'da malloc yasak — STM32 ajanıyla paralel
- **Tetikleyiciler:** `esp32`, `arduino`, `ledc`, `esp_gpio`, `idf`, "yaz", "sürücü", "implement"

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
- **Tetikleyiciler:** `xTask`, `pinnedToCore`, `esp-idf freertos`, "görev", "task", "watchdog"

### `esp32-debugger`
- **Kapsam:** ESP32'ye özgü hata ayıklama
- **Araçlar:** Panic handler analizi, core dump (`idf.py coredump-info`), JTAG/OpenOCD
- **Özellikler:** Stack trace yorumlama, Arduino Serial debug, ESP-IDF log seviyesi yönetimi
- **Tetikleyiciler:** `core dump`, `panic`, `idf.py`, `openocd`, "donuyor", "crash"

### `esp32-ota`
- **Kapsam:** Over-the-air firmware güncelleme
- **Özellikler:** Partition table tasarımı (factory + ota_0 + ota_1), `esp_ota_ops` API
- **Güvenlik:** HTTPS OTA, rollback mekanizması, secure boot + flash encryption entegrasyonu
- **Tetikleyiciler:** `esp_ota`, `partition`, `ota update`, "güncelleme", "rollback"

### `esp32-bridge`
- **Kapsam:** STM32↔ESP32 iletişim protokolü tasarımı ve kod üretimi
- **Protokoller:** UART framing (başlık + CRC), SPI master/slave rol ataması, I2C adres yönetimi
- **Çıktı:** Her iki taraf için örnek kod — STM32 HAL tarafı + ESP-IDF/Arduino tarafı
- **Hafıza:** Kararlar `project_memory/bridge.md`'ye kaydedilir
- **Tetikleyiciler:** `bridge`, `stm32+esp32`, `uart protocol`, `spi bridge`, "köprü", "haberleşme"

---

## 4. `embedded-master.md` Değişiklikleri

### Platform Algılama Bloğu (oturum başına eklenir)
```
STATE.md'den platform alanını oku:
- platform: stm32  → STM32 routing aktif
- platform: esp32  → ESP32 routing aktif
- platform: mixed  → her iki routing aktif
- alan yoksa       → kullanıcıya sor, STATE.md'ye yaz
```

### ESP32 Keyword Routing (mevcut tabloya eklenir)
```
esp_wifi, wifi, mqtt, http, websocket, TLS       → esp32-wifi
ble, bluetooth, gatt, nimble, bluedroid, pairing → esp32-ble
esp_ota, partition, ota update, rollback         → esp32-ota
core dump, panic, idf.py, openocd                → esp32-debugger
xTask, pinnedToCore, esp-idf freertos            → esp32-rtos
bridge, stm32+esp32, uart protocol, spi bridge   → esp32-bridge
esp32, arduino, ledc, esp_gpio, idf              → esp32-firmware-coder
```

### Mixed Mod Oturum Başı Mesajı
```
🔧 Embedded Master Agent Aktif
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Proje    : <PROJECT.md'den>
Platform : STM32 + ESP32 (Mixed)
MCU/SoC  : <STM32 modeli> + <ESP32 modeli>
Son durum: <STATE.md özeti>
Aktif PRP: <active_prp veya "yok">
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Nasıl yardımcı olabilirim?
```

---

## 5. `STATE.md` Yeni Alanları

```markdown
platform: mixed          # stm32 | esp32 | mixed
esp32_model: ESP32-S3    # ESP32 | ESP32-S2 | ESP32-S3 | ESP32-C3
esp32_framework: idf     # idf | arduino
esp32_idf_version: 5.1
```

---

## 6. Proje Hafızası Değişiklikleri

- ESP32 projeleri mevcut `project_memory/` yapısını kullanır
- Mixed projeler için ek dosya: `project_memory/bridge.md`
  - İçerik: STM32↔ESP32 protokol kararları, pin atamaları, framing formatı, versiyon notları

---

## 7. `/new-project` Komutu Değişikliği

Platform sorusu eklenir:
```
Platform seçin:
1. STM32
2. ESP32
3. Mixed (STM32 + ESP32)
```

Mixed seçilirse:
- `stm32_model` ve `esp32_model` ayrı ayrı sorulur
- `esp32_framework` sorulur (idf / arduino)
- `project_memory/bridge.md` şablonu otomatik oluşturulur

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

### Değiştirilecek Dosyalar
```
commands/embedded-master.md   ← platform algılama + ESP32 routing
commands/new-project.md       ← platform sorusu + Mixed akışı
```

---

## 9. Tasarım Kararları

| Karar | Tercih | Neden |
|-------|--------|-------|
| Entegrasyon şekli | Mevcut master'a eklenti | Tek komut, tutarlı arayüz |
| ESP32 master | Yok | Gereksiz karmaşıklık |
| Bridge ajanı | Ayrı ajan | Her iki platform context'ini bilmesi gerekiyor |
| Framework desteği | ESP-IDF + Arduino | Her ikisi de yaygın kullanımda |
| STATE.md | `platform` alanı eklendi | Otomatik mod algılama için gerekli |
