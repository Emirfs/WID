---
name: init-project
description: Mevcut bir STM32 projesine embedded-master sistemini ekler. Eksik dosyaları oluşturur, var olanları dokunmaz. /new-project ile oluşturulmamış projeler için kullanılır.
---

# /init-project — Mevcut Projeye Ajan Sistemi Ekleme

Var olan bir projeye embedded-master sistemini entegre etmek için şu adımları izle:

## Adım 1: Mevcut Durumu Tara

Mevcut dizinde hangi dosyaların var olduğunu kontrol et:

```bash
ls -la
ls project_memory/ 2>/dev/null || echo "project_memory/ YOK"
```

Şu dosyaların varlığını tek tek kontrol et ve bir tablo göster:

| Dosya | Durum |
|-------|-------|
| `CLAUDE.md` | ✅ var / ❌ yok |
| `PROJECT.md` | ✅ var / ❌ yok |
| `STATE.md` | ✅ var / ❌ yok |
| `REQUIREMENTS.md` | ✅ var / ❌ yok |
| `ROADMAP.md` | ✅ var / ❌ yok |
| `project_memory/hardware.md` | ✅ var / ❌ yok |
| `project_memory/decisions.md` | ✅ var / ❌ yok |
| `project_memory/bugs.md` | ✅ var / ❌ yok |
| `project_memory/progress.md` | ✅ var / ❌ yok |
| `project_memory/changelog.md` | ✅ var / ❌ yok |

Kullanıcıya tabloyu göster ve devam etmek isteyip istemediğini sor.

## Adım 2: Proje Bilgilerini Topla

Eksik dosyaları doldurmak için gereken bilgileri sor.
Zaten var olan dosyalardan bilgi çıkarabiliyorsan sorma — önce dosyaları oku.

Varsa bu dosyaları oku: `CLAUDE.md`, `README.md`, `CMakeLists.txt`,
`Makefile`, `*.ioc` (CubeMX proje dosyası)

Çıkarılamayan bilgiler için **tek tek** sor:

1. MCU modeli bilinmiyorsa: "Hangi STM32 MCU kullanıyorsunuz?"
2. Clock bilinmiyorsa: "Sistem clock frekansı nedir?"
3. FreeRTOS bilinmiyorsa: "FreeRTOS kullanılıyor mu?"
4. Projenin amacı bilinmiyorsa: "Projenin kısa açıklaması nedir?"

## Adım 3: Eksik Dosyaları Oluştur

**KURAL: Var olan dosyalara dokunma. Sadece eksik olanları oluştur.**

### CLAUDE.md yoksa
`~/.claude/templates/claude-stm32.md` şablonunu kopyala.
Dosya bulunamazsa: "⚠️ Şablon bulunamadı: `~/.claude/templates/claude-stm32.md`"

### PROJECT.md yoksa
```markdown
# Proje: <proje_adı>

## Donanım
- **MCU:** <mcu_modeli>
- **Sistem Clock:** <clock_frekansı>
- **Flash:** <flash_boyutu veya "bilinmiyor">
- **RAM:** <ram_boyutu veya "bilinmiyor">

## Proje Hedefi
<kullanıcıdan alınan açıklama veya README'den çıkarılan>

## RTOS
<FreeRTOS kullanılıyor / kullanılmıyor / bilinmiyor>

## Init-project ile Eklendi
<tarih>
```

### STATE.md yoksa
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
Mevcut projeye embedded-master sistemi eklendi (/init-project).
```

### REQUIREMENTS.md yoksa
Mevcut kod tabanından gereksinimleri çıkarmaya çalış
(README, kaynak dosya yorumları, Makefile hedefleri).
```markdown
# Gereksinimler: <proje_adı>

## Fonksiyonel Gereksinimler
<mevcut kod tabanından çıkarılan veya kullanıcıdan alınan liste>

## Kısıtlar
- Flash: <flash_boyutu veya "bilinmiyor">
- RAM: <ram_boyutu veya "bilinmiyor">
- Clock: <clock_frekansı>

> Not: /init-project ile oluşturuldu. Güncellenmesi gerekebilir.
```

### ROADMAP.md yoksa
```markdown
# Yol Haritası: <proje_adı>

> Not: /init-project ile oluşturuldu. Mevcut projenin durumuna göre güncelle.

## Mevcut Durum
- [ ] Mevcut kodu incele ve tamamlanan özellikleri işaretle

## Devam Eden Çalışmalar
<kullanıcıdan alınan veya kod tabanından çıkarılan>

