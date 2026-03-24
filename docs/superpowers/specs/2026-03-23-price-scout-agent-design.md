# Price Scout Agent — Tasarım Spesifikasyonu

**Tarih:** 2026-03-23
**Durum:** Gözden Geçirildi (v4)
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

## Frontmatter Tanımı

Ajan dosyasının başında aşağıdaki frontmatter yer alır. Proaktif tetikleyici ifadeler **`description: >` bloğuna gömülmelidir** — Claude Code sub-ajan sisteminin otomatik aktivasyonu bu alandan çalışır (bkz. `github-agent.md`, `debugger.md` örüntüsü):

```yaml
name: price-scout
description: >
  Donanım bileşeni fiyat karşılaştırma ajanı. Kullanıcı tanımlı platformlarda
  bileşen arar, fiyat/stok/kargo bilgilerini toplayarak detaylı karşılaştırma raporu sunar.
  PROACTIVELY devreye girer: "fiyat bak", "fiyat karşılaştır", "nereden alayım",
  "en ucuz", "ucuz", "stok", "stokte var mı", "sipariş", "satın al", "temin et",
  "price check", "compare prices", "where to buy", "cheapest", "in stock",
  "order", "purchase", "find component" ifadelerinde.
tools: WebSearch, WebFetch
```

`WebSearch` ve `WebFetch` araçları Claude Code'da yerleşik olarak mevcuttur ve ajan tarafından doğrudan kullanılabilir.

---

## Mimari

### Çalışma Akışı

```
Kullanıcı → [bileşen adı + platform listesi + miktar (opsiyonel) + para birimi (opsiyonel)]
    ↓
Platform Döngüsü (her platform için sırayla):
    1. WebSearch  → Arama stratejisi (aşağıya bakın)
    2. WebFetch   → bulunan ürün sayfası URL'si
    3. Parse      → fiyat, stok, kargo, MOQ, para birimi, link
    ↓
Normalizasyon → döviz kuru WebSearch ile çekilir; başarısız olursa orijinal para biriminde raporla
    ↓
Rapor         → karşılaştırma tablosu + en iyi öneri + notlar + durum
```

### Arama Stratejisi (Adım 1)

Her platform için sırayla denenir; ilk başarılı sonuç kullanılır:

1. `"{bileşen}" site:{platform}` — en spesifik
2. `"{bileşen}" buy {platform}` — `site:` operatörü çalışmazsa
3. `"{bileşen}" price {platform}` — genel fiyat araması
4. `"{bileşen}" datasheet {platform}` — sadece part numarası varsa

Adımların hiçbiri ürün sayfası döndürmezse: tablo satırına "Bulunamadı" yaz, sıradaki platforma geç.

### Platform Arama Stratejileri

| Platform Tipi | URL Deseni | Notlar |
|--------------|------------|--------|
| LCSC | `lcsc.com/product-detail/...` | WebFetch genellikle çalışır; fiyat HTML içinde mevcuttur |
| Mouser / DigiKey | `mouser.com/ProductDetail/...` | JavaScript ağırlıklı; WebFetch yetersiz kalabilir → snippet fallback |
| AliExpress | `aliexpress.com/item/...` | WebFetch kısıtlıdır; WebSearch snippet'inden fiyat çekilir, "Tahmini" notu eklenir |
| Türk satıcılar (direnc.net, robotistan.com vb.) | `/urun/...` veya `/product/...` | WebFetch genellikle çalışır |
| Amazon | `amazon.com/dp/...` veya `amazon.com.tr/dp/...` | Anti-bot; snippet fallback uygulanır |
| Genel (tanınmayan platform) | Arama sonucundaki ilk URL — `/product/`, `/item/`, `/detail/`, `/urun/` içeren tercih edilir | Ürün sayfası değilse (kategori, blog, forum) bir sonraki arama sonucuna geçilir |

**Snippet fallback tanımı:** WebFetch başarısız veya yetersiz olduğunda, WebSearch sonucundaki `snippet` alanının (kısa açıklama metni) içinden fiyat bilgisi regex ile çıkarılır. Sadece snippet metninde görünen sayısal değer ve para birimi kullanılır; sayfa içeriğinden ek veri çekilmez.

### Proaktif Tetikleyiciler

Tetikleyici ifadeler frontmatter `description: >` bloğuna gömülür (yukarıya bakın). Ajan şu ifadelerde devreye girer:

**Türkçe:** "fiyat bak", "fiyat karşılaştır", "nereden alayım", "en ucuz", "ucuz", "stok", "stokte var mı", "sipariş", "satın al", "temin et"

