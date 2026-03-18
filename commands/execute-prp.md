---
name: execute-prp
description: Belirtilen PRP dosyasını uygular. Wave bazlı Task paralel yürütme, alt ajan çağrısı, validation gates ve STATE.md güncelleme işlemlerini gerçekleştirir.
---

# /execute-prp — PRP Uygulama

PRP dosyasını uygulamak için şu adımları izle:

## Adım 0: Argüman Kontrolü

Argüman verilmediyse STATE.md'yi oku ve `active_prp` alanını kontrol et:
- `active_prp` null değilse: "Aktif PRP: `<active_prp>`. Bunu çalıştırmamı ister misiniz? (evet/hayır)"
- `active_prp` null ise: "Lütfen PRP dosya yolunu belirtin. Örnek: `/execute-prp PRPs/uart2-debug-console.md`"

## Adım 1: PRP ve Bağlamı Yükle

1. Belirtilen PRP dosyasını oku (`PRPs/<dosya>.md`)
2. STATE.md oku (failure_count, önceki blokajlar)
3. project_memory/*.md oku

## Adım 2: Kullanıcıya Açıkla

```
🚀 PRP Yürütme Başlıyor: <prp_adı>
   Assigned Agent: <assigned_agent>
   Wave sayısı: <n>
   Validation Gates: Gate 1-4 (zorunlu) + Gate 5 (donanım gerekirse) + Gate 6 (RTOS varsa)
```

## Adım 3: Context Payload Hazırla

Her alt ajan çağrısı için payload:
```
## CONTEXT PAYLOAD
### STATE.md
<STATE.md içeriği>

### PRP
<PRP içeriği>

### HAL Header Excerpt
<Varsa, kullanıcıdan onay alınarak ilgili header>

### Görev
<Wave içindeki spesifik alt görev>
```

## Adım 4: Wave Bazlı Yürütme

> **Önemli Not:** Wave bağımlılık sıralaması (Wave N+1'in Wave N'i beklemesi) bu prompttaki talimata dayalı bir davranıştır — Claude Code tarafından yapısal olarak zorlanmaz. Doğrulama yöntemi manuel gözlemdir: Wave 1 tüm görevleri tamamlanmadan Wave 2 çıktısı görünmemeli. Test adımı: Task 9, Adım 7e'ye bak.

Her wave için:
1. Wave içindeki bağımsız görevleri **paralel** Task olarak başlat
2. Tüm Task'lar BAŞARILI dönene kadar bekle
3. Herhangi biri BAŞARISIZ dönerse:
   - failure_count++ (STATE.md'de)
   - Hata kategorisini belirle (COMPILE_ERROR / LOGIC_ERROR / AGENT_FAILURE / UNKNOWN)
   - failure_count ≥ 2 ise: "Bu yaklaşım 2 kez işe yaramadı. Farklı yol deneyelim mi?" (kullanıcıya sor, failure_count=0)
   - failure_count < 2 ise: farklı yaklaşımla tekrar dene
4. Wave BAŞARILI → bir sonraki wave'e geç

## Adım 5: Validation Gates Çalıştır

PRP'nin Wave görev listesinden hedef `.c` dosyalarını çıkar, sırayla çalıştır:
```bash
# Gate 1 (PRP'deki hedef .c dosyası için)
arm-none-eabi-gcc -Wall -Werror -c src/<prp-den-okunan-dosya>.c
# Gate 2
arm-none-eabi-size build/firmware.elf
# Gate 3 (PRP'deki hedef .c dosyası için)
cppcheck --enable=all src/<prp-den-okunan-dosya>.c
# Gate 4
make clean && make
# Gate 5 (donanım varsa)
# Kullanıcıya sor: "STM32 bağlı mı?"
# Gate 6 (RTOS varsa)
# Stack watermark kontrolü kodu ekle
```

Gate başarısız olursa:
- COMPILE_ERROR (Gate 1/4): firmware-coder'ı tekrar çağır
- SIZE_OVERFLOW (Gate 2): kullanıcıya bildir, optimizasyon iste
- STATIC_FAIL (Gate 3): uyarıları sun
- HARDWARE_UNAVAILABLE (Gate 5): STATE.md'ye `gate5_pending:true`, `gate5_prp:<yol>` yaz

## Adım 6: Tamamlama

Tüm gates geçtikten sonra:
1. STATE.md güncelle: `failure_count=0`, `active_prp=null`, `gate5_pending` durumu
2. ROADMAP.md'de ilgili fazı işaretle
3. project_memory/progress.md güncelle (tamamlanan PRP'yi ekle)
4. **github-agent** çağır — aşağıdaki iki işlemi yaptır:
   - `project_memory/changelog.md`'ye bu PRP için yeni entry ekle (ne tamamlandı, ne düzeltildi, öğrenilenler)
   - Enriched commit at: tüm değişen dosyaları stage'e al, PRP adını ve sonuçlarını commit body'ye yaz

```
✅ PRP Tamamlandı: <prp_adı>
   Tüm validation gates geçti.
   STATE.md güncellendi.
   ROADMAP: <faz> → tamamlandı ✓
   changelog.md güncellendi.
   Enriched commit atıldı.
```
