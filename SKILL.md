---
name: serp-analyzer
description: >
  AI-powered SERP analysis skill for Claude Code. Takes a target keyword
  and optionally your own URL, then performs SERP feature detection,
  content gap analysis, and competitor comparison using Ahrefs API + WebFetch.
  Outputs: word count, H1/H2 count, FAQ yes/no, Schema type, Internal Link
  count, Backlink count (DR>40), brand + term search volume, content gaps.
  Use when user says "SERP analizi", "SERP analysis", "rakip analizi",
  "competitor analysis", "anahtar kelime analizi", "keyword analysis",
  "SERP karsilastirma", or provides a keyword for SERP-level comparison.
user-invokable: true
argument-hint: "<keyword> [our-url] [country:tr]"
license: MIT
metadata:
  author: inbound-seo
  version: "1.1.0"
  category: seo
  requires:
    - "Ahrefs MCP server (mcp__ahrefs__*)"
    - "WebFetch tool"
  sources:
    - "Ahrefs SERP Overview API"
    - "Ahrefs Keywords Explorer API"
    - "Ahrefs Site Explorer API"
---

# SERP Analizi Sistemi v1

Bu skill, hedef keyword icin SERP'teki ilk 10 sonucu analiz eder.
Her rakip sayfa icin on-page ve off-page metrikleri toplar,
content gap'leri tespit eder ve aksiyon listesi olusturur.

Ahrefs MCP server bagli olmalidir. WebFetch ile sayfa icerikleri cekilir.

## Giris Formati

Kullanici su sekillerde giris yapabilir:

**Basit:** `/serp-analyzer 5g`

**Kendi URL'si ile:** `/serp-analyzer 5g https://turkcell.com.tr/5g/`

**Ulke belirterek:** `/serp-analyzer 5g https://turkcell.com.tr/5g/ country:tr`

**Sadece keyword + ulke:** `/serp-analyzer misir turlari country:tr`

Varsayilan ulke: `tr` (Turkiye)

## Analiz Adimlari

### Adim 1 — Keyword Metriklerini Cek (Ahrefs)

`mcp__ahrefs__keywords-explorer-overview` ile keyword'un metriklerini al:

```
select: volume,difficulty,cpc,cps,parent_topic,parent_volume
country: [ulke kodu]
keywords: [hedef keyword]
```

