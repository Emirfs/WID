---
name: verify-hardware
description: STATE.md'de gate5_pending=true olan PRP'lerin ertelenmiş donanım doğrulamasını (Gate 5) çalıştırır.
---

# /verify-hardware — Ertelenmiş Gate 5 Tamamlama

## Adım 1: STATE.md Kontrolü

STATE.md'yi oku. `gate5_pending: true` olan kayıtları listele.

Eğer hiç kayıt yoksa:
"✓ Bekleyen donanım doğrulaması yok. Tüm Gate 5'ler tamamlanmış."

## Adım 2: Listeyi Göster

```
Bekleyen donanım doğrulamaları:
1. PRPs/add-spi-dma-driver.md (eklenme: <tarih>)
2. PRPs/uart-dma-rx.md (eklenme: <tarih>)

STM32 bağlı mı ve OpenOCD çalışıyor mu? (evet/hayır)
```

## Adım 3A: Donanım Bağlı (Evet)

Her bekleyen PRP için sırayla:
1. PRP'deki Gate 5 komutunu çalıştır:
   ```bash
   openocd -f interface/stlink.cfg -f target/<mcu>.cfg \
     -c "program build/firmware.elf verify reset exit"
   ```
2. Başarılıysa:
   - STATE.md'de `gate5_pending: false` yap (ilgili PRP için)
   - ROADMAP.md'de ilgili fazı "tam" olarak güncelle
   - Kullanıcıya bildir: "✅ <prp_adı> Gate 5 geçti — ROADMAP güncellendi"
3. Başarısızsa:
   - LOGIC_ERROR olarak kaydet
   - debugger ajanını çağır: "Donanım smoke test başarısız. debugger devreye alıyorum."

## Adım 3B: Donanım Bağlı Değil (Hayır)

```
⏳ Bekleyen Gate 5 doğrulamaları:
   • PRPs/add-spi-dma-driver.md
   • PRPs/uart-dma-rx.md

STM32'yi bağladığınızda `/verify-hardware` komutunu tekrar çalıştırın.
```
