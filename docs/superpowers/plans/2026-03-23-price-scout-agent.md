# Price Scout Agent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** WID sistemine donanım bileşeni fiyat karşılaştırma ajanı ekle (`agents/price-scout.md`).

**Architecture:** Tek bir markdown ajan dosyası oluşturulur. Ajan, platform başına WebSearch → WebFetch → Parse döngüsü çalıştırır; sonuçları USD+TRY ikili fiyat tablosu ve "En İyi Öneri" ile raporlar. Mevcut WID ajan formatını (frontmatter, Anti-patterns, Durum bloğu) takip eder.

**Tech Stack:** Claude Code agent markdown format, WebSearch, WebFetch tools

**Spec:** `docs/superpowers/specs/2026-03-23-price-scout-agent-design.md`
**Reference agent:** `agents/github-agent.md`

---

## Dosya Yapısı

| Dosya | İşlem | Sorumluluk |
|-------|--------|------------|
| `agents/price-scout.md` | Oluştur | Tüm ajan tanımı |

---

## Task 1: Frontmatter ve Görev Tanımı

**Files:**
- Create: `agents/price-scout.md`

- [ ] **Adım 1: Dosyayı oluştur — frontmatter + Görev Tanımı**

Aşağıdaki içeriği `agents/price-scout.md` olarak yaz (frontmatter `---` ile açılıp kapanır, `description: >` bloğu tüm tetikleyicileri içerir):

    ---
    name: price-scout
    description: >
      Donanım bileşeni fiyat karşılaştırma ajanı. Kullanıcı tanımlı platformlarda
      bileşen arar, fiyat/stok/kargo bilgilerini toplayarak detaylı karşılaştırma raporu sunar.
      PROACTIVELY devreye girer: "fiyat bak", "fiyat karşılaştır", "nereden alayım",
      "en ucuz", "ucuz", "stok", "stokte var mı", "sipariş", "satın al", "temin et",
      "price check", "compare prices", "where to buy", "cheapest", "in stock",
      "order", "purchase", "find component" ifadelerinde.
    tools: WebSearch, WebFetch
    ---

    # Price Scout — Donanım Bileşeni Fiyat Karşılaştırma Ajanı

    ## Görev Tanımı

    Kullanıcının belirttiği donanım bileşenini, kullanıcının listesindeki platformlarda arar.
    Her platform için fiyat, stok, kargo ve MOQ bilgisini çeker; USD ve TRY cinsinden
    karşılaştırmalı tablo ile en iyi öneriyi sunar.

    **Kapsam:** Elektronik komponentler (direnç, kondansatör, IC, MCU), geliştirme kartları,
    modüller, mekanik/endüstriyel parçalar — her türlü donanım bileşeni.

    **Araçlar:** WebSearch (arama + snippet fallback), WebFetch (ürün sayfası içeriği)

    ---

- [ ] **Adım 2: Frontmatter doğrula**

Dosyayı oku; şunları kontrol et:
- `---` ile başlayıp kapanıyor mu?
- `tools: WebSearch, WebFetch` satırı var mı?
- `description: >` bloğu hem Türkçe hem İngilizce tetikleyicilerin tümünü içeriyor mu?

- [ ] **Adım 3: Commit**

```bash
git add agents/price-scout.md
git commit -m "feat(agent): add price-scout skeleton — frontmatter and task definition"
```

---

## Task 2: Girdi Yapısı, Proaktif Tetikleyiciler ve Çalışma Protokolü

**Files:**
- Modify: `agents/price-scout.md`

- [ ] **Adım 1: Girdi yapısı bölümünü ekle**

Dosyaya ekle (iç içe kod bloğu sorununu önlemek için örnek girdiler 4 boşluk girintili yazılır):

    ## Girdi Yapısı

    | Alan | Zorunluluk | Açıklama |
    |------|-----------|----------|
    | Bileşen | Zorunlu | Aranacak bileşen adı veya part numarası |
    | Platformlar | Zorunlu | Virgülle ayrılmış site listesi (ör: lcsc.com, mouser.com) |
    | Miktar | Opsiyonel | Toplam maliyet hesabı için adet — varsayılan: 1 |
    | Para Birimi | Opsiyonel | USD, TRY veya belirtilmezse her ikisi birden |

    **Örnek (ikili para birimi — varsayılan):**

        Bileşen: STM32F103C8T6
        Platformlar: lcsc.com, mouser.com, direnc.net
        Miktar: 10

    **Örnek (tek para birimi):**

        Bileşen: STM32F103C8T6
        Platformlar: lcsc.com, mouser.com
        Miktar: 5
        Para Birimi: USD

    ---

