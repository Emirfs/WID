---
name: new-project
description: Yeni bir STM32 gömülü sistem projesi kurar. Proje klasöründe CLAUDE.md, PROJECT.md, STATE.md, REQUIREMENTS.md, ROADMAP.md ve project_memory/ oluşturur.
---

# /new-project — Yeni STM32 Projesi Kurulumu

Yeni bir STM32 gömülü sistem projesi kurmak için aşağıdaki adımları izle:

## Adım 1: Kullanıcıdan Bilgi Topla

Şu soruları **tek tek** sor (cevap alınmadan bir sonrakine geçme):

1. "Proje adı nedir?" (klasör adı olarak kullanılacak)
2. "Hangi STM32 MCU kullanıyorsunuz?" (örn: STM32F446RE, STM32H743ZI)
3. "Projenin amacı nedir? (kısa açıklama)"
4. "Sistem clock frekansı nedir? (örn: 180MHz, 480MHz)"
5. "FreeRTOS kullanılacak mı?"
6. "Projenin ana gereksinimleri neler?" (virgülle ayırarak liste ver)

## Adım 2: Proje Klasörü Oluştur

Kullanıcının gösterdiği klasörde (veya mevcut dizinde) şunları oluştur:

### CLAUDE.md
`~/.claude/templates/claude-stm32.md` dosyasını oku ve `CLAUDE.md` olarak yaz.
Dosya bulunamazsa kullanıcıya bildir: "⚠️ Şablon bulunamadı: `~/.claude/templates/claude-stm32.md`. Lütfen dosyanın var olduğunu doğrulayın."

### PROJECT.md
```markdown
# Proje: <proje_adı>

## Donanım
- **MCU:** <mcu_modeli>
- **Sistem Clock:** <clock_frekansı>
- **Flash:** <flash_boyutu>
- **RAM:** <ram_boyutu>

## Proje Hedefi
<kullanıcının verdiği açıklama>

## RTOS
<FreeRTOS kullanılıyor / kullanılmıyor>

## Oluşturulma
<tarih>
```

### STATE.md
```markdown
# Proje Durumu

## Genel
- **Aktif Proje:** <proje_adı>
- **MCU:** <mcu>
- **Son Güncelleme:** <tarih>

## Görev Durumu
- **active_prp:** null
- **failure_count:** 0
- **gate5_pending:** false
- **gate5_prp:** null
- **git_language:** null

## Blokajlar
(yok)

## Son Oturum Özeti
Proje oluşturuldu.
```

### REQUIREMENTS.md
```markdown
# Gereksinimler: <proje_adı>

## Fonksiyonel Gereksinimler
<kullanıcının verdiği gereksinim listesi — her satır maddeli>

## Kısıtlar
- Flash: <flash_boyutu>
- RAM: <ram_boyutu>
- Clock: <clock_frekansı>
```

### ROADMAP.md
```markdown
# Yol Haritası: <proje_adı>

## Faz 1: Temel Altyapı
- [ ] Clock tree konfigürasyonu
- [ ] GPIO init
- [ ] Debug UART (console)
**Tamamlanma Kriteri:** LED yanıp sönüyor, UART'tan "Hello" geliyor.

## Faz 2: Peripheral Sürücüleri
- [ ] <kullanıcının ihtiyacına göre sürücüler>
**Tamamlanma Kriteri:** Her peripheral loopback testi geçiyor.

## Faz 3: Uygulama Katmanı
- [ ] <uygulama modülleri>
**Tamamlanma Kriteri:** Tüm gereksinimler karşılanıyor.
```

### project_memory/hardware.md
```markdown
# Donanım Notları

## MCU Bilgileri
- **Model:** <mcu_modeli>
- **Core:** Cortex-M<x>
- **Flash:** <flash>
- **RAM:** <ram>
- **Clock (max):** <max_clock>

## Konfigürasyon
- **Sistem Clock:** <clock_frekansı>
- **HSE/HSI:** <seçilen kaynak>

## Pin Atamaları
(Peripheral'lar eklendikçe güncellenecek)

## Donanım Kısıtları
(Keşfedildikçe eklenecek)
```

### project_memory/decisions.md
```markdown
# Mimari Kararlar
(Henüz kayıt yok — kararlar `/embedded-master` oturumlarında eklenir)
```

### project_memory/bugs.md
```markdown
# Bug Kayıtları
(Henüz kayıt yok — debugger çıktılarından eklenir)
```

### project_memory/progress.md
```markdown
# İlerleme Kaydı

## Tamamlananlar
(Henüz yok)

## Yapılacaklar
<ROADMAP.md'deki Faz 1 görevleri>
```

### project_memory/changelog.md
```markdown
# Changelog — <proje_adı>

> Her PRP veya önemli aşama tamamlandığında `github-agent` tarafından güncellenir.
> Format: En yeni aşama en üstte.

---

## [başlangıç] — Proje Oluşturuldu

**Branch:** main
**Commit:** —
**PRP:** null

### Tamamlananlar
- ✅ Proje yapısı oluşturuldu (CLAUDE.md, PROJECT.md, STATE.md, ROADMAP.md)
- ✅ project_memory/ dizini hazırlandı

### Sonraki Adım
- İlk PRP: `/generate-prp` ile Faz 1 başlat

---
```

## Adım 3: Kullanıcıya Bildir

```
✓ Proje oluşturuldu: <proje_adı>
  MCU: <mcu> | Clock: <clock>

Oluşturulan dosyalar:
  CLAUDE.md, PROJECT.md, STATE.md, REQUIREMENTS.md, ROADMAP.md
  project_memory/hardware.md, decisions.md, bugs.md, progress.md, changelog.md

Sonraki adım: /generate-prp ile ilk görevinizi tanımlayın.
Örnek: /generate-prp "Temel GPIO ve LED blink implementasyonu"
```
