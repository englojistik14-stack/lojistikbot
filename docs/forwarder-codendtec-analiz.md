# Forwarder (codendtec) Operasyon Paneli – Kılavuz ve ENG TMS Aktarım Raporu

**İncelenen adres:** https://forwarder.codendtec.app/login
**Tarih:** 25.11.2026
**Yöntem:** Giriş yapılmadan, sitenin herkese açık sunduğu 62 JS/CSS paketinin (≈1,6 MB) statik analizi. Panele giriş yapılıp ekranlar gezilmedi; aşağıdaki bilgiler kaynak koddaki ekran metinleri, veri modelleri ve iş kurallarından çıkarıldı.

---

## 1. Genel tespit

| Konu | Bulgu |
|---|---|
| Ürün adı (sayfa başlığı) | "ENG Global Logistics - Operasyon Paneli" – ayarlardaki firma adı `ENG Global Logistics & Forwarding` |
| Teknoloji | React 19 + Vite (rolldown), TanStack Query/Router, Tailwind, Leaflet harita, Lucide ikonlar |
| Backend | **Yok.** Tüm veri tarayıcının `localStorage`'ında (`forwarder-db` anahtarı) mock veri olarak tutuluyor. Entegrasyon testlerinde "canlı bağlantı backend fazında doğrulanacak" yazıyor. |
| Kimlik doğrulama | Demo: 3 sabit kullanıcıdan biri seçiliyor (Selin Kara – Yönetici, Ayşe Yılmaz – Operasyon, Kerem Aslan – Muhasebe). Şifre/sunucu kontrolü yok. |
| Yedekleme | Tarayıcıda son 10 yedek (`forwarder-backups`), "Demo verilerini sıfırla", JSON dışa aktar/geri yükle |
| Odak | Türkiye → Avrupa **hafif ticari / panelvan ekspres** taşımacılık (Doblò, Ducato, Crafter, Daily), parsiyel/ekspres forwarding |

**Sonuç:** Bu, ENG için hazırlanmış bir **tıklanabilir prototip / ürün tasarımı**. Gerçek veri yok; değeri, iş akışı tasarımında, veri modellerinde ve iş kurallarında.

---

## 2. Roller ve menü

| Rol | Panel adı | Görebildiği modüller |
|---|---|---|
| Operasyon | Operasyon Paneli | Ana Ekran, Bildirimler, Yük Talepleri, Seferler, Firmalar, Filo & Sürücüler, Yük Hesaplama, Rota Planlama |
| Muhasebe | Muhasebe Paneli | Ana Ekran, Bildirimler, Faturalar, Tahsilat & Ödeme, Cari & Mutabakat, Banka, Raporlar, Firmalar, Şoför Hesapları (araç bildirimleri gizli) |
| Yönetici | Yönetim Paneli | Hepsi + Ayarlar & Bakım |

Kısayollar: `G A` Ana ekran, `G B` Bildirimler, `G Y` Yük talepleri, `G S` Seferler, `G F` Faturalar, `G R` Raporlar, `Ctrl+K` genel arama, `Ctrl+J` panel asistanı (sohbet botu).

---

## 3. Modül modül kılavuz

### 3.1 Ana Ekran (`/`)
- Rol bazlı karşılama ve **"Bekleyen işler"** listesi; her kalem ilgili kayda götürür:
  - Talep WhatsApp'ta paylaşılmadı · X teklif geldi, müşteriye teklif ver
  - Nakliyeciden plaka alınmadı · Bugünkü konum müşteriye gönderilmedi
  - CMR sisteme girilmedi · Satış faturası kesilmedi (CMR hazır/bekleniyor)
  - Araç belgesi (muayene/kasko/yeşil sigorta/bakım) X gün içinde bitiyor
  - Mutabakat bekliyor / fark var
- Yönetici: Ciro / Maliyet / Kâr (son 4 hafta, önceki 4 haftaya göre), aktif sefer, gümrükteki sefer, haftalık ciro-maliyet grafiği.
- Muhasebe: Açık alacak/borç, geciken alacak/borç, eşleşmemiş banka hareketi, "bu hafta tahsil edilecek / ödenecek".

