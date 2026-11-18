# Kâtip — Hukuk Bürosu için Claude Tabanlı AI Sistemi

> Kaynak ilham: ReStructure AI "The Private AI System for Local Law Firms" (Instagram reel DedWJbVhS7Y).
> Orijinal: yerel Ollama (`gpt-oss:120b`) + açık kaynak araçlar. Bu tasarım aynı 8 departmanlı yapıyı
> **Claude** üzerine kurar ve **Türk hukuk bürosu** gerçeklerine (UYAP, HMK süreleri, AAÜT, KVKK, Av. K. m.36) uyarlar.
> Tarih: 2026-10-26 · Durum: tasarım, kod yok.

---

## 1. Tek cümlede

Büronun verisi ofiste kalır (arşiv, veritabanı, ses kaydı, OCR, embedding yerelde); yalnızca **akıl yürütme
gereken metin parçaları** AB bölgesindeki Claude'a gider; Claude dışarıya **hiçbir şeyi kendiliğinden göndermez** —
her e-posta, fatura, dilekçe ve takvim daveti bir avukatın onay kuyruğundan geçer.

## 2. Orijinal tasarımdan neden ve nerede ayrılıyoruz

| Konu | Videodaki (Ollama) | Kâtip (Claude) | Gerekçe |
|---|---|---|---|
| Model | `gpt-oss:120b` ofis sunucusunda | Claude Opus 5.5 / Sonnet 5.5 / Haiku 5.5, Amazon Bedrock **eu-central-1 (Frankfurt)** | Dilekçe kalitesi ve Türkçe hukuk dili; 80 GB+ GPU sunucusu (~15–30 bin $) yerine kullanım başı ödeme |
| Gizlilik | "Hiçbir şey ofisten çıkmaz" | Yerel veri katmanı + **Gizlilik Kalkanı** (TCKN/IBAN/isim maskeleme) + dosya hassasiyet seviyesi | KVKK m.9 yurt dışı aktarım ve Av. K. m.36 sır saklama |
| Orkestrasyon | n8n akışları | **Claude Agent SDK** (ofis sunucusunda) + departman alt-ajanları + MCP sunucuları; n8n yalnız olay tetikleyici | Ajan döngüsü, izin kancaları (hooks), alt-ajan, skill hazır |
| Bilgi katmanı | `firm-config.md`, `billing-rules.md`… | Aynı fikir → **Claude Skills** + prompt cache'li büro profili | Kural değişince kod değil markdown değişir |
| Faturalama | Saatlik (USD 380/saat) | AAÜT asgari ücret kontrolü + sabit/saatlik/başarı karması + e-SMM | Türkiye'de saatlik faturalama azınlık |
| Süre takibi | Docket takvimi | UYAP tebliğ evrakı → **deterministik süre hesaplayıcı** (HMK, adli tatil, resmi tatil) | LLM tarih hesaplamaz; süre kaçırma = mesleki sorumluluk |

**Bilinçli karar:** Model bulutta, veri yerelde. Tam yerel model isteyen büro için "L3 — çok gizli" seviyesi
tanımlı (bkz. §5); bu dosyalarda Claude'a hiçbir içerik gitmez.

## 3. Mimari (katmanlar)