## Planlanan
<kullanıcıdan alınan>
```

### project_memory/ dizini yoksa
```bash
mkdir -p project_memory
```

### project_memory/hardware.md yoksa
Mevcut kaynak dosyalardan (.ioc, Makefile, stm32xxx_hal_conf.h) bilgi çıkar:
```markdown
# Donanım Notları

## MCU Bilgileri
- **Model:** <mcu_modeli>
- **Core:** <cortex-m serisi>
- **Flash:** <flash veya "bilinmiyor">
- **RAM:** <ram veya "bilinmiyor">

## Konfigürasyon
- **Sistem Clock:** <clock_frekansı>
- **HSE/HSI:** <mevcut koddan çıkarılan veya "bilinmiyor">

## Pin Atamaları
<.ioc dosyasından veya kaynak koddan çıkarılan — yoksa boş bırak>

## Donanım Kısıtları
(Keşfedildikçe eklenecek)

> Not: /init-project ile oluşturuldu. Eksik bilgileri tamamla.
```

### project_memory/decisions.md yoksa
```markdown
# Mimari Kararlar

> Not: /init-project ile oluşturuldu.
> Mevcut kod tabanındaki önemli mimari kararları buraya ekle.

(Henüz kayıt yok — `/embedded-master` oturumlarında eklenir)
```

### project_memory/bugs.md yoksa
```markdown
# Bug Kayıtları

> Not: /init-project ile oluşturuldu.

(Henüz kayıt yok — debugger çıktılarından eklenir)
```

### project_memory/progress.md yoksa
```markdown
# İlerleme Kaydı

> Not: /init-project ile oluşturuldu.

## Tamamlananlar
(Mevcut projeye bakılarak doldurulacak)

## Yapılacaklar
(ROADMAP.md ile senkronize edilecek)
```

### project_memory/changelog.md yoksa
```markdown
# Changelog — <proje_adı>

> Her PRP veya önemli aşama tamamlandığında `github-agent` tarafından güncellenir.
> Format: En yeni aşama en üstte.

---

## [init] — embedded-master Sistemi Eklendi

**Tarih:** <tarih>
**Branch:** <mevcut branch>
**PRP:** null

### Tamamlananlar
- ✅ embedded-master ajan sistemi mevcut projeye entegre edildi (/init-project)
- ✅ Eksik hafıza dosyaları oluşturuldu

### Mevcut Proje Durumu
<kısa özet: neler var, ne tamamlanmış>

### Sonraki Adım
- `/embedded-master` ile oturum başlat
- `/generate-prp` ile devam edilecek görevi tanımla

---
```

## Adım 4: Git Deposu Kontrolü

```bash
git rev-parse --is-inside-work-tree 2>/dev/null && echo "GIT_OK" || echo "GIT_YOK"
```

**Git yoksa:** "Bu proje bir git deposu değil. Git başlatmamı ister misiniz?"
- Evet: `git init && git add -A && git commit -m "chore: init git + add embedded-master system"`
- Hayır: Devam et, uyar: "Git olmadan changelog ve commit özelliği çalışmaz."

**Git varsa:** Eklenen dosyaları commit et:
```bash
git add STATE.md PROJECT.md REQUIREMENTS.md ROADMAP.md CLAUDE.md \
        project_memory/hardware.md project_memory/decisions.md \
        project_memory/bugs.md project_memory/progress.md \
        project_memory/changelog.md 2>/dev/null
git commit -m "chore(init): add embedded-master agent system to existing project

## Neden
Mevcut projeye embedded-master ajan sistemi entegre edildi.
/init-project komutu ile eksik hafıza ve yapılandırma dosyaları oluşturuldu.

## Ne Eklendi
- STATE.md: Proje durumu ve PRP takibi
- project_memory/: hardware, decisions, bugs, progress, changelog
- ROADMAP.md / REQUIREMENTS.md: Proje planlaması

Context:
- /init-project ile oluşturuldu
"
```

## Adım 5: Kullanıcıya Bildir

```
✅ embedded-master sistemi eklendi: <proje_adı>

Oluşturulan dosyalar:
  <oluşturulan dosyaların listesi>

Dokunulmayan dosyalar (zaten vardı):
  <var olan dosyaların listesi>

⚠️ Dikkat: Şu dosyaları manuel olarak gözden geçir:
  <"bilinmiyor" içeren alanlar>

Başlamak için:
  /embedded-master   → oturum başlat, projeyi özetle
  /generate-prp "..."  → ilk görevi tanımla
```