### 3.2 Yük Talepleri (`/loads`) – satış/teklif hunisi
**Akış (durumlar):** `Yeni` → `Gruplarda` (WhatsApp'ta paylaşıldı) → `Teklif verildi` → `Onaylandı` (sefer açılır).

1. **Talep girişi:** Outlook/Gmail'den gelen e-posta okunur (veya metin yapıştırılır). Ayrıştırıcı şu alanları çıkarır: müşteri, yükleme yeri, teslim yeri, yük cinsi, palet, ağırlık, hacim, yükleme tarihi. Okunamayan alanları (ayarda açıksa) yapay zekâ tamamlar; her alanın kaynağı işaretlenir (`ayrıştırıcı` / `yapay zekâ` / `kullanıcı`).
2. **WhatsApp paylaşımı:** Hazır mesaj şablonu üretilir ve tanımlı gruplara (ör. "Avrupa Parsiyel Yük", "Balkan Van Grubu", "Trakya Nakliyeciler") tek tıkla paylaşılır:
   ```
   🚚 *YÜK VAR* - YT-1088
   📍 İstanbul / Esenyurt ➜ Münih, DE
   📦 Kumaş topu · 4 palet · 950 kg
   📅 Yükleme: 27 Kasım Cuma
   Fiyat ve araç bilgisini DM atınız 🙏
   ```
3. **Nakliyeci teklifleri:** DM'den gelen fiyat mesajı yapıştırılır, fiyat + para birimi ayrıştırılır. "En uygun" teklif otomatik seçilir.
4. **Müşteri teklifi:** `Fiyat = yukarı_yuvarla_10( en düşük nakliyeci fiyatı × (1 + marj/100) )`. Varsayılan marj %15; marj <0 veya >100 engellenir, varsayılanın yarısının altına inerse uyarı. Teklif e-postası şablonu: güzergâh, yük, yükleme tarihi, EUR fiyat (KDV hariç), **3 iş günü geçerlilik**.
5. **Onay:** Müşteri onayıyla talep `SF-xxxxx` sefere dönüşür (`loadRef` ile bağlı).

Talep veri modeli: `id (YT-…), source (Outlook/Gmail/Elle), company, email, subject, body, receivedAt, status, from, to, cargo, weightKg, pallets, readyDate, quotes[{carrier, price, at, note}], marginPct, chosen, quotedAt, tripId`.

### 3.3 Seferler (`/shipments`)
**Aşamalar:** `Planlandı → Yüklendi → Gümrükte → Kapıda → Boşaltma → Teslim`
- Bir seferde yalnızca **bir aşama ileri/geri** taşınabilir; geri alma loglanır.
- **Plaka girilmeden "Yüklendi"ye geçilemez** ("Önce nakliyeciden plakayı girin"). Plaka formatı 5–12 harf/rakam.
- Her değişiklik `events[]` zaman çizelgesine kim/ne zaman/ne olarak yazılır.
- Görünümler: Tümü, Aktif, Gümrük/kapıda, Eksik CMR, Faturası kesilmemiş; aşama filtresi; arama (sefer, firma, plaka, şehir, sınır kapısı).
- Sefer detay: müşteri, nakliyeci, yükleme/teslim adresi, güzergâh, sınır kapısı, gümrükçü, plaka, şoför, araç, yük, yükleme tarihi, ETA, GPS konumu + harita, CMR durumu (`Bekleniyor / Sisteme girilmedi / Ulaştı`), satış ve maliyet (EUR), fatura durumu.
- Müşteriye tek tık iletişim: **Plakayı gönder, Konumu gönder, Fatura ve CMR gönder**; CMR fotoğrafı yükleme; EDM satış faturası.
- **Toplu sefer açma:** Excel'den yapıştır (Ref, Müşteri, Yükleme, Varış, Plaka, Nakliyeci, Tarih) – mevcut ref'ler atlanır.

### 3.4 Firmalar (`/companies`)
Sekmeler ve kolonlar:
| Sekme | Kolonlar |
|---|---|
| Müşteriler | Firma, Şehir, Yetkili, Sefer (yıl), Bakiye |
| Nakliyeciler | Firma, Şehir, Araç sayısı, Sefer (yıl), Zamanında %, Bakiye |
| Müşteri Adayları | Firma, Şehir, Kaynak (CMR alıcı/gönderici, Açık kaynak, Yük talebi), İlgili hat (TR→DE…), Durum (Aranacak/Görüşüldü/Teklif gönderildi) – Kanban; "Müşteri oldu"ya sürüklenince cari kart açılır |
| Gümrükçüler | Firma, Gümrük (Kapıkule, İpsala, Halkalı…), Telefon, Kaynak |
| Depolar | Antrepo/depo, bağlı gümrük müdürlüğü, adres |

Yarım kalan formlar taslak olarak saklanır.

### 3.5 Filo & Sürücüler (`/fleet`)
- **Araç veri deposu:** plaka, model, tip (Panelvan/Hafif ticari/Kamyonet/Frigo van), sahip (nakliyeci), sürücü, km, aylık yakıt maliyeti.
- **Belge takvimi:** Muayene (30 gün önce uyar), Kasko (30), Yeşil sigorta (15), Bakım (15) – durum: `Güncel / 30 gün içinde / Süresi geçmiş`. Belge türleri ayarlardan genişletilebilir.
- **Sürücü–araç ataması** (bir sürücü tek araçta).
- **Şoför hesabı:** sefer harcamaları + **harcırah** (gün × ₺/gün, seferin ilk günündeki değerle sabitlenir), CSV dışa aktarma.
- **Fiş okuma (OCR/ayrıştırıcı):** firma/istasyon (Shell, Opet, PO, BP, Total, Aytemiz, Lukoil, OMV, MOL, Aral, Esso, Eni, Petrom, Rompetrol, KGM, Osmangazi, Avrasya, ASFINAG, Nemzeti Útdíj, BG Toll, İSPARK…), tarih, tutar, para birimi, litre, plaka; tür (Akaryakıt/Otoyol/Vinyet/Otopark/Yemek…). Döviz fişi **fiş tarihindeki kurla** TL'ye çevrilir.
- **Yakıt kartı ekstresi:** DKV / RMC hareketleri plaka bazlı araç carisine düşer.

### 3.6 Finans & Cari (`/finance`) – EDM merkezli
Sekmeler: **Giden faturalar** (müşteri), **Gelen faturalar** (nakliyeci), **Tahsilat/Ödeme**, **Cari & Mutabakat**, **Banka**.
- EDM'den yeni faturaları çek; bilinmeyen firma için otomatik **cari kart açılır**.
- Fatura: no, tür (Satış/Alış), cari, sefer ref, tutar, düzenleme, vade, durum (`Bekliyor / Gecikti / Ödendi`), hatırlatma tarihi. Faturalar sefer numarasıyla eşleşir.
- "Tahsil edildi / Ödendi işaretle" → cariye otomatik işlenir.
- **Mutabakat:** cari ekstre + şablonlu mutabakat mektubu (e-posta, ekstre PDF eki), durum `Mutabık / Bekliyor / Fark var`, gönderilen mektup geçmişi.
- **Banka:** hareketler elle girilir; cari veya gider hesabına (Banka masrafları, Vergi ve SGK, Genel giderler) eşleştirilir; geri alınabilir.
- Döviz: EUR/USD/GBP kuru ve günlük değişim; ana para birimi EUR, vade 30 gün.

### 3.7 Raporlar (`/reports`)
- Kartlar: en çok çalışılan hat, en kârlı hat, en sık çalışılan firma, en sık çalışılan nakliyeci.
- **Hat kârlılığı (aylık, EUR)** – hatta tıklayınca seferleri listeler.
- **Nakliyeci ve teslimat performansı:** sefer sayısı, zamanında teslim %, ortalama gecikme (saat).
- Günlük operasyon raporu, finansal rapor (borç/alacak/kâr).

### 3.8 Araçlar
- **Yük Hesaplama (`/tools/load`):** ağırlık, hacim (m³), euro palet, en uzun parça → uygun aracı önerir; doluluk % ve "sığmaz" gerekçeleri. Araç kataloğu:
  | Araç | İç ölçü (U×G×Y m) | Faydalı yük | Euro palet |
  |---|---|---|---|
  | Fiat Doblò Maxi | 2,17×1,71×1,25 | 1.000 kg | 2 |
  | Fiat Ducato L4H2 | 4,07×1,87×1,93 | 1.400 kg | 4 |
  | VW Crafter L4H3 | 4,30×1,83×2,19 | 1.250 kg | 4 |
  | Iveco Daily 35S 18m³ | 4,68×1,80×2,10 | 1.100 kg | 4–5 |
- **Rota Planlama (`/tools/planner`):** Nominatim (adres→koordinat) + OSRM (km/güzergâh). Ara noktalar, harita, **güzergâhtaki sınır kapıları** (Kapıkule, Dereköy, İpsala, Horgoš-Röszke…), maliyet = yakıt (km × L/100 × ₺/L) + otoyol (km × ₺/km). **Sürüş/dinlenme simülasyonu:** 4,5 saat sürüşte 45 dk mola, 9 saat sürüşte 11 saat günlük dinlenme → varış tahmini.

### 3.9 Bildirimler ve Asistan
- Bildirimler: Finans, Yük talepleri, Araç belgeleri türlerinde; okundu/okunmadı.
- Asistan (Ctrl+J): kural tabanlı soru-cevap – "Geciken ödemeler", "Bu hafta ne tahsil edeceğiz?", "Gümrükte bekleyen sefer var mı?", "Belgesi yaklaşan araçlar", "En kârlı hat", "Euro kaç?"; sefer no/plaka/firma yazınca kayıt özeti. Rol yetkisine uyar.

### 3.10 Ayarlar & Bakım (`/settings`, yalnız Yönetici)
- **Genel tanımlamalar (tarihli/versiyonlu):** varsayılan marj %15, vade 30 gün, GPS bildirim saati 09:00, harcırah 2.500 ₺/gün, motorin 52 ₺/L, tüketim 10,5 L/100 km, otoyol 1,8 ₺/km, ortalama hız 75 km/sa, EUR 45,82 ₺, HUF 0,1165 ₺. Her değişiklik **"şu tarihten itibaren geçerli"** olarak loglanır; eski kayıtlar kendi tarihinin değeriyle kalır, geçmişe dönülebilir.
- **Listeler:** araç tipleri, belge tipleri (+uyarı günü), gider hesapları, araç kataloğu, **ülke bazlı otoyol/vinyet tarifeleri** (TR KGM 1,8 ₺/km; BG e-vinyet haftalık 8 €; GR Egnatia 20 €; RO Rovinieta 7 gün 6 €; RS gişe 30 €; HU e-matrica 6.400 HUF; AT 10 gün 12,4 €; SI 7 gün 16 €).
- **Entegrasyonlar:** EDM e-Fatura, Uyumsoft ERP (ops.), DKV Mobility REST API, RMC (dosya/RPA), Outlook/Gmail (OAuth), WhatsApp Business (uygulama, API'siz), GPS (Arvento, Mobiliz…), OSRM, Nominatim; bağlantı testi.
- **Ayrıştırıcılar:** E-posta yük talebi, DM fiyat teklifi, Fiş; "Yapay zekâ eksikleri tamamlıyor" anahtarı.
- WhatsApp grupları, kullanıcılar, yedekler.

---

## 3A. Detay: WhatsApp talep paylaşımı ve nakliyeci fiyatlarının toplanması

> Özet: Fiyatlar **otomatik toplanmıyor**. Nakliyeciden WhatsApp DM ile gelen her mesajı operasyon personeli panele **elle yapıştırıyor**; panel metinden fiyatı ve notu ayrıştırıp talebe ekliyor.

### Adım 1 – Talep gelir (durum: `Yeni`)
- Müşteri e-postası (Outlook/Gmail) okunur ya da "Yeni talep" ile metin yapıştırılır.
- Ayrıştırıcı şunları çıkarır: müşteri, yükleme yeri, teslim yeri, yük cinsi, palet, kg, yükleme tarihi ("cuma", "yarın", "27 kasım", "27.11" gibi ifadeler dahil).

### Adım 2 – WhatsApp gruplarında paylaşım (durum: `Gruplarda`)
- "Gruplarda paylaş" butonu hazır ilan metnini üretir:
  ```
  🚚 *YÜK VAR* - YT-1088
  📍 İstanbul / Esenyurt ➜ Münih, DE
  📦 Kumaş topu · 4 palet · 950 kg
  📅 Yükleme: 27 Kasım Cuma
  Fiyat ve araç bilgisini DM atınız 🙏
  ```
- Buton yalnızca `https://wa.me/?text=...` linkini açar. **Grubu kullanıcı kendi WhatsApp'ında seçer**; API yok, gruplara otomatik gönderim yok. Ayarlardaki grup listesi ("Avrupa Parsiyel Yük", "Balkan Van Grubu", "Trakya Nakliyeciler") yalnızca bilgi amaçlı gösteriliyor.
- Talep durumu `Gruplarda` olur, paylaşım zamanı (`sharedAt`) kaydedilir.

### Adım 3 – Nakliyeci fiyatının eklenmesi ("DM fiyatı ekle" penceresi)
Nakliyeci özelden yazar, ör. *"Abi 2.100 EUR crafter cuma"*. Operasyon personeli:
1. **Nakliyeciyi seçer** (Firmalar → Nakliyeciler listesinden).
2. Mesajı **"Gelen mesaj"** kutusuna yapıştırır.
3. Ayrıştırıcı alanları otomatik doldurur:

| Alan | Kural |
|---|---|
| Fiyat | "2.100 EUR", "€2100", "2,1 bin euro", "1950 TL", "$1800" gibi biçimleri tanır; binlik ayırıcı nokta/boşluk, ondalık virgül, "bin" çarpanı desteklenir. Para birimi yazılmamışsa kg/ton/palet/adet/gün/saat olmayan ve **100'den büyük ilk sayı** fiyat kabul edilir. |
| Para birimi | `€ / eur / euro / avro / eu` → EUR · `₺ / tl / try / lira` → TRY · `$ / usd / dolar` → USD |
| Not | Fiyat ifadesi çıkarılır; "merhaba, selam, abi, hocam, kardeşim, fiyatımız, olur, veririz, yaparız, teklifimiz, net, son, +kdv, dahil, hariç, teşekkürler, kolay gelsin, iyi çalışmalar" gibi kalıplar silinir; kalan metin not olur (örnekte: "Crafter cuma"). |
| Kaynak etiketi | Her alanın yanında `ayrıştırıcı` / `yapay zekâ` / `kullanıcı` gösterilir. Ayarlarda yapay zekâ açıksa okunamayan alanları o tamamlar. |

4. Para birimi EUR değilse uyarı: *"Teklifler EUR ile karşılaştırılır. Tutarı EUR'ya çevirip girin ve para birimini EUR seçin."* → **Kur dönüşümü elle yapılıyor.**
5. Kayıt: `quotes[] += {carrier, price, at, note}`
   - **Para birimi kaydedilmiyor** (tüm fiyatlar EUR varsayılıyor).
   - **Aynı nakliyeci yeniden fiyat verirse eski teklif siliniyor** ("Bu nakliyecinin önceki teklifi bununla değişir") → pazarlık/fiyat geçmişi tutulmuyor.
   - Talep hâlâ `Yeni` ise otomatik `Gruplarda` durumuna geçer.
   - Mesajın ham metni teklif kaydına yazılmıyor.

### Adım 4 – Karşılaştırma ve müşteri teklifi (durum: `Teklif verildi`)
- Teklifler ucuzdan pahalıya sıralanır; **en düşük fiyat "En uygun" etiketiyle otomatik seçilir**, istenirse başka nakliyeci seçilebilir.
- Müşteri fiyatı: **seçilen nakliyeci fiyatı × (1 + marj%) → 10'a yukarı yuvarla**
  - Örnek: 1.890 € × 1,15 = 2.173,5 → **2.180 €**
  - Varsayılan marj %15; marj <%0 veya >%100 engellenir; varsayılanın yarısının (%7,5) altı uyarı verir.
- Teklif e-postası şablondan gelir (güzergâh, yük, yükleme tarihi, EUR fiyat, KDV hariç, 3 iş günü geçerli), gönderimden önce düzenlenebilir. Marj, seçilen nakliyeci ve teklif zamanı kaydedilir.

### Adım 5 – Onay (durum: `Onaylandı`)
- "Onaylandı, sefer aç" → talep `SF-xxxxx` sefere dönüşür; seferde `loadRef` olarak talep numarası, adresler, yük, satış ve maliyet taşınır.

### Değerlendirme ve ENG TMS önerisi
| | |
|---|---|
| **Güçlü yan** | WhatsApp API / Meta onayı / ücret gerektirmez; mevcut çalışma alışkanlığı bozulmaz, sadece kopyala-yapıştır + otomatik fiyat okuma. |
| **Zayıf yan** | Toplama elle; fiyat geçmişi üzerine yazılıyor; para birimi ve kur saklanmıyor; ham mesaj saklanmıyor; teklif sayısı/yanıt süresi gibi nakliyeci metrikleri çıkarılamıyor. |
| **TMS'te olması gereken** | Her teklif ayrı satır: `carrier_quote(id, load_request_id, carrier_id, price, currency, fx_rate, price_eur, raw_message, note, vehicle, available_date, received_at, entered_by, source[parser/ai/user], is_selected)`. Böylece hat bazlı fiyat benchmark'ı, nakliyeci yanıt süresi ve kazanma oranı raporlanabilir. |
| **İleri adım (opsiyonel)** | Otomatik toplama isteniyorsa WhatsApp Business API veya grup/DM mesajlarını okuyan bir bot entegrasyonu ayrıca değerlendirilmeli. |

---

## 4. ENG TMS'e aktarılacak değerli alanlar

> Not: Bu oturumdaki `lojistikbot` deposu boş olduğu için ENG TMS'in mevcut veri şemasıyla birebir karşılaştırma yapılamadı. Aşağıdaki liste "TMS'te yoksa eklenmeli" önerisidir; mevcut şemayı paylaşırsanız boşluk analizini netleştirebilirim.

### 4.1 Öncelik 1 – Doğrudan değer üreten (hemen aktarılmalı)
| # | Alan / Özellik | Neden değerli | Önerilen TMS alanları |
|---|---|---|---|
| 1 | **Yük talebi → teklif → sefer hunisi** | Satış dönüşümünü ölçülebilir yapar, talep-sefer izlenebilirliği | `load_request(id, source, customer_id, email, subject, raw_body, received_at, status, origin, destination, cargo, weight_kg, volume_m3, pallets, ready_date, margin_pct, chosen_carrier_id, quoted_at, quoted_price, trip_id)` |
| 2 | **Nakliyeci teklifleri tablosu** | Fiyat geçmişi = hat bazlı maliyet benchmark'ı, en uygun nakliyeci seçimi | `carrier_quote(load_request_id, carrier_id, price, currency, note, received_at, channel)` |
| 3 | **Teklif fiyat formülü** | Standart, denetlenebilir fiyatlama | `ceil(min_quote × (1+marj)/10)×10`, marj alt-sınır uyarısı, teklif geçerlilik 3 iş günü |
| 4 | **Sefer aşama makinesi + kurallar** | Operasyon disiplini (plakasız yükleme yok, tek adım ilerleme) | `trip.stage` enum (6 aşama), `trip_event(trip_id, at, by, text, kind)` |
| 5 | **Sefer ↔ fatura ↔ CMR bağı** | "Teslim edildi ama faturalanmadı / CMR yok" kaçağını kapatır | `trip.cmr_status`, `trip.invoiced`, `invoice.trip_ref` |
| 6 | **Sefer kârlılığı** | Hat/nakliyeci/müşteri bazında marj | `trip.sale_price`, `trip.cost`, `trip.currency` → hat kârlılığı raporu |
| 7 | **Bekleyen işler motoru** | Ekibe "bugün ne yapmalıyım" listesi | Kurallar: paylaşılmamış talep, teklif bekleyen talep, plakasız sefer, konum gönderilmemiş, CMR eksik, fatura kesilmemiş, belge bitiyor, mutabakat bekliyor |

### 4.2 Öncelik 2 – Operasyonel verim
| # | Alan / Özellik | Not |
|---|---|---|
| 8 | **WhatsApp yük ilanı şablonu + grup listesi** | Mevcut çalışma şekline birebir uyuyor; API gerekmez |
| 9 | **E-posta / DM / fiş ayrıştırıcı + alan kaynağı (`parser/ai/user`)** | Veri kalitesi denetimi için kaynağı saklamak önemli |
| 10 | **Müşteri iletişim şablonları** | Plaka bildirimi, günlük konum (09:00), fatura+CMR, navlun teklifi, mutabakat mektubu |
| 11 | **Sınır kapısı & gümrükçü & depo alanları** | `trip.border_crossing`, `trip.customs_broker_id`, gümrükçü/antrepo rehberi |
| 12 | **Araç kataloğu + yük sığma hesabı** | İç ölçü, faydalı yük, palet kapasitesi → otomatik araç önerisi |
| 13 | **Rota maliyeti + AETR sürüş/dinlenme simülasyonu** | 4,5 sa/45 dk, 9 sa/11 sa; yakıt + otoyol; ETA |
| 14 | **Ülke bazlı otoyol/vinyet tarifeleri** | TR, BG, GR, RO, RS, HU, AT, SI – fiyat + para birimi + aktif |

### 4.3 Öncelik 3 – Finans & filo
| # | Alan / Özellik | Not |
|---|---|---|
| 15 | **Araç belge takvimi** | Muayene, kasko, yeşil sigorta, bakım + tip bazlı uyarı günü |
| 16 | **Şoför hesabı + harcırah** | Harcırah oranı seferin başladığı günün değeriyle sabitlenir |
| 17 | **Yakıt kartı entegrasyonu (DKV API, RMC dosya)** | Plaka bazlı yakıt/otoyol maliyeti → sefer maliyetine bağlanabilir |
| 18 | **Cari mutabakat akışı** | Durum + mektup geçmişi + ekstre |
| 19 | **Banka hareketi eşleştirme** | Cari / gider hesabı, geri alınabilir |
| 20 | **Tarihli (versiyonlu) parametreler** | Kur, yakıt, harcırah, marj değişikliklerinin geçmiş kayıtları bozmaması – denetim için çok değerli |
| 21 | **Nakliyeci performansı** | Zamanında %, ortalama gecikme (saat), sefer sayısı |
| 22 | **Müşteri adayı kaynağı** | CMR'deki alıcı/gönderici firmalardan lead üretimi – satış için ek değer |

### 4.4 Aktarılmaması / dikkat edilmesi gerekenler
- **Veriler mock'tur** (Atlas Tekstil, Medipro, Yıldız Nakliyat, plakalar, tutarlar vb.); TMS'e veri olarak aktarılmamalı.
- **Güvenlik yok:** giriş, rol kontrolü ve entegrasyon şifreleri istemci tarafında / localStorage'da. Üretimde sunucu tarafı yetkilendirme ve gizli bilgi kasası şart.
- Kur, yakıt fiyatı, otoyol ücretleri sabit/elle girilmiş örnek değerler; TCMB/EPDK gibi kaynaklardan güncellenmeli.
- Sefer numaralandırma (`SF-`, `YT-`) istemcide "max+1" ile üretiliyor; çok kullanıcılı ortamda çakışır, sunucu sırası (sequence) kullanılmalı.

---

## 5. Önerilen sonraki adımlar
1. ENG TMS'in mevcut tablo/alan listesini paylaşın → Bölüm 4'ü "TMS'te var / yok / kısmen" olarak işaretleyen boşluk analizi.
2. Öncelik 1 maddeleri (talep–teklif–sefer hunisi, aşama makinesi, fatura/CMR bağı, kârlılık) için TMS veri modeli taslağı.
3. Prototipe gerçek ekran görüntüleriyle doğrulama: bunun için panelde oturum açma izni gerekli.