- [ ] **Adım 2: Proaktif Tetikleyiciler bölümünü ekle (spec section 4 — zorunlu)**

    ## Proaktif Tetikleyiciler

    Aşağıdaki ifadeler kullanıldığında bu ajan devreye girer (frontmatter'da da tanımlıdır):

    **Türkçe:** "fiyat bak", "fiyat karşılaştır", "nereden alayım", "en ucuz", "ucuz",
    "stok", "stokte var mı", "sipariş", "satın al", "temin et"

    **İngilizce:** "price check", "compare prices", "where to buy", "cheapest",
    "in stock", "order", "purchase", "find component"

    ---

- [ ] **Adım 3: Çalışma Protokolü bölümünü ekle**

    ## Çalışma Protokolü

    Aşağıdaki adımları sırayla uygula:

    ### 1. Girdi Doğrulama
    - Bileşen ve Platformlar zorunlu — eksikse kullanıcıdan iste
    - Miktar yoksa 1 kabul et
    - Para Birimi yoksa hem USD hem TRY göster

    ### 2. Platform Döngüsü (her platform için sırayla)

    **2a. Ürün Sayfasını Bul (WebSearch)**

    Her platform için aşağıdaki sorguları sırayla dene; ilk başarılı sonucu kullan:
    1. "{bileşen}" site:{platform}
    2. "{bileşen}" buy {platform}
    3. "{bileşen}" price {platform}
    4. "{bileşen}" datasheet {platform}

    Hiçbiri ürün sayfası döndürmezse → tablo satırına "Bulunamadı", sıradaki platforma geç.

    **2b. İçerik Çek (WebFetch)**

    Bulunan URL'ye WebFetch yap. Başarılı olursa HTML içeriğinden parse et.
    Başarısız olursa (anti-bot, JS engeli) → snippet fallback uygula (aşağıya bak).

    **2c. Veri Çıkar**

    Her platform için şunları bul:
    - Birim fiyat + para birimi
    - Stok miktarı
    - Kargo ücreti (bilinmiyorsa "?" yaz)
    - MOQ — minimum sipariş miktarı (bilinmiyorsa "?" yaz)
    - Ürün sayfası URL'si

    ### 3. Döviz Kurunu Çek

    Para birimi normalizasyonu için gereken her kur çifti için WebSearch yap:
    "{kaynak} to {hedef} exchange rate today" — ör. "CNY to USD exchange rate today"

    Başarısız olursa → o platform orijinal para biriminde kalır, Notlar'a uyarı ekle.

    ### 4. Normalizasyon ve Hesaplama

    - Fiyatları hedef para birimine çevir (varsayılan: hem USD hem TRY)
    - Toplam = birim fiyat × miktar + kargo
    - Kargo bilinmiyorsa toplam = birim fiyat × miktar ("kargo hariç" notu ekle)

    ### 5. En İyi Öneri Seç

    Sıralama kriterleri (sırayla):
    1. En düşük toplam maliyet (fiyat × miktar + kargo)
    2. Kargo bilinmiyorsa en düşük birim fiyat
    3. MOQ > miktar ise öneri dışı (tabloda göster, önerme)
    4. Eşitlikte daha yüksek stok tercih edilir
    5. "Erişilemedi" veya "Tahmini — doğrulayın" işaretli platformlar öneri dışıdır

    ### 6. Raporu Oluştur

    Rapor formatı: aşağıdaki "Rapor Formatı" bölümüne göre

    ---

- [ ] **Adım 4: Snippet Fallback tanımını ekle**

    ## Snippet Fallback

    WebFetch başarısız veya yetersiz olduğunda:
    1. WebSearch sonucundaki snippet alanının (kısa açıklama metni) içinden fiyat bilgisini regex ile çıkar
    2. Sadece snippet metninde görünen sayısal değer ve para birimini kullan
    3. Ürün sayfasından ek veri çekme
    4. Tablo satırına fiyat bilgisini "Tahmini — doğrulayın" etiketiyle yaz
    5. Notlar bölümüne: "{Platform} fiyatı tahminidir, satın almadan önce doğrulayın"

    ---

- [ ] **Adım 5: Bu noktaya kadar dosyayı doğrula**

Dosyayı oku; şunları kontrol et:
- `## Girdi Yapısı` bölümü var mı?
- `## Proaktif Tetikleyiciler` bölümü var mı (hem Türkçe hem İngilizce)?
- `## Çalışma Protokolü` 6 adım içeriyor mu?
- `## Snippet Fallback` "Tahmini — doğrulayın" etiketini kullanıyor mu?

- [ ] **Adım 6: Commit**

```bash
git add agents/price-scout.md
git commit -m "feat(agent): add price-scout input, triggers, workflow protocol, snippet fallback"
```

---

## Task 3: Platform Arama Stratejileri

**Files:**
- Modify: `agents/price-scout.md`

- [ ] **Adım 1: Platform stratejileri tablosunu ekle**

    ## Platform Arama Stratejileri

    | Platform Tipi | URL Deseni | Fetch Durumu | Notlar |
    |--------------|------------|-------------|--------|
    | LCSC | lcsc.com/product-detail/... | WebFetch çalışır | Fiyat HTML içinde mevcut |
    | Mouser / DigiKey | mouser.com/ProductDetail/... | Snippet fallback | JavaScript ağırlıklı |
    | AliExpress | aliexpress.com/item/... | Snippet fallback | WebFetch kısıtlı |
    | Türk satıcılar (direnc.net, robotistan.com vb.) | /urun/... veya /product/... | WebFetch çalışır | HTML erişilebilir |
    | Amazon | amazon.com/dp/... veya amazon.com.tr/dp/... | Snippet fallback | Anti-bot koruması |
    | Genel (tanınmayan platform) | İlk ürün sayfası URL'si | Dene, gerekirse fallback | /product/, /item/, /detail/, /urun/ içeren URL tercih et |

    **Tanınmayan platformlarda URL seçimi:**
    - WebSearch sonuçlarından /product/, /item/, /detail/, /urun/ içeren URL'yi tercih et
    - Kategori sayfası, blog veya forum URL'si ise bir sonraki arama sonucuna geç
    - Hiçbiri ürün sayfasına benzemiyorsa "Bulunamadı" yaz

    ---

- [ ] **Adım 2: Commit**

```bash
git add agents/price-scout.md
git commit -m "feat(agent): add price-scout platform search strategies"
```

---

## Task 4: Para Birimi Normalizasyonu ve MOQ+Öneri Mantığı

**Files:**
- Modify: `agents/price-scout.md`

- [ ] **Adım 1: Para birimi normalizasyonu bölümünü ekle**

    ## Para Birimi Normalizasyonu

    1. Varsayılan: hem USD hem TRY sütunları göster. Kullanıcı tek birim belirtmişse yalnızca o sütun.
    2. Platformlardan çekilen her benzersiz para birimi için ayrı kur sorgusu yap:
       "{kaynak} to {hedef} exchange rate today" — ör. "CNY to USD exchange rate today"
    3. Başarılı kur → fiyatı çevir, tabloda dönüştürülmüş değeri göster
    4. Birden fazla kaynak para birimi varsa (USD, CNY, EUR, GBP vb.) → her biri için ayrı kur çek
    5. Kur alınamazsa veya belirsiz sonuç varsa → o platform orijinal para biriminde kalır,
       Notlar'a: "Döviz kuru alınamadı; {Platform} fiyatı {birim} cinsinden gösteriliyor"

    ---

- [ ] **Adım 2: MOQ ve En İyi Öneri Mantığı bölümünü ekle (tek bölüm — spec section 7)**

    ## MOQ ve En İyi Öneri Mantığı

    ### MOQ (Minimum Sipariş Miktarı) Tespiti
    - WebFetch içeriği veya snippet'ten MOQ'yu bulmaya çalış
    - MOQ bulunamazsa → tablo sütununa "?" yaz
    - MOQ > istenen miktar ise:
      - Tablo satırında MOQ değerini göster
      - "En İyi Öneri" bölümünden çıkar
      - Notlar'a: "{Platform} için minimum sipariş miktarı {MOQ} adettir"

    ### En İyi Öneri Seçim Kriterleri (öncelik sırası)
    1. En düşük toplam maliyet (birim fiyat × miktar + kargo)
    2. Kargo bilinmiyorsa en düşük birim fiyat; raporda "kargo hariç" notu ekle
    3. MOQ > miktar olan platformlar öneri dışı (tabloda gösterilir)
    4. Eşitlikte daha yüksek stok tercih edilir
    5. "Erişilemedi" veya "Tahmini — doğrulayın" işaretli platformlar öneri dışıdır

    ---

- [ ] **Adım 3: Commit**

```bash
git add agents/price-scout.md
git commit -m "feat(agent): add price-scout currency normalization and MOQ+recommendation logic"
```

---

## Task 5: Hata Yönetimi

**Files:**
- Modify: `agents/price-scout.md`

- [ ] **Adım 1: Hata yönetimi tablosunu ekle**

    ## Hata Yönetimi

    | Durum | Davranış |
    |-------|----------|
    | Platform erişilemiyor | Tablo satırına "Erişilemedi" yaz, diğer platformlara devam et |
    | Ürün bulunamıyor | Tablo satırına "Bulunamadı" yaz |
    | Anti-bot / JS engeli | Snippet fallback — "Tahmini — doğrulayın" notu ekle |
    | Fiyat çıkarılamıyor | "Fiyat alınamadı, linki ziyaret edin" |
    | Döviz kuru alınamıyor | Orijinal para biriminde raporla, Notlar'a uyarı ekle |
    | MOQ > istenen miktar | Tabloda göster, öneri dışı bırak, Notlar'a uyarı ekle |
    | Tüm platformlar başarısız | Durum: BAŞARISIZ — "Hiçbir platformdan veri alınamadı" |

    ---

- [ ] **Adım 2: Commit**

```bash
git add agents/price-scout.md
git commit -m "feat(agent): add price-scout error handling table"
```

---

## Task 6: Rapor Formatı

**Files:**
- Modify: `agents/price-scout.md`

- [ ] **Adım 1: Rapor formatını ekle**

    ## Rapor Formatı

    Aşağıdaki formatı birebir kullan. Zaman damgası YYYY-MM-DD HH:MM UTC formatında olmalıdır.

    ───────────────────────────────────────────

    ## Fiyat Karşılaştırma Raporu — {Bileşen}
    *{YYYY-MM-DD HH:MM UTC} tarihinde alınan fiyatlar*

    | Platform | Birim Fiyat (USD) | Birim Fiyat (TRY) | MOQ | Stok | Kargo | {Miktar} Adet Toplam (USD) | {Miktar} Adet Toplam (TRY) | Link |
    |----------|-------------------|-------------------|-----|------|-------|----------------------------|----------------------------|------|
    | LCSC     | $0.85             | ₺28.90            | 1   | 5000+| $3.20 | $11.70                     | ₺398.20                    | [↗](url) |
    | Mouser   | $1.12 (Tahmini)   | ₺38.08 (Tahmini)  | 1   | 2341 | Ücretsiz | $11.20                 | ₺380.80                    | [↗](url) |

    *Kullanıcı tek para birimi belirtmişse (ör. Para Birimi: USD), TRY sütunları kaldırılır.*

    ### En İyi Öneri
    {Miktar} adet için → **{Platform}** ({toplam_usd} / {toplam_try}, {kargo_durumu})

    ### Notlar
    - {Varsa MOQ uyarıları}
    - {Varsa stok uyarıları}
    - {Varsa "Tahmini — doğrulayın" uyarıları}
    - {Varsa döviz kuru uyarıları}

    ### Kullanıcı İçin Notlar
    - Fiyatlar anlık olup değişkenlik gösterebilir; siparişten önce platformu ziyaret edin
    - "Tahmini" işaretli fiyatlar WebSearch snippet'inden alınmıştır, daha az güvenilirdir
    - Kargo ücretleri ülkeye ve sipariş büyüklüğüne göre farklılık gösterebilir

    ## Durum
    BAŞARILI

    ───────────────────────────────────────────

    **Durum değerleri:**
    - BAŞARILI — en az bir platformdan tam fiyat bilgisi alındı
    - KISMEN_TAMAMLANDI — bazı platformlar erişilemedi veya tahmini fiyat kullanıldı
    - BAŞARISIZ — hiçbir platformdan fiyat bilgisi alınamadı

    ---

- [ ] **Adım 2: Commit**

```bash
git add agents/price-scout.md
git commit -m "feat(agent): add price-scout report format with dual currency template"
```

---

## Task 7: Anti-Pattern'lar ve Final Doğrulaması

**Files:**
- Modify: `agents/price-scout.md`

- [ ] **Adım 1: Anti-Pattern'lar bölümünü ekle**

    ## Anti-patterns

    - Hata döndüren bir platformu "En İyi Öneri" olarak gösterme
    - "Tahmini" işaretli fiyatı kesin fiyatmış gibi sunma
    - Önbelleklenmiş veya eski fiyatı güncel fiyat olarak raporlama — her çağrıda taze veri çek
    - MOQ > miktar olan bir platformu önerilen seçenek olarak sunma
    - Döviz kuru başarısız olduğunda sessizce yanlış dönüştürme yapma
    - Kullanıcının belirtmediği platformları araştırma
    - Stok veya kargo bilinmiyorken "0" veya "ücretsiz" varsayma
    - Zaman damgasını rapor başlığından çıkarma — her çağrıda YYYY-MM-DD HH:MM UTC formatında yaz

- [ ] **Adım 2: Spec'e göre final yapısal kontrol**

Şu soruları cevapla (hepsi "evet" olmalı):
- [ ] Frontmatter `description: >` tüm tetikleyici ifadeleri içeriyor mu?
- [ ] `tools: WebSearch, WebFetch` var mı?
- [ ] `## Proaktif Tetikleyiciler` body bölümü var mı?
- [ ] 4 adımlı arama stratejisi (site: → buy → price → datasheet) yazıldı mı?
- [ ] Snippet fallback "Tahmini — doğrulayın" etiketini kullanıyor mu?
- [ ] Platform tablosunda 6 satır var mı?
- [ ] `## MOQ ve En İyi Öneri Mantığı` tek birleşik bölüm olarak var mı?
- [ ] Döviz normalizasyonu çok kaynaklı (CNY, EUR, GBP) senaryoyu kapsıyor mu?
- [ ] Hata tablosunda 7 satır ve "Tahmini — doğrulayın" etiketi var mı?
- [ ] Rapor şablonunda `## Durum`, `### Kullanıcı İçin Notlar`, `## Anti-patterns` var mı?
- [ ] Zaman damgası YYYY-MM-DD HH:MM UTC formatında mı?
- [ ] İkili para birimi varsayılan, tek birim opsiyonel olarak belirtildi mi?
- [ ] Dosya yapısı `agents/github-agent.md` ile tutarlı mı (frontmatter, bölüm sırası)?

- [ ] **Adım 3: Test — Tek Platform, Başarılı Senaryo**

Ajanı şu girdiyle çağır:

    Bileşen: 100nF 0402 kapasitör
    Platformlar: lcsc.com
    Miktar: 100

Beklenen çıktı: fiyat tablosu (birim fiyat + stok), USD ve TRY sütunları, `## Durum: BAŞARILI`

- [ ] **Adım 4: Test — Çoklu Platform, Karma Senaryo**

    Bileşen: ESP32-WROOM-32
    Platformlar: lcsc.com, mouser.com, direnc.net
    Miktar: 5

Beklenen:
- LCSC → tam fiyat
- Mouser → "Tahmini — doğrulayın" (snippet fallback)
- Direnc.net → tam fiyat veya "Bulunamadı"
- `## Durum: BAŞARILI` veya `KISMEN_TAMAMLANDI`

- [ ] **Adım 5: Test — Tüm Platformlar Başarısız Senaryosu**

    Bileşen: XYZ-NONEXISTENT-PART-99999
    Platformlar: lcsc.com, mouser.com
    Miktar: 1

Beklenen: tüm platformlar "Bulunamadı", `## Durum: BAŞARISIZ`

- [ ] **Adım 6: Reference agent karşılaştırması**

`agents/github-agent.md` ile yan yana karşılaştır:
- Frontmatter formatı uyuşuyor mu?
- Çıktı bloğu `## Durum` içeriyor mu?
- `## Kullanıcı İçin Notlar` bölümü var mı?
- `## Anti-patterns` bölümü var mı?

- [ ] **Adım 7: Final commit**

```bash
git add agents/price-scout.md
git commit -m "$(cat <<'EOF'
feat(agent): complete price-scout hardware component price comparison agent

Donanım bileşeni fiyat karşılaştırma ajanı tamamlandı.

## Ne Eklendi
- agents/price-scout.md: tam ajan tanımı
- WebSearch + WebFetch ile platform başına sıralı arama
- 4 adımlı arama stratejisi (site:, buy, price, datasheet)
- Snippet fallback — "Tahmini — doğrulayın" etiketi (Mouser, DigiKey, Amazon, AliExpress)
- USD + TRY ikili para birimi (varsayılan), tek birim opsiyonel
- Çok kaynaklı döviz normalizasyonu (CNY, EUR, GBP vb.)
- MOQ ve En İyi Öneri Mantığı tek bölümde
- 7 hata durumu + Durum bloğu + Kullanıcı İçin Notlar
- Anti-patterns bölümü (zaman damgası zorunluluğu dahil)
- 3 test senaryosu (başarılı, karma, tümü başarısız)

Spec: docs/superpowers/specs/2026-03-23-price-scout-agent-design.md

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```