```
┌──────────────────────────── 01 GİRİŞ KANALLARI ─────────────────────────────┐
│ Telefon (SIP: Verimor/Netgsm)  E-posta (M365/Gmail)  Web formu  UYAP evrak     │
│ Takvim (Cal.com)   Taranmış arşiv (tarayıcı → klasör)   Avukat sohbeti (panel) │
└───────────────┬──────────────────────────────────────────────────────────────┘
                ▼
┌──────────── 02 YEREL VERİ KATMANI (ofis sunucusu, Docker Compose) ───────────┐
│ Postgres 17 + pgvector + pg_trgm   Paperless-ngx (arşiv+OCR)   Nextcloud      │
│ faster-whisper large-v3 (Türkçe STT)   bge-m3 embedding   Kimai   Invoice/e-SMM│
│ Denetim kaydı (append-only, hash zinciri)   Vault (sırlar)   Authentik (SSO)   │
└───────────────┬──────────────────────────────────────────────────────────────┘
                ▼
┌──────────── 03 GİZLİLİK KALKANI ──────────────────────────────────────────────┐
│ Dosya seviyesi: L1 doğrudan · L2 takma ad (TCKN/IBAN/isim/plaka → ⟦KİŞİ_3⟧)    │
│ · L3 buluta hiç gitmez.  Dönüş yolunda takma adlar yerelde geri çözülür.       │
└───────────────┬──────────────────────────────────────────────────────────────┘
                ▼
┌──────────── 04 KÂTİP ÇEKİRDEĞİ (Claude Agent SDK, ofis sunucusunda) ─────────┐
│ Yönlendirici (Haiku 5.5) → 8 departman alt-ajanı → MCP araçları               │
│ Skills: büro profili, dilekçe üslubu, ücret politikası, süre kuralları, ton    │
│ Hooks: PreToolUse izin kapısı · PostToolUse denetim kaydı · zaman kaydı olayı │
│ Model çağrıları → Bedrock eu-central-1 (Opus 5.5 / Sonnet 5.5 / Haiku 5.5)     │
└───────────────┬──────────────────────────────────────────────────────────────┘
                ▼
┌──────────── 05 DEPARTMANLAR ─────────────────────────────────────────────────┐
│ Kabul & Resepsiyon │ Çatışma & Vekalet │ Süreler & Duruşma │ Araştırma & İçtihat│
│ Dilekçe & İnceleme │ E-posta & Müvekkil│ Zaman Kaydı       │ Faturalama & Emanet│
└───────────────┬──────────────────────────────────────────────────────────────┘
                ▼
┌──────────── 06 İNSAN ONAYI ──────────────────────────────────────────────────┐
│ Onay kuyruğu (panel + mobil bildirim): Avukat onayı · Stajyer/sekreter · Müvekkil│
│ Dışarı giden her eylem burada bekler.  Sabah 08:00 → tek sayfa brifing.        │
└──────────────────────────────────────────────────────────────────────────────┘
```

## 4. Sekiz departman

Her departman bir **alt-ajan**dır (kendi sistem istemi, kendi araç listesi, kendi model/efor ayarı).
Araçlar MCP sunucusu olarak yazılır; bir alt-ajan yalnız kendi departmanının araçlarını görür.

### 4.1 Kabul & Resepsiyon
- **Telefon:** SIP trunk (Verimor/Netgsm) → LiveKit Agents (yerel) → faster-whisper (Türkçe) →
  **Haiku 5.5** (düşük efor, kısa tur) → Piper/XTTS Türkçe ses. Mesai dışı aramaları karşılar.
- Görüşme başında KVKK aydınlatma + kayıt bildirimi (sabit metin, model üretmez).
- Yapılandırılmış kabul formu doldurur: ad, konu türü, karşı taraf, olay tarihi, aciliyet.
  `strict: true` araç şeması → kabul kaydı Postgres'e.
- **Yasak:** hukuki görüş, başarı/ücret vaadi (Av. K. m.55 reklam yasağı, TBB meslek kuralları).
  Bu kısıt sistem isteminde + çıktı filtresinde.
- Karşı taraf adı → otomatik çatışma kontrolü (4.2) → temizse Cal.com'da ön görüşme önerisi;
  avukata 07:30 bildirimi: "Yeni müvekkil · 20 Ekim 10:00 · trafik kazası · çatışma yok · notlar ekte".
- Web formu aynı akışa düşer.