**İngilizce:** "price check", "compare prices", "where to buy", "cheapest", "in stock", "order", "purchase", "find component"

---

## Girdi Yapısı

| Alan | Zorunluluk | Açıklama |
|------|-----------|----------|
| Bileşen | Zorunlu | Aranacak bileşen adı veya part numarası |
| Platformlar | Zorunlu | Virgülle ayrılmış site listesi (ör: `lcsc.com, mouser.com`) |
| Miktar | Opsiyonel | Toplam maliyet hesabı için adet (varsayılan: 1) |
| Para Birimi | Opsiyonel | Normalizasyon birimi — varsayılan olarak hem USD hem TRY gösterilir; tek birim belirtilirse yalnızca o gösterilir |

**Örnek çağrı (varsayılan, ikili para birimi):**
```
Bileşen: STM32F103C8T6
Platformlar: lcsc.com, mouser.com, direnc.net
Miktar: 10
```

**Örnek çağrı (tek para birimi):**
```
Bileşen: STM32F103C8T6
Platformlar: lcsc.com, mouser.com
Miktar: 5
Para Birimi: USD
```

---

## Çıktı Yapısı

```markdown
## Fiyat Karşılaştırma Raporu — {Bileşen}
*{YYYY-MM-DD HH:MM UTC} tarihinde alınan fiyatlar*

| Platform | Birim Fiyat (USD) | Birim Fiyat (TRY) | MOQ | Stok | Kargo | {Miktar} Adet Toplam (USD) | {Miktar} Adet Toplam (TRY) | Link |
|----------|-------------------|-------------------|-----|------|-------|----------------------------|----------------------------|------|
| ...      | ...               | ...               | ... | ...  | ...   | ...                        | ...                        | [↗]  |

*Kullanıcı tek para birimi belirtmişse ilgili sütunlar teke indirilir.*

### En İyi Öneri
{Miktar} adet için → **{Platform}** ({toplam fiyat}, {kargo durumu})

### Notlar
- MOQ uyarıları (varsa): "{Platform} için minimum sipariş miktarı {MOQ} adettir"
- Kritik stok uyarıları (varsa): "{Platform} stok kritik: {stok} adet"
- Tahmini fiyat uyarıları (varsa): "{Platform} fiyatı tahminidir, satın almadan önce doğrulayın"
- Döviz kuru uyarısı (normalizasyon başarısız olduysa): "Döviz kuru alınamadı; {Platform} fiyatı orijinal para biriminde ({birim}) gösteriliyor"

### Kullanıcı İçin Notlar
- Fiyatlar anlık olup değişkenlik gösterebilir; siparişten önce platformu ziyaret edin
- "Tahmini" işaretli fiyatlar WebSearch snippet'inden alınmıştır, daha az güvenilirdir
- Kargo ücretleri ülkeye ve sipariş büyüklüğüne göre farklılık gösterebilir

## Durum
BAŞARILI | KISMEN_TAMAMLANDI | BAŞARISIZ
```

**Durum değerleri:**
- `BAŞARILI` — en az bir platformdan tam fiyat bilgisi alındı
- `KISMEN_TAMAMLANDI` — bazı platformlar erişilemedi veya tahmini fiyat kullanıldı
- `BAŞARISIZ` — hiçbir platformdan fiyat bilgisi alınamadı

---

## En İyi Öneri Seçim Mantığı

Sıralama kriteri (sırayla uygulanır):

1. **Toplam maliyet** (birim fiyat × miktar + kargo) — en düşük kazanır
2. Kargo bilinmiyorsa: **birim fiyat** baz alınır; raporda "kargo hariç" notu eklenir
3. MOQ > istenen miktar olan platformlar öneri dışı bırakılır; tabloda gösterilir ama öneri yapılmaz
4. Eşitlik durumunda: **stok** daha yüksek olan tercih edilir
5. "Erişilemedi" veya "Tahmini" işaretli platformlar öneri dışıdır; tabloda gösterilir

---

## MOQ (Minimum Sipariş Miktarı) Yönetimi

- WebFetch içeriğinden veya snippet'ten MOQ bilgisi çıkarılmaya çalışılır
- MOQ tespit edilemezse tablo sütununa "?" yazılır
- MOQ > istenen miktar ise:
  - Tablo satırına MOQ değeri yazılır
  - "En İyi Öneri" bölümünden bu platform çıkarılır
  - Notlar bölümüne uyarı eklenir

---

## Para Birimi Normalizasyonu

