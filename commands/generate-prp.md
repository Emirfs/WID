---
name: generate-prp
description: Verilen görev açıklaması için bir PRP (Product Requirements Prompt) dosyası oluşturur. Görev sınıflandırması yapar, uygun alt ajanı belirler ve PRPs/<görev-adı>.md dosyasına yazar.
---

# /generate-prp — PRP Oluşturma

Yeni bir firmware görevi için PRP oluşturmak üzere şu adımları izle:

## Adım 1: Bağlamı Yükle

STATE.md ve project_memory/hardware.md dosyalarını oku.

## Adım 2: Görevi Sınıflandır ve Ajan Belirle

Kullanıcının görev açıklamasını analiz et:

| Görev Türü | assigned_agent |
|-----------|----------------|
| Kod yazma, sürücü geliştirme, HAL/LL API kullanımı | firmware-coder |
| Hata analizi, peripheral debug, HardFault | debugger |
| Şematik/PCB inceleme, devre hesabı | hardware-reviewer |
| Yeni kart devreye alma | hardware-bringup |
| .ld dosyası, flash/RAM overflow | linker-expert |
| FreeRTOS görev tasarımı | rtos-specialist |
| Güvenlik, şifreleme, bootloader | security-analyst |
| Doxygen, dokümantasyon | doc-generator |
| Unit test, HIL | test-engineer |
| Güç tüketimi optimizasyonu | power-management |

## Adım 3: Validation Budget Belirle

hardware.md'den flash ve RAM bilgisini oku.
Mevcut kullanım + tahminî ek kullanım = bütçe.

## Adım 4: PRP Dosyasını Oluştur

`PRPs/<kebab-case-görev-adı>.md` dosyasını aşağıdaki şemaya göre yaz:

```markdown
# PRP: <görev adı>
**Oluşturulma:** <YYYY-MM-DD>
**Assigned Agent:** <ajan-adı>
**Validation Budget:** flash=<mevcut+tahmini>KB, ram=<mevcut+tahmini>KB

## Hedef
<Görevin amacı ve başarı kriterleri>

## Bağlam
- **MCU:** <hardware.md'den>
- **İlgili HAL API'leri:** <görevle ilgili API'ler>
- **Mevcut driver pattern'ları:** <projede varsa benzer sürücüler>
- **Donanım kısıtları:** <pin, DMA kanal, clock bağımlılıkları>
- **Bilinen errata:** <varsa>

## Uygulama Planı (Wave Yapısı)
### Wave 1 (paralel)
- [ ] <bağımsız alt görev 1>
- [ ] <bağımsız alt görev 2>

### Wave 2 (paralel, Wave 1 tamamlanınca)
- [ ] <Wave 1'e bağımlı alt görev>

## Validation Gates
- [ ] Gate 1 — Syntax: `arm-none-eabi-gcc -Wall -Werror src/<dosya>.c`
- [ ] Gate 2 — Size: `arm-none-eabi-size build/firmware.elf` (flash < <budget>KB)
- [ ] Gate 3 — Static: `cppcheck --enable=all src/<dosya>.c`
- [ ] Gate 4 — Build: `make clean && make`
- [ ] Gate 5 — Hardware: `openocd -f interface/stlink.cfg -f target/<mcu>.cfg -c "program build/firmware.elf verify reset exit"` [OPTIONAL — donanım gerekli]
- [ ] Gate 6 — RTOS: `uxTaskGetStackHighWaterMark()` watermark > 50 words [RTOS varsa]

## Anti-patterns
- <Bu görev için özgül kaçınılacak yaklaşımlar>
```

## Adım 5: Kullanıcıya Sun ve Onay Al

PRP'yi kullanıcıya göster:
"PRP oluşturuldu: `PRPs/<dosya-adı>.md`
Assigned Agent: <ajan-adı>
Wave sayısı: <n>
Onaylıyor musunuz? (evet/düzenle)"

Onaylanırsa STATE.md'ye yaz: `active_prp: PRPs/<dosya-adı>.md`