### 4.2 Çatışma & Vekalet
- Postgres: `taraflar` (TCKN/VKN tam eşleşme + `pg_trgm` bulanık isim eşleşmesi), ilişkili şirketler.
- Haiku metinden tarafları çıkarır; **eşleşme kararı SQL'dedir**, model yalnız sonucu açıklar.
- Temiz ise: vekaletname taslağı + avukatlık ücret sözleşmesi (AAÜT asgari altı kontrolü — Av. K. m.164).

### 4.3 Süreler & Duruşma
- UYAP'ın açık API'si yok; otomatik giriş/kazıma yapılmaz. Avukat/sekreter UYAP'tan indirdiği evrakı
  `UYAP-gelen/` klasörüne bırakır veya e-postayla iletir.
- **Sonnet 5.5** evraktan çıkarır: mahkeme, esas no, evrak türü, tebliğ tarihi, istenen işlem.
- **Süre hesaplayıcı deterministik koddur** (LLM değil): HMK süreleri, istinaf/temyiz 2 hafta,
  adli tatil 20 Temmuz–31 Ağustos uzaması (HMK m.104), resmi/dini bayram tatilleri tablosu.
- Takvim: Nextcloud CalDAV; son günden 7/3/1 gün önce uyarı. Kritik süreler **iki kişi onayı** ister.

### 4.4 Araştırma & İçtihat
- Büro arşivi: Paperless-ngx OCR (eski taramalar) → parçalama → `bge-m3` embedding (yerel) →
  pgvector + Postgres `turkish` tam metin arama (hibrit).
- Dış kaynak: Yargıtay/Danıştay/AYM karar arama siteleri, lisanslıysa Lexpera/Kazancı entegrasyonu.
- **Opus 5.5**, sonuçlar `search_result` içerik bloğu olarak verilir, **citations açık**: her iddia
  kaynağa bağlı. Çıktıdaki her karar künyesi (daire, E., K., tarih) DB'de doğrulanır; doğrulanmayan
  künye kırmızı işaretlenir. Kural: **kaynaksız içtihat yok.**

### 4.5 Dilekçe & İnceleme
- **Opus 5.5, efor `high`**, akış (streaming). Girdi: dosya özeti + ilgili içtihat + büro şablonu.
- Şablonlar: `docxtpl` (Word) — büronun antetli kâğıdı, imza bloğu, UYAP'a uygun biçim.
- Skill `dilekce-uslubu`: büronun geçmiş 20 iyi dilekçesinden çıkarılmış üslup kuralları.
- Nextcloud + OnlyOffice'te izlenen değişikliklerle (redline) avukata sunulur; avukat onaylar,
  e-imzalar ve UYAP'a **kendisi** yükler. Kâtip UYAP'a hiçbir şey göndermez.

### 4.6 E-posta & Müvekkil İletişimi
- Microsoft Graph / Gmail API. Haiku gelen kutusunu sınıflandırır (dosya, aciliyet, cevap gerekir mi).
- **Sonnet 5.5** büro tonunda yanıt taslağı yazar → **Taslaklar** klasörüne koyar, gönderMEZ.
- Haftalık müvekkil durum özeti (dosyada ne oldu, sıradaki adım) — avukat onayıyla gider.

### 4.7 Zaman Kaydı
- n8n olayları (gönderilen e-posta, takvim toplantısı, onaylanan taslak, telefon notu) →
  Haiku dosyaya eşler ve süre önerir → Kimai'ye **taslak** kayıt.
- Akşam 18:00 avukata tek ekran: "Bugün 6 kayıt, 3,4 saat — onayla / düzelt".
- Videodaki gelir iddiasının kaynağı burası: kaçan faturalanabilir zaman. Saatlik çalışmayan bürolarda
  değer, dosya kârlılığı raporu ve iş yükü görünürlüğüdür.

### 4.8 Faturalama & Emanet
- Kimai saatleri + sabit ücretli dosyalar + masraf avansı → fatura taslağı.
- Haiku/Sonnet kontrol eder: AAÜT asgari altında mı, sözleşmedeki ücretle uyumlu mu, emanet
  bakiyesi yeterli mi.