Cikartilacak veriler:
- **Search Volume** (aylik arama hacmi)
- **Keyword Difficulty** (KD skoru 0-100)
- **CPC** (tiklanma basina maliyet — USD cent, 100'e bol)
- **CPS** (clicks per search)
- **Parent Topic** ve hacmi

### Adim 2 — Brand + Term Arama Hacmi (Ahrefs)

Eger kullanici kendi URL'sini verdiyse, brand keyword tespiti yap:
- Domain'den brand adi cikar (ornek: turkcell.com.tr → "turkcell")
- `mcp__ahrefs__keywords-explorer-overview` ile su keyword'leri sorgula:
  - `[brand]` (ornek: "turkcell")
  - `[brand] + [keyword]` (ornek: "turkcell 5g")

Cikartilacak: brand arama hacmi, brand+term arama hacmi

### Adim 3 — SERP Overview (Ahrefs)

`mcp__ahrefs__serp-overview` ile SERP'teki ilk 10 sonucu al:

```
select: url,title,position,backlinks,dofollow,refdomains,domain_rating,url_rating,traffic,keywords
country: [ulke kodu]
keyword: [hedef keyword]
top_positions: 10
```

Bu veriler SERP tablosunun temelini olusturur.

### Adim 4 — SERP Feature Detection

SERP overview sonuclarindan SERP feature'lari tespit et:
- Featured Snippet (position 0)
- People Also Ask (PAA)
- Knowledge Panel
- Video carousel
- Image pack
- Local pack
- Shopping results
- Sitelinks

Not: Ahrefs SERP overview bu feature'larin cogunu position ve URL
yapisindan cikarabilir. Ek olarak keyword overview'daki SERP feature
bilgilerini kullan.

### Adim 5 — Her Rakip Sayfa Icin On-Page Analiz (Cascade Fetch)

SERP'teki her URL icin sayfa icerigini al ve metrikleri cikar.
**Sadece Ahrefs verisine guvenme — her sayfayi dogrudan fetch et.**

#### Dual-Fetch Stratejisi (IKI KATMANLI — her URL icin)

**KRITIK**: WebFetch HTML'i markdown'a cevirirken `<script>` taglarini
siler. Bu yuzden JSON-LD schema, meta description gibi veriler KAYBOLUR.
Her URL icin iki katmanli fetch yap:

##### Katman A — Raw HTML (Schema + Meta Tags)

Bash tool ile `curl` komutu calistir:
```bash
curl -sL -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) \
AppleWebKit/537.36" "[URL]" | python3 -c "
import sys, re, json
html = sys.stdin.read()
blocks = re.findall(r'<script[^>]*type=\"application/ld\+json\"[^>]*>(.*?)</script>', html, re.DOTALL)
schemas = []
for b in blocks:
    try:
        d = json.loads(b.strip())
        schemas.append(d.get('@type','unknown'))
    except: pass
meta = re.search(r'<meta[^>]*name=[\"']description[\"'][^>]*content=[\"'](.*?)[\"']', html, re.I)
micro = re.findall(r'itemtype=[\"']https?://schema\.org/(\w+)[\"']', html)
print(json.dumps({'json_ld': schemas, 'microdata': list(set(micro)), 'meta_desc': meta.group(1) if meta else None}))
"
```

curl basarisiz olursa (bot korumasi, "Request Rejected"):
→ Schema ve meta'yi "N/A (bot korumasi)" olarak isaretle

##### Katman B — Icerik (Word Count, Headings, Links)

Sirasyla dene:
1. **WebFetch** — cogu site icin calisir
2. **Google Cache** — `webcache.googleusercontent.com/search?q=cache:[URL]`
3. **Ahrefs Cache** — `mcp__ahrefs__site-audit-page-content`
4. **Chrome MCP** — `tabs_create_mcp` → `navigate` → `get_page_text` + `read_page`

Hicbiri calismadiysa → "N/A" olarak isaretle, analizi DURDURMA.

##### Veri Birlestirme
- Schema → Katman A'dan (raw HTML, guvenilir)
- Meta desc → Katman A'dan (raw HTML, guvenilir)
- Word count, H1/H2, links → Katman B'den (WebFetch/Chrome)

#### Cikarilacak metrikler (her URL icin):

1. **Word Count**: Body text'in kelime sayisi
2. **H1 Count**: Sayfadaki H1 tag sayisi ve icerigi
3. **H2 Count**: Sayfadaki H2 tag sayisi ve icerikleri
4. **FAQ Var/Yok**: Sayfada FAQ bolumu var mi?
   - JSON-LD FAQPage schema kontrolu (Katman A!)
   - HTML'de "FAQ", "Sikca Sorulan Sorular", "Frequently Asked" text kontrolu
   - Soru-cevap formati (dt/dd, accordion, details/summary) kontrolu
5. **Schema Turu**: JSON-LD + Microdata type'lari (Katman A!)
6. **Internal Link Count**: Ayni domain'e giden linklerin sayisi

### Adim 6 — Her Rakip Icin Backlink Analizi (Ahrefs)

Her SERP URL'si icin `mcp__ahrefs__site-explorer-referring-domains` ile
DR>40 olan referring domain sayisini al:

```
select: domain_rating
target: [rakip URL]
mode: exact
where: domain_rating >= 40
limit: 1000
```

Toplam satir sayisi = DR>40 backlink count.

Alternatif olarak SERP overview'dan gelen `refdomains` ve `domain_rating`
verilerini dogrudan kullan (daha hizli, daha az API call).

### Adim 7 — Content Gap Analizi

SERP'teki tum sayfalarin on-page metriklerini karsilastir ve gap'leri tespit et:

1. **Word Count Gap**: Bizim sayfamiz SERP ortalamasinin altinda mi?
2. **Heading Gap**: Rakipler daha fazla H2 mi kullaniyor?
3. **FAQ Gap**: Rakiplerde FAQ var bizde yok mu?
4. **Schema Gap**: Rakiplerde schema var bizde yok mu?
5. **Internal Link Gap**: Rakipler daha fazla internal link mi kullaniyor?
6. **Backlink Gap**: DR>40 backlink sayimiz rakiplerden dusuk mu?
7. **Keyword Coverage Gap**: Rakiplerin kapsadigi ama bizim kapsamadigimiz alt konular

Not: Content gap analizi sadece kullanici kendi URL'sini verdiyse yapilir.
URL verilmediyse sadece SERP karsilastirma tablosu goster.

## Cikti Formati

### ZORUNLU CIKTI FORMATI — Bu formattan sapma!

```
═══════════════════════════════════════════════════
  SERP ANALİZ RAPORU v1
═══════════════════════════════════════════════════

Keyword:  [hedef keyword]
Ulke:     [ulke]
Tarih:    [bugunun tarihi]

───────────────────────────────────────────────────
  KEYWORD METRİKLERİ
───────────────────────────────────────────────────

  Search Volume:      [aylik hacim]
  Keyword Difficulty: [KD]/100  [zorluk seviyesi]
  CPC:                $[deger]
  Clicks/Search:      [CPS]
  Parent Topic:       [topic] (vol: [hacim])

  Brand Aramalari:
  - "[brand]":              [hacim]/ay
  - "[brand] + [keyword]":  [hacim]/ay

───────────────────────────────────────────────────
  SERP FEATURES
───────────────────────────────────────────────────

  ✓ Featured Snippet    — [varsa URL]
  ✗ People Also Ask
  ✓ Video Carousel      — [varsa kaynak]
  ✗ Knowledge Panel
  ✓ Image Pack
  ✗ Local Pack
  ✗ Shopping

───────────────────────────────────────────────────
  RAKİP KARŞILAŞTIRMA TABLOSU
───────────────────────────────────────────────────

| # | URL | DR | Word | H1 | H2 | FAQ | Schema | Int.Link | BL(DR>40) | Traffic |
|---|-----|----|----- |----|----| --- |--------|----------|-----------|---------|
| 1 | example.com/... | 78 | 3200 | 1 | 8 | ✓ | Article | 24 | 45 | 12.5K |
| 2 | competitor.com/... | 65 | 2800 | 1 | 6 | ✗ | — | 18 | 22 | 8.2K |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |

SERP Ortalamalari:
  Word Count:    [ort] kelime
  H2 Count:      [ort]
  FAQ Orani:     %[oran] ([X]/10 sayfada)
  Int. Link:     [ort]
  BL (DR>40):    [ort]

[Eger kullanicinin URL'si SERP'te varsa, o satiri ★ ile isaretle]

───────────────────────────────────────────────────
  CONTENT GAP ANALİZİ
───────────────────────────────────────────────────

(Bu bolum sadece kullanici kendi URL'sini verdiyse gosterilir)

Sizin Sayfaniz vs SERP Ortalamasi:

| Metrik | Sizin | SERP Ort. | Fark | Durum |
|--------|-------|-----------|------|-------|
| Word Count | 1500 | 2800 | -1300 | 🔴 Eksik |
| H2 Count | 4 | 7 | -3 | 🟡 Dusuk |
| FAQ | ✗ | %60 | — | 🔴 Eksik |
| Schema | — | %70 Article | — | 🟡 Eksik |
| Int. Link | 8 | 18 | -10 | 🔴 Eksik |
| BL (DR>40) | 5 | 25 | -20 | 🔴 Eksik |

Durum kodlari:
  🔴 SERP ortalamasinin %50'sinden az (kritik gap)
  🟡 SERP ortalamasinin %50-80'i arasi (iyilestirilebilir)
  🟢 SERP ortalamasi veya uzeri (iyi)

───────────────────────────────────────────────────
  ÖNCELİKLİ AKSİYON LİSTESİ
───────────────────────────────────────────────────

Gap buyuklugune ve etkisine gore sirali:

1. 🔴 [Metrik] — [somut aksiyon onerisi]
   Mevcut: [deger] → Hedef: [SERP ort. veya uzeri]

2. 🟡 [Metrik] — [somut aksiyon onerisi]
   Mevcut: [deger] → Hedef: [deger]

3. ...

Her aksiyon TURKCE ve SOMUT olmali.

───────────────────────────────────────────────────
  RAKİP İÇERİK TEMATİK ANALİZ
───────────────────────────────────────────────────

Rakiplerin H2 basliklarindan cikarilan ortak temalar:

| Tema / Alt Konu | Kac Rakipte Var | Sizde Var mi? |
|-----------------|-----------------|---------------|
| [tema 1] | 8/10 | ✓ |
| [tema 2] | 7/10 | ✗ ← Ekle |
| [tema 3] | 6/10 | ✗ ← Ekle |
| ... | ... | ... |

Bu tablo, rakiplerin H2 basliklarini semantic olarak gruplar.
"Sizde Var mi?" sutunu sadece kullanici URL verdiyse doldurulur.

───────────────────────────────────────────────────
  METODOLOJİ
───────────────────────────────────────────────────
Veri kaynaklari: Ahrefs SERP Overview, Keywords Explorer,
Site Explorer Referring Domains API.
On-page veriler: WebFetch ile HTML analizi.
```

## Onemli Kurallar

### Veri Kaynaklari — Sadece Ahrefs'e Guvenme!
- Ahrefs off-page veriler (DR, backlink, traffic, KD) icin kullan
- On-page veriler (word count, H1/H2, FAQ, schema, internal link) icin
  HER ZAMAN sayfayi dogrudan fetch et ve kendin analiz et
- Ahrefs verisi ile kendi fetch verini karsilastir — tutarsizlik varsa
  kendi fetch verini once tut (guncel veri)
- Ahrefs bagli degilse veya API hatasi verirse, on-page analizi yine de
  WebFetch/Chrome ile yap — raporu Ahrefs'siz de uretebilmelisin
  (DR, backlink, traffic sutunlari "N/A" olur)

### Fetch Kurallari
- Cascade fetch sirasini takip et: WebFetch → Google Cache → Ahrefs Cache → Chrome
- Bir yontem basarisiz olursa sessizce bir sonrakine gec
- Hangi yontemle basarili oldugunu rapor sonunda belirt
- Fetch basarisiz olan sayfalari tabloda "N/A" olarak goster, analizi DURDURMA
- WebFetch ile sayfa cekme: paralel degil sirayla yap (rate limit korumasi)

### API ve Format Kurallari
- Ahrefs API'dan gelen monetary degerler (CPC, traffic value) USD **cent** cinsindendir — 100'e bolup goster
- `mcp__ahrefs__doc` tool'unu HER Ahrefs tool'unu ilk kez kullanmadan once cagir
- SERP overview'dan gelen `domain_rating` ve `refdomains` verilerini mumkunse dogrudan kullan (API tasarrufu)
- DR>40 backlink count icin ek API call sadece detayli analiz istenirse yap
- Brand keyword tespitinde domain'den brand cikarilamazsa (subdomain, path-only URL) kullaniciya sor
- Content gap analizinde "bizim sayfamiz" sadece kullanici URL verdiyse belirlenir
- Tematik analiz icin rakiplerin H2 basliklarini semantic olarak grupla, birebir karsilastirma yapma
- Turkiye disindaki ulkeler icin `country:us`, `country:de` vb. destekle
- Raporu Turkce yaz (varsayilan), kullanici Ingilizce isterse Ingilizce yaz
- En fazla 10 SERP sonucu analiz et
- Ahrefs render tool'larini (`render-data-table`, `render-scorecard`) mumkunse kullan