1. Hedef para birimi kullanıcıdan alınır. Varsayılan: **hem USD hem TRY** — iki sütun gösterilir. Kullanıcı tek birim belirtirse yalnızca o sütun gösterilir.
2. Platformlardan çekilen her benzersiz para birimi (USD, TRY, CNY, EUR, GBP vb.) için ayrı bir döviz kuru WebSearch sorgusu yapılır: `"{kaynak} to {hedef} exchange rate today"` — örn. `"CNY to USD exchange rate today"`
3. Başarılı olursa: ilgili platform fiyatı hedef para birimine çevrilir, tabloda dönüştürülmüş değer gösterilir
4. Birden fazla kaynak para birimi varsa: her biri için ayrı kur çekilir; başarılı olanlar çevrilir, başarısız olanlar orijinal para biriminde bırakılır
5. **Başarısız olursa (kur alınamadı, belirsiz sonuç veya birden fazla kur döndü):** o platform için normalizasyon atlanır, orijinal para biriminde gösterilir, Notlar bölümüne uyarı eklenir

---

## Hata Yönetimi

| Durum | Davranış |
|-------|----------|
| Platform erişilemiyor | Tablo satırına "Erişilemedi" yaz, diğer platformlara devam et |
| Ürün bulunamıyor | Tablo satırına "Bulunamadı" yaz |
| Anti-bot koruması (fetch engeli) | WebSearch snippet'inden tahmini fiyat çıkar, "Tahmini — doğrulayın" notu ekle |
| Fiyat çıkarılamıyor | "Fiyat alınamadı, linki ziyaret edin" notu ekle |
| Döviz kuru alınamıyor | Orijinal para biriminde raporla, döviz uyarısı ekle |
| MOQ > istenen miktar | Tabloda göster, öneri dışı bırak, uyarı ekle |
| Tüm platformlar başarısız | Durum: BAŞARISIZ; "Hiçbir platformdan veri alınamadı" mesajı |

---

## Anti-Pattern'lar

Ajan şunları YAPMAMALDIR:

- Hata döndüren bir platformu "En İyi Öneri" olarak göstermek
- "Tahmini" işaretli fiyatı kesin fiyatmış gibi sunmak
- Önbelleklenmiş veya eski fiyatı güncel fiyat olarak raporlamak — her çağrıda taze veri çekilmeli
- MOQ > miktar olan bir platformu önerilen seçenek olarak sunmak
- Döviz kuru başarısız olduğunda sessizce yanlış dönüştürme yapmak
- Kullanıcının belirtmediği platformları araştırmak

---

## Ajan Dosyası Yapısı

`agents/price-scout.md` aşağıdaki bölümleri içerecek:

1. **Frontmatter** — `name`, `description: >` (tetikleyiciler dahil), `tools: WebSearch, WebFetch`
2. **Görev Tanımı** — ajanın amacı ve kapsamı
3. **Çalışma Protokolü** — adım adım arama, fetch, parse akışı; arama stratejisi sırası
4. **Proaktif Tetikleyiciler** — Türkçe ve İngilizce ifade listesi (frontmatter'daki ile aynı, referans için)
5. **Platform Arama Stratejileri** — platform tipine göre URL desenleri, snippet fallback tanımı, fallback kuralları
6. **Para Birimi Normalizasyonu** — kur çekme, çok kaynaklı senaryo, başarısız durum yönetimi
7. **MOQ ve En İyi Öneri Mantığı** — seçim kriterleri
8. **Rapor Formatı** — çıktı şablonu (Kullanıcı İçin Notlar ve Durum alanları dahil)
9. **Anti-Pattern'lar** — yapılmayacaklar listesi

---

## Kısıtlamalar

- Fiyatlar dinamiktir; rapor üretildiği anı yansıtır (`YYYY-MM-DD HH:MM UTC` formatında belirtilir)
- Bazı platformlar (Mouser, DigiKey, Amazon) JavaScript gerektirdiğinden WebFetch tam içerik döndürmeyebilir; bu durumda snippet fallback uygulanır ve "Tahmini" notu eklenir
- Döviz kuru için harici bir API kullanılmaz; ajan en güncel kuru WebSearch ile bulur; başarısız olursa orijinal para biriminde raporlar

---

## Başarı Kriterleri

- Kullanıcı verilen platformlardan en az birinde fiyat bilgisi alabilmeli
- Rapor tablo + en iyi öneri + notlar + durum formatında sunulmalı
- Erişilemeyen platformlar raporu bozmamalı, uyarıyla geçilmeli
- MOQ > miktar olan platformlar öneri dışı bırakılmalı ama tabloda gösterilmeli
- Döviz kuru başarısız olduğunda sessiz hata yerine açık uyarı verilmeli
- Ajan mevcut WID ajan yapısıyla (frontmatter, durum bloğu, anti-pattern'lar) tamamen uyumlu olmalı