- e-SMM (serbest meslek makbuzu) GİB entegratörü üzerinden (Paraşüt/Logo/Uyumsoft — büro hangisini
  kullanıyorsa). Onay kuyruğu → yönetici ortak imzalar → gönderim.

### Sabah Brifingi (her gün 08:00)
Tek sayfa: bugünkü duruşmalar, 7 gün içindeki süreler, gece gelen aramalar/kabuller, onay bekleyen
taslaklar, dünkü onaylanmamış zaman kayıtları, tahsil edilmemiş faturalar. Sonnet 5.5 üretir;
veriler SQL'den gelir.

## 5. Gizlilik ve hukuki uyum

| Seviye | Örnek dosyalar | Claude'a ne gider |
|---|---|---|
| **L1 Normal** | Ticari alacak, kira, iş hukuku | Metin olduğu gibi (AB bölgesi, eğitimde kullanılmaz) |
| **L2 Hassas** | Ceza, aile/boşanma, sağlık verisi içeren (KVKK m.6 özel nitelikli) | Takma adlı metin: `⟦KİŞİ_1⟧`, `⟦TCKN_1⟧`, `⟦IBAN_1⟧`; eşleme tablosu yerelde |
| **L3 Çok gizli** | Avukatın işaretlediği dosyalar | Hiçbir şey. Yalnız yerel arama + şablon doldurma |

- **Gizlilik Kalkanı:** Microsoft Presidio + Türkçe tanıyıcılar (TCKN algoritma kontrolü, `TR` IBAN,
  telefon, plaka, adres, isim NER). Claude'un yanıtı yerelde geri çözülür.
- **KVKK m.9:** Yurt dışı aktarım için sağlayıcıyla standart sözleşme; imzadan sonra 5 iş günü içinde
  Kurum'a bildirim. Müvekkil sözleşmesine ve aydınlatma metnine yapay zekâ işleme maddesi.
- **Av. K. m.36 sır saklama:** Sağlayıcı verileri model eğitiminde kullanmaz; saklama süresi sözleşmede.
  Bunu hukuk danışmanıyla teyit et (bu tasarım hukuki görüş değildir).
- **Denetim kaydı:** Her model çağrısı (hangi dosya, hangi seviye, kaç token, hangi kullanıcı) ve her
  araç eylemi append-only tabloya, satırlar hash zinciriyle.
- **Erişim:** Authentik SSO + rol: ortak, avukat, stajyer, sekreter. Alt-ajan, kullanıcının yetkisinin
  dışındaki dosyaya araç üzerinden erişemez (MCP sunucusu kullanıcı kimliğiyle sorgular).

## 6. Model yönlendirme

| İş | Model | Efor | Not |
|---|---|---|---|
| E-posta sınıflama, zaman eşleme, taraf çıkarma | Haiku 5.5 | low | Yüksek hacim |
| Telefon resepsiyonu | Haiku 5.5 | low | Gecikme kritik |
| E-posta taslağı, UYAP evrak çıkarımı, brifing | Sonnet 5.5 | medium | |
| İçtihat araştırması, dilekçe, sözleşme incelemesi | Opus 5.5 | high | Akış + citations |
| Gece arşiv indeksleme özetleri | Haiku 5.5, Batch API | — | %50 indirim (Batch'in Bedrock'ta varlığını doğrula) |

- Büro profili + skill'ler + araç tanımları **prompt cache** önekinde sabit tutulur.
- Opus 5.5 ve Sonnet 5.5'te reddetme (`stop_reason: refusal`) işlenir; Bedrock'ta SDK'nın
  istemci tarafı fallback ara katmanı kullanılır.

### Kaba maliyet tahmini (5 avukatlı büro, varsayım)

