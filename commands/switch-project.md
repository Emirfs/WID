---
name: switch-project
description: Aktif STM32 projesini değiştirir. Mevcut STATE.md'yi kaydeder, hedef projenin hafızasını yükler ve kullanıcıya bildirir.
---

# /switch-project — Proje Değiştirme

Aktif projeyi değiştirmek için şu adımları izle:

## Adım 1: Argüman Kontrolü

Kullanıcı `/switch-project <yol>` şeklinde çağırmalı.
Yol verilmediyse: "Lütfen proje yolunu belirtin: `/switch-project /path/to/project`"

## Adım 2: Mevcut Proje Uyarısı

Mevcut dizinde STATE.md varsa:
- `active_prp` null değilse: "⚠️ Aktif görev var: <active_prp>. Geçmeden önce tamamlamak ister misiniz?"
- Kullanıcı "evet" derse dur, "hayır" derse devam et.

## Adım 3: Hedef Proje Doğrulama

Hedef klasörde şunlar mevcut olmalı:
- `CLAUDE.md` (STM32 standartları)
- `PROJECT.md` (proje tanımı)
- `STATE.md` (proje durumu)

Eksikse: "⚠️ <yol> geçerli bir embedded-master projesi değil. CLAUDE.md ve PROJECT.md bulunamadı."

## Adım 4: Proje Yükle

1. Hedef `CLAUDE.md` oku
2. Hedef `STATE.md` oku
3. Hedef `project_memory/*.md` oku
4. Hedef `ROADMAP.md` oku
5. Hedef `STATE.md`'de `active_prp` null değilse: hedef `active_prp` yolundaki PRP dosyasını oku

## Adım 5: Kullanıcıya Bildir

```
✓ Proje değiştirildi: <yeni_proje_adı>
  MCU: <mcu> | Clock: <clock>
  Son durum: <STATE.md özeti>
  Aktif PRP: <active_prp veya "yok">
  Tamamlanan fazlar: <ROADMAP'tan tamamlananlar>
```
