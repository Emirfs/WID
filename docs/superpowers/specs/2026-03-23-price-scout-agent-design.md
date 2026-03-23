# Price Scout Agent — Tasarım Spesifikasyonu

**Tarih:** 2026-03-23
**Durum:** Taslak
**Konum:** `agents/price-scout.md`

---

## Genel Bakış

`price-scout`, donanım bileşenlerini kullanıcı tarafından belirlenen platformlarda arayan, fiyat/stok/kargo bilgilerini toplayarak karşılaştırmalı rapor sunan bir Claude Code ajanıdır. WID sistemindeki diğer ajanlarla aynı yapısal formatı takip eder.

---

## Amaç ve Kapsam

- Her türlü donanım bileşenini kapsar: elektronik komponentler (direnç, kondansatör, IC, MCU), geliştirme kartları, modüller, mekanik/endüstriyel parçalar
- Platform listesi kullanıcı tarafından belirlenir; sabit bir platform listesi yoktur
- Ajan, Claude Code içinden çağrılır (`agents/price-scout.md`)

---

## Mimari

### Çalışma Akışı

```
Kullanıcı → [bileşen adı + platform listesi + miktar (opsiyonel) + para birimi (opsiyonel)]
    ↓
Platform Döngüsü (her platform için sırayla):
    1. WebSearch  → "{bileşen} site:{platform}"
    2. WebFetch   → bulunan ürün sayfası URL'si
    3. Parse      → fiyat, stok, kargo, para birimi, link
    ↓
Normalizasyon → tüm fiyatlar hedef para birimine çevrilir (varsayılan: USD)
    ↓
Rapor         → karşılaştırma tablosu + en iyi öneri + notlar
```

### Proaktif Tetikleyiciler

Kullanıcı şu ifadeleri kullandığında ajan devreye girer:
- "fiyat bak", "fiyat karşılaştır"
- "nereden alayım", "en ucuz", "ucuz"
- "stok", "stokte var mı"
- "sipariş", "satın al", "temin et"

---

## Girdi Yapısı

| Alan | Zorunluluk | Açıklama |
|------|-----------|----------|
| Bileşen | Zorunlu | Aranacak bileşen adı veya part numarası |
| Platformlar | Zorunlu | Virgülle ayrılmış site listesi (ör: `lcsc.com, mouser.com`) |
| Miktar | Opsiyonel | Toplam maliyet hesabı için adet (varsayılan: 1) |
| Para Birimi | Opsiyonel | Normalizasyon birimi — TRY veya USD (varsayılan: USD) |

**Örnek çağrı:**
```
Bileşen: STM32F103C8T6
Platformlar: lcsc.com, mouser.com, direnc.net
Miktar: 10
Para Birimi: TRY
```

---

## Çıktı Yapısı

```markdown
## Fiyat Karşılaştırma Raporu — {Bileşen}

| Platform | Birim Fiyat | Stok | Kargo | {Miktar} Adet Toplam | Link |
|----------|-------------|------|-------|----------------------|------|
| ...      | ...         | ...  | ...   | ...                  | [↗]  |

### En İyi Öneri
{Miktar} adet için → **{Platform}** ({toplam fiyat}, {kargo durumu})

### Notlar
- Varsa MOQ (minimum sipariş miktarı) uyarıları
- Kritik stok uyarıları
- Tahmini fiyat uyarıları (anti-bot fallback durumunda)
- Fiyat tarihi
```

---

## Hata Yönetimi

| Durum | Davranış |
|-------|----------|
| Platform erişilemiyor | Tablo satırına "Erişilemedi" yaz, diğer platformlara devam et |
| Ürün bulunamıyor | Tablo satırına "Bulunamadı" yaz |
| Anti-bot koruması (fetch engeli) | WebSearch snippet'inden tahmini fiyat çıkar, "Tahmini — doğrulayın" notu ekle |
| Fiyat çıkarılamıyor | "Fiyat alınamadı, linki ziyaret edin" notu ekle |

---

## Ajan Dosyası Yapısı

`agents/price-scout.md` aşağıdaki bölümleri içerecek:

1. **Frontmatter** — name, description, trigger keywords, tools
2. **Görev Tanımı** — ajanın amacı ve kapsamı
3. **Çalışma Protokolü** — adım adım arama, fetch, parse akışı
4. **Platform Arama Stratejileri** — her platform türü için URL desenleri ve arama ipuçları
5. **Para Birimi Normalizasyonu** — döviz çevirimi kuralları
6. **Rapor Formatı** — çıktı şablonu
7. **Önemli Notlar** — anti-bot fallback, MOQ kontrolü, fiyat doğruluk uyarısı

---

## Kısıtlamalar

- Fiyatlar dinamiktir; rapor üretildiği anı yansıtır
- Bazı platformlar (Mouser, DigiKey) JavaScript gerektirdiğinden WebFetch tam içerik döndürmeyebilir; bu durumda fallback stratejisi uygulanır
- Döviz kuru için harici bir API kullanılmaz; ajan en güncel kuru WebSearch ile bulur

---

## Başarı Kriterleri

- Kullanıcı verilen tüm platformlardan en az birinde fiyat bilgisi alabilmeli
- Rapor tablo + öneri + notlar formatında sunulmalı
- Erişilemeyen platformlar raporu bozmamalı, uyarıyla geçilmeli
- Ajan mevcut WID ajan yapısıyla tamamen uyumlu olmalı