Günde ~200 e-posta sınıflama, 15 yanıt taslağı, 3 dilekçe, 5 araştırma, 10 telefon görüşmesi:
≈ **5 $/gün → ~100–150 $/ay** (Anthropic liste fiyatı; cache ile daha düşük; Bedrock fiyatı farklı
olabilir). Karşılaştırma: `gpt-oss:120b`'yi yerelde çalıştıracak sunucu 15–30 bin $.
Bu bir tahmindir; pilot ayında `usage` loglarından gerçek rakam çıkarılmalı.

## 7. Teknoloji yığını

- **Dil:** TypeScript (pnpm monorepo — kipay ile aynı alışkanlık).
- `apps/kernel` — Claude Agent SDK çekirdeği + alt-ajanlar + hooks.
- `apps/panel` — Next.js: onay kuyruğu, sabah brifingi, sohbet, dosya ekranı.
- `apps/voice` — LiveKit Agents + faster-whisper + TTS (Python servis).
- `mcp/` — `mcp-dosyalar`, `mcp-arsiv`, `mcp-cakisma`, `mcp-sureler`, `mcp-eposta`, `mcp-kimai`,
  `mcp-fatura`, `mcp-takvim`.
- `packages/db` — Drizzle + Postgres 17 + pgvector.
- `packages/kalkan` — Gizlilik Kalkanı (Presidio, Python yan servis).
- `skills/` — `buro-profili`, `dilekce-uslubu`, `ucret-politikasi`, `sure-kurallari`, `iletisim-tonu`.
- **Dağıtım:** Ofis sunucusunda Docker Compose (veri ofiste). Sunucu: 64 GB RAM, 2 TB NVMe (RAID1),
  Whisper + embedding için 16 GB'lık tek GPU yeterli. Gece şifreli yedek (restic → büronun kendi
  depolaması). Railway yalnız herkese açık web formu için, isteğe bağlı.
- **Ağ:** Dışarıya yalnız Bedrock uç noktası + e-posta/takvim API'leri; panel Tailscale arkasında.

## 8. Yol haritası

| Faz | Süre | Kapsam | Neden bu sırada |
|---|---|---|---|
| 0 | 1 hafta | Sunucu, Postgres, SSO, denetim kaydı, Kalkan, KVKK belgeleri, skill'lerin ilk hali | Temel |
| 1 | 2–3 hafta | E-posta taslakları + zaman kaydı + sabah brifingi | En hızlı görünür fayda, düşük risk |
| 2 | 3 hafta | Arşiv indeksleme + araştırma + dilekçe | Asıl kalite farkı |
| 3 | 2 hafta | UYAP evrak çıkarımı + süre hesaplayıcı + çatışma | Yüksek risk → kapsamlı test (her süre kuralı için birim test) |
| 4 | 2 hafta | Faturalama + e-SMM + emanet | Muhasebe entegrasyonu büroya özel |
| 5 | 3 hafta | Telefon resepsiyonisti | En karmaşık ve en görünür; en son |

**Pilot başarı ölçütleri:** taslak kabul oranı (%70+ küçük düzeltmeyle), günlük kaydedilen
faturalanabilir saat artışı, kaçırılan süre = 0, avukat başına haftalık kazanılan saat.

## 9. Açık sorular

1. Hedef büro var mı (kuzen/tanıdık pilotu)? Kaç avukat, hangi alanlar, saatlik mi sabit ücret mi?
2. E-posta: Microsoft 365 mi Gmail mi? Muhasebe: hangi e-SMM entegratörü?
3. Claude erişim yolu: Bedrock Frankfurt (AB veri yerleşimi) mi, Anthropic API (ZDR sözleşmesiyle) mi?
   Opus 5.5'in eu-central-1'de bulunduğu doğrulanmalı.
4. L3 dosyalar için küçük bir yerel model de istenir mi (ör. özetleme), yoksa "AI yok" yeterli mi?
5. Ürün mü (çok büroya satış, SaaS-benzeri kurulum paketi) yoksa tek büroya özel kurulum mu?
