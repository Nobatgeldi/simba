# Simba — Teknik Altyapı Mimarisi (v0.1)

> Görsel versiyon: [`docs/architecture.html`](architecture.html) · Öğrenci arayüzü demosu: [`demo/index.html`](../demo/index.html)

Simba; oyunlaştırılmış dil öğrenme (Duolingo tarzı), canlı öğretmenle birebir ders (Cambly tarzı) ve içerik üreticilerin kurs satıp para kazandığı bir pazar yerini tek platformda birleştirir. Hedef diller ilk aşamada **İngilizce, Çince, Rusça, Fransızca**.

**Hedef pazar:** Bağımsız Devletler Topluluğu (BDT) ülkeleri, ilk sırada **Türkmenistan**. Arayüz ve kaynak diller: **Türkmence, Rusça, Türkçe**. Bu pazarın ödeme, ağ ve hukuk koşulları ayrıca [Bölüm 6](#6-hedef-pazar-türkmenistan-ve-bdt)'da ele alınıyor.

---

## 1. Ürünler ve kullanıcılar

| Uygulama | Kim kullanır | Platform | Ana işlevler |
|---|---|---|---|
| **Simba** (öğrenci) | Öğrenciler | iOS, Android, Web | Ders yolu, egzersizler, tekrar (hatalar), lig/görev/seri, öğretmenle ders, creator kursları, Premium |
| **Simba Teach** | Öğretmenler | iOS, Android, Web | Başvuru & profil, uygunluk takvimi, rezervasyonlar, “Şimdi Görüş” kuyruğu, görüntülü ders, ders notu, kazanç & ödeme |
| **Simba Studio** | İçerik üreticiler + iç içerik ekibi | Web | Kurs/ünite/ders/egzersiz editörü, medya yükleme, önizleme, inceleme süreci, satış & gelir paneli |
| **Simba Admin** | Admin, destek, moderatör, finans | Web (ayrı domain) | Kullanıcı yönetimi, destek talepleri, moderasyon, içerik onayı, öğretmen onayı, ödemeler/iadeler, raporlar, sistem ayarları |

Öğrenci ve öğretmen mobil uygulamaları **mağazada iki ayrı uygulama** olarak yayınlanır (Cambly / Cambly Tutor modeli): bildirimler, izinler ve mağaza incelemeleri birbirine karışmaz. İkisi de aynı paylaşılan paketlerden derlenir.

---

## 2. Temel kararlar

1. **Uçtan uca TypeScript.** Web, mobil ve backend aynı dili konuşur; API tipleri, egzersiz değerlendirme mantığı ve tasarım token’ları paylaşılır. Küçük ekip için en büyük hız kazancı budur.
2. **Monorepo** — Turborepo + pnpm. Altı uygulama + ortak paketler tek repoda.
3. **Mobil: React Native + Expo** (EAS Build, EAS Update ile OTA güncelleme). Ses kaydı, push, offline ders paketleri, kamera/mikrofon (görüntülü ders) Expo ekosisteminde hazır.
4. **Web: Next.js (App Router).** Öğrenci web’i aynı zamanda SEO’lu pazarlama sitesi. Teach, Studio ve Admin ayrı Next.js uygulamaları — ayrı domain, ayrı güvenlik politikası, ayrı deploy.
5. **Backend: NestJS modüler monolit.** Başta tek deploy; modül sınırları ve veritabanı şemaları ayrı tutulur. Yükü ayrışan parçalar (realtime, medya işleme, analitik) sonradan ayrı servise çıkarılır. Mikroservisle başlamak bu ölçekte maliyetli ve yavaştır.
6. **Veri:** PostgreSQL (ana kaynak), Redis (cache, lig sıralamaları, rate-limit, kuyruk), S3 uyumlu obje depolama + CDN (ses, görsel, video).
7. **Görüntülü ders:** WebRTC medya sunucusu yazılmaz — **LiveKit** (Cloud ile başla, maliyet büyürse self-host). Kayıt için LiveKit Egress.
8. **Ses teknolojileri:** **Azure AI Speech** — TTS (doğal sesler), STT ve telaffuz değerlendirme; dört hedef dili de destekler.
9. **Yapay zekâ:** LLM (ör. Claude) — AI sohbet partneri, “neden yanlış?” açıklamaları, serbest cevap değerlendirme, Studio’da egzersiz üretim asistanı.
10. **Ödeme:** Ülkeye göre takılabilen ödeme sağlayıcıları. Mağazaların çalıştığı yerlerde App Store / Google Play IAP (RevenueCat ile tek API), web’de iyzico (TR) + Stripe (global). Türkmenistan için **aktivasyon kodu** ve yerel ödeme ortağı. Öğretmen ve creator kazançları iç **muhasebe defterinde (ledger)** tutulur, dönemsel payout yapılır.
11. **Çok dilli arayüz ilk günden:** Türkmence, Rusça, Türkçe. Hiçbir metin koda gömülmez; ICU MessageFormat ile çeviri dosyaları, çeviri yönetimi için Tolgee.
12. **Kısıtlı ağa göre tasarım:** Tüm trafik kendi alan adlarımızdan geçer, offline ders paketleri, düşük bant genişliğinde çalışan ses/video.

---

## 3. Sistem diyagramı

```mermaid
flowchart TB
  subgraph Clients[İstemciler]
    LM[Simba Mobil<br/>Expo]
    LW[Simba Web<br/>Next.js]
    TM[Simba Teach<br/>Mobil + Web]
    ST[Simba Studio<br/>Web]
    AD[Simba Admin<br/>Web]
  end

  CF[Cloudflare<br/>DNS · WAF · CDN]
  Clients --> CF --> ALB[Load Balancer]

  subgraph API[NestJS Modüler Monolit]
    ID[identity]
    US[users]
    CAT[catalog]
    LRN[learning]
    GAM[gamification]
    TUT[tutoring]
    MKT[marketplace]
    BIL[billing / ledger]
    NOT[notifications]
    SUP[support & moderation]
    RT[realtime WS]
  end
  ALB --> API
  API --> Q[(BullMQ<br/>Redis kuyruk)] --> W[Worker'lar]

  subgraph Data[Veri]
    PG[(PostgreSQL)]
    RD[(Redis)]
    S3[(S3 + CDN)]
    MS[(Meilisearch)]
    AN[(PostHog / ClickHouse)]
  end
  API --> Data
  W --> Data

  subgraph Ext[Harici servisler]
    LK[LiveKit<br/>görüntülü ders]
    AZ[Azure Speech<br/>TTS · STT · telaffuz]
    LLM[LLM API]
    PAY[RevenueCat · iyzico · Stripe<br/>yerel ödeme ortakları]
    PUSH[FCM / APNs · E-posta]
    MUX[Mux<br/>video]
  end
  API --> Ext
  W --> Ext
```

---

## 4. Backend modülleri

| Modül | Sorumluluk | Notlar |
|---|---|---|
| `identity` | Kayıt/giriş, Google & Apple ile giriş, JWT (kısa ömürlü access + dönen refresh), RBAC | Roller: `learner`, `teacher`, `creator`, `support`, `content_editor`, `finance`, `admin`. Admin rollerinde 2FA zorunlu. |
| `users` | Profil, takip/arkadaş, ayarlar, ülke/saat dilimi | KVKK/GDPR: veri dışa aktarma ve hesap silme uçları. |
| `catalog` | Kurs → Ünite → Seviye → Ders → Egzersiz; medya | Kurs = (kaynak dil, hedef dil) çifti. Egzersiz `type + payload(jsonb)`, JSON Schema ile doğrulanır, versiyonlanır. Taslak → İnceleme → Yayın. |
| `learning` | Ders oturumu, cevap değerlendirme, ilerleme, can/seri/XP | Aralıklı tekrar **FSRS** algoritmasıyla (“Ekstra Çalışma: hatalı cümlelerini tekrar et”). Offline cevaplar idempotent event olarak senkronlanır. |
| `gamification` | Haftalık ligler, günlük görevler, rozetler, elmas ekonomisi | Lig grupları ~30 kişi, Redis sorted set; hafta sonu terfi/düşme worker’la. |
| `tutoring` | Öğretmen başvurusu & onayı, uygunluk takvimi, rezervasyon, anlık eşleşme, ders oturumu, değerlendirme | Tüm saatler UTC saklanır, gösterimde kullanıcı saat dilimi. LiveKit oda token’ını bu modül üretir. İptal/gelmeme kuralları. |
| `marketplace` | Creator kursları, fiyat, erişim (entitlement), yorum/puan | Creator içeriği aynı `catalog` modelini kullanır, `owner_type = creator`. |
| `billing` | Abonelik, ders kredisi paketleri, kurs satışı, iade, payout | Store/RevenueCat/iyzico/Stripe webhook’ları. **Çift kayıtlı ledger**; bakiye hiçbir zaman tek sütunda tutulmaz. TR e-Arşiv fatura entegrasyonu. |
| `notifications` | Push, e-posta, uygulama içi bildirim | Seri hatırlatma kullanıcının yerel saatine göre zamanlanır. |
| `support` | Destek talepleri, uygulama içi destek sohbeti, kullanıcıyı salt-okunur görüntüleme | Admin’in yaptığı her işlem `audit_log`’a yazılır. |
| `moderation` | Kullanıcı/içerik şikayetleri, ders sohbeti raporları, otomatik filtre | Creator içerik incelemesi buradan geçer. |
| `realtime` | WebSocket gateway: sohbet, öğretmen çevrimiçi durumu, canlı bildirim | Yatay ölçek için Redis adapter; ileride ayrı servis. |
| `analytics` | Ürün event’leri, A/B test, tutunma (retention) | PostHog (ürün analitiği + feature flag), ileride ClickHouse/BigQuery. |

API: **REST + OpenAPI**; istemci SDK’sı OpenAPI’den otomatik üretilir (`packages/api-client`). Gerçek zamanlı ihtiyaçlar WebSocket üzerinden.

---

## 5. Dile özel gereksinimler

- **Çince:** Hanzi, pinyin ve ton ayrı alanlarda saklanır. Karakter yazma egzersizi (Hanzi Writer + stroke-order verisi), ton telaffuz puanlaması, cevap kontrolünde kelime bölme.
- **Rusça:** Kiril ekran klavyesi, transliterasyon ipucu, vurgu işaretleri (ударение), hâl çekimi egzersizleri.
- **Fransızca:** Aksan toleransı (é ↔ e) ayarlanabilir, liaison içeren sesler.
- **İngilizce:** US / UK aksan seçimi.
- **Genel:** Cevap normalizasyonu (Unicode NFC, noktalama, büyük/küçük harf), birden fazla doğru cevap, Levenshtein ile yazım toleransı.
- **Arayüz dilleri:** Türkmence, Rusça, Türkçe (ayrıntılar Bölüm 6.1).

---

## 6. Hedef pazar: Türkmenistan ve BDT

İlk pazar Türkmenistan; ardından Özbekistan, Kazakistan, Kırgızistan, Tacikistan, Azerbaycan gibi BDT ülkeleri. Bu pazar ürünü ve altyapıyı aşağıdaki noktalarda şekillendirir. “Doğrulanmalı” diye işaretli maddeler varsayımdır; lansmandan önce yerel ortak ve hukukçu ile teyit edilmeli.

### 6.1 Arayüz dilleri

| Dil | Kod | Neden | Teknik not |
|---|---|---|---|
| Türkmence | `tk` | Türkmenistan’da varsayılan dil | Latin alfabesi: ä ç ž ň ö ş ü ý. Tarayıcı ve mobil Intl verisinde çoğu zaman yok |
| Rusça | `ru` | BDT’nin ortak dili, Türkmenistan’da da yaygın | Kiril; çoğul 3 biçimli (1 урок, 2 урока, 5 уроков) |
| Türkçe | `tr` | Türkiye pazarı ve ekibin dili | — |

Sonraki adaylar: Özbekçe (`uz`, Latin), Kazakça (`kk`), Azerice (`az`), İngilizce (`en`).

- **Altyapı:** `packages/i18n` içinde anahtar + çeviri dosyaları, ICU MessageFormat (FormatJS). Web, mobil, push bildirimi ve e-posta aynı dosyaları kullanır.
- **Çeviri yönetimi:** Tolgee (açık kaynak, self-host). Çevirmenler bağlamı ekran görüntüsüyle görür; CI eksik anahtarları yakalar. Her metin anadili konuşan biri tarafından onaylanmadan yayına girmez.
- **Dil seçimi:** İlk açılışta cihaz dili, sonra kullanıcının tercihi (profilde saklanır, bildirim ve e-postalar da bu dilde gider).
- **Türkmence Intl eksikleri:** Gün/ay adları ve sayı biçimi için kendi yerelleştirme tablomuz olmalı (demoda uygulandı). React Native (Hermes) için FormatJS polyfill’leri.
- **Font:** Latin Extended + Kiril kapsayan font (Nunito uygun); Çince için sistem fontuna yedek.
- **Mağaza:** App Store / Google Play açıklamaları ve ekran görüntüleri üç dilde.

### 6.2 Kurs matrisi: kaynak dil × hedef dil

Arayüz dili aynı zamanda öğrenmenin kaynak dilidir: Türkmence konuşan biri İngilizceyi Türkmence açıklamalarla öğrenir. 3 kaynak × 4 hedef = 11 kurs (Rusça → Rusça hariç). Her kursu baştan yazmamak için:

- Egzersizin **hedef dil** kısmı (cümle, ses kaydı, doğru cevaplar) bir kez üretilir.
- **Kaynak dile** bağlı alanlar (talimat, çeviri, ipucu, kabul edilen çeviri cevapları) `exercise_translations` tablosunda dil başına tutulur.
- Studio’da LLM taslak çeviri üretir, anadili konuşan editör onaylar. Yeni bir kaynak dil eklemek, mevcut içeriği çevirmek demektir.
- **Lansman önceliği:** tk→en, ru→en; ardından tk→ru (Türkmenistan’da yüksek talep beklenir), sonra Çince ve Fransızca.
- **Öğretmenler:** Profilde “konuştuğu diller” alanı ve “ana dilimi bilen öğretmen” filtresi. Yeni başlayanlar için Türkmence/Rusça bilen yerel öğretmen havuzu hem daha uygun fiyatlı hem daha erişilebilir.

### 6.3 Ödeme

Ödeme, `billing` modülünde ülkeye göre seçilen `PaymentProvider` adaptörleriyle kurulur; ülke → sağlayıcı eşlemesi ve ülke bazlı fiyat tablosu Admin panelinden yönetilir.

| Ülke | Yaklaşım | Durum |
|---|---|---|
| Türkmenistan | **Aktivasyon kodu** (Premium ve ders kredisi): yerel bayi/ortaklar satar, kullanıcı uygulamada girer. Paralel olarak manat ile yerel banka/ödeme ortağı entegrasyonu | Yerel kartlarla uluslararası online ödeme çok kısıtlı; doğrulanmalı |
| Rusya | Yerel ödeme sağlayıcısı (ör. YooKassa, CloudPayments); yerel tüzel kişilik gerektirir | 2022’den beri mağaza ödemeleri kısıtlı; yaptırım uyumu hukuken değerlendirilmeli; sonraki faz |
| Kazakistan, Özbekistan | Kart + yerel cüzdanlar (ör. Kaspi; Payme, Click) | Doğrulanmalı |
| Türkiye | iyzico + mağaza IAP | — |

- Aktivasyon kodları: tek kullanımlık, imzalı, partiye/bayiye bağlı; `activation_codes` tablosu, bayi bazında satış raporu ve iptal.
- Fiyatlar yerel para biriminde ve satın alma gücüne göre ülke bazında belirlenir.
- **Öğretmen payout’u:** Stripe Connect Türkmenistan’ı desteklemez; yerel öğretmenlere ödeme için yerel ortak üzerinden toplu ödeme seçeneği araştırılmalı.

### 6.4 Ağ ve erişim koşulları

Türkmenistan’da internet görece yavaş ve pahalı; birçok yabancı servis ve alan adı engellenebiliyor. Bu yüzden:

- **Tek alan adı:** İstemciler yalnızca kendi alan adlarımızla konuşur (API, CDN, medya). Analitik, hata takibi, font gibi üçüncü taraf istekleri kendi proxy’miz üzerinden geçer; fontlar uygulamayla paketlenir.
- **Giriş:** Telefon numarası + SMS OTP birincil yöntem; Google/Apple girişi ikincil. SMS sağlayıcısının +993 numaralara teslimatı test edilmeli.
- **Bildirim:** FCM/APNs erişilemezse yedek olarak uygulama içi bildirim kutusu (açılışta çekilir) ve SMS.
- **Dağıtım:** Google Play’e erişemeyen kullanıcılar için web sürümü (PWA) ve sitemizden doğrudan APK; uygulama içi güncelleme kontrolü.
- **Hafiflik:** Opus düşük bitrate ses, WebP/AVIF görseller, ders paketlerinin önceden indirilmesi (offline), uygulama boyutu bütçesi.
- **Görüntülü ders:** WebRTC/UDP engellenebilir; LiveKit’in TCP/TLS 443 üzerinden TURN yedeği zorunlu. Düşük bant için “sadece ses” modu ve otomatik kalite düşürme.
- **Ölçüm:** Lansmandan önce Aşkabat’taki test kullanıcılarıyla gerçek ağ testi (gecikme, engellenen alan adları). Frankfurt sonuçlara göre yeterli değilse bölgeye yakın CDN/edge değerlendirilir.

### 6.5 Hukuk ve veri

- Türkmenistan kişisel veri mevzuatı ve olası veri yerelleştirme şartları yerel hukukçuyla doğrulanmalı.
- Rusya’ya girilirse Rus vatandaşlarının kişisel verileri Rusya’da tutulmalı (242-FZ); ayrı bir bölge ve altyapı kararı gerektirdiği için Rusya sonraki faza bırakılır.
- Creator içerikleri yayından önce hedef ülkelerin içerik kurallarına göre incelenir (`moderation`).

---

## 7. Çekirdek veri modeli

```
users ─┬─ user_roles, profiles, follows, devices
       ├─ enrollments ── lesson_progress ── attempts
       ├─ review_items            (FSRS: stability, difficulty, due_at)
       ├─ streaks, gem_transactions, league_memberships, quest_progress, achievements
       ├─ subscriptions, purchases, entitlements
       └─ tickets ── ticket_messages

courses (source_lang, target_lang, owner_type, creator_id, status)
  └─ units ── lessons ── exercises (type, payload jsonb, version)
                            └─ exercise_translations (source_lang, prompt, hints, accepted_answers)
media_assets (s3_key, kind, duration, transcoding_status)

teacher_profiles (speaks[], country) ── availability_rules ── bookings ── sessions (livekit_room, recording_key) ── reviews
creator_profiles ── payout_accounts

countries (currency, payment_providers[], price_tier)
prices (product, country, amount)
activation_codes (code_hash, product, batch, reseller_id, redeemed_by, redeemed_at)
ledger_accounts ── ledger_entries (double-entry)  ── payouts
audit_logs, reports
```

---

## 8. Para akışları

| Gelir kaynağı | Nasıl satılır | Kim kazanır |
|---|---|---|
| **Simba Premium** (aylık/yıllık) | Ülkeye göre: mağaza IAP (RevenueCat), web’de iyzico/Stripe/yerel sağlayıcı, Türkmenistan’da aktivasyon kodu | Platform; bir kısmı creator havuzuna ayrılır |
| **Öğretmen dersleri** (kredi/paket) | Web + mobil; Türkmenistan’da ders kredisi kodu | Öğretmen ders başı hak ediş (ör. %75), platform komisyonu |
| **Creator kursları** | Tekil satış veya “Premium’a dahil” | Satışta ör. %70 creator; Premium havuzunda tamamlanan ders süresine göre pay |

- Kazançlar ledger’a **bekleyen** olarak düşer, iade süresi (ör. 14 gün) dolunca **kullanılabilir** olur, aylık payout ile ödenir (Stripe Connect; desteklenmeyen ülkelerde Payoneer/Wise).
- App Store, birebir canlı insan hizmetleri için uygulama dışı ödemeye izin verir (Guideline 3.1.3(d)); öğretmen paketleri bu kapsamda değerlendirilebilir. Dijital içerik (Premium, creator kursları) mobilde IAP gerektirir. Güncel mağaza kuralları ve vergi tarafı lansmandan önce hukuken doğrulanmalı.

---

## 9. Altyapı ve DevOps

| Alan | Seçim |
|---|---|
| Bulut | AWS `eu-central-1` (Frankfurt) — Türkiye ve Orta Asya’ya makul gecikme; Türkmenistan ölçümüne göre edge eklenir (6.4) |
| Backend çalıştırma | ECS Fargate (API, worker’lar). Kubernetes ihtiyaç doğana kadar ertelenir |
| Web uygulamaları | Vercel (Next.js) |
| Veritabanı / cache | RDS PostgreSQL (Multi-AZ), ElastiCache Redis |
| Medya | S3 + CloudFront; creator videoları için Mux (yükleme, HLS, oynatıcı) |
| Arama | Meilisearch (öğretmen ve kurs arama, Türkçe/Çince destekli) |
| Edge | Cloudflare (DNS, WAF, DDoS) |
| IaC & CI/CD | Terraform, GitHub Actions, Expo EAS Build/Submit/Update |
| Gözlemlenebilirlik | Sentry (hata), OpenTelemetry + Grafana Cloud (metrik/log/trace), PostHog |

**Güvenlik ve uyum**
- Admin paneli ayrı domain, SSO + 2FA, IP kısıtı; tüm işlemler audit log’da.
- KVKK: yurtdışı veri aktarımı için açık rıza/aydınlatma metni; hassas alanlar şifreli.
- 13 yaş altı kullanıcı için ebeveyn onayı; öğretmen derslerinde şikayet akışı ve (onaylı) kayıt.
- Öğretmen başvurularında kimlik doğrulama (KYC) ve demo ders değerlendirmesi.

---

## 10. Monorepo yapısı

```
simba/
├─ apps/
│  ├─ mobile/          # Simba öğrenci uygulaması (Expo)
│  ├─ mobile-teach/    # Simba Teach (Expo)
│  ├─ web/             # Öğrenci web + pazarlama (Next.js)
│  ├─ teach/           # Öğretmen web (Next.js)
│  ├─ studio/          # İçerik üretici paneli (Next.js)
│  ├─ admin/           # Admin & destek paneli (Next.js)
│  ├─ api/             # NestJS modüler monolit
│  └─ worker/          # BullMQ job’ları (lig, bildirim, payout, medya)
├─ packages/
│  ├─ ui/              # Tasarım token’ları + ortak bileşenler (NativeWind/Tailwind)
│  ├─ exercise-engine/ # Egzersiz tipleri, render + değerlendirme mantığı (web & mobil ortak)
│  ├─ api-client/      # OpenAPI’den üretilen tipli istemci
│  ├─ i18n/            # tk / ru / tr çevirileri (ICU), Türkmence tarih/sayı tabloları
│  └─ config/          # eslint, tsconfig, tailwind preset
└─ infra/              # Terraform
```

---

## 11. Yol haritası

| Faz | Süre (tahmini) | Kapsam |
|---|---|---|
| **0 — Temel** | 2–3 hafta | Tasarım sistemi, monorepo, CI/CD, kimlik doğrulama (telefon + SMS), i18n altyapısı (tk/ru/tr), ortamlar; Türkmenistan’dan ağ ve ödeme ön araştırması |
| **1 — MVP (Türkmenistan)** | ~3 ay | Öğrenci mobil + web (Türkmence, Rusça, Türkçe arayüz), egzersiz motoru (4 hedef dil), tk→en ve ru→en A1 içerik, can/seri/görev/lig, Premium + aktivasyon kodu, Admin v1 (kullanıcı, destek, kod/bayi yönetimi), Studio v0 (iç içerik ekibi) |
| **2 — Öğretmenler** | ~2 ay | Simba Teach mobil + web, takvim & rezervasyon, “Şimdi Görüş”, LiveKit (TURN/443 + sadece ses modu), ders kredisi kodları, öğretmen payout |
| **3 — Creator pazarı** | ~2 ay | Studio dış üreticilere açılır, inceleme akışı, kurs satışı, Premium havuzu, creator payout |
| **4 — Genişleme** | sürekli | tk→ru, Çince ve Fransızca kursları; Özbekistan/Kazakistan ödeme adaptörleri ve dilleri; AI sohbet partneri, telaffuz puanlama; Rusya için ayrı hukuk/altyapı kararı |

**MVP ekibi önerisi:** 1 tech lead (backend), 1 backend, 2 TypeScript frontend (RN + Next.js), 1 UI/UX tasarımcı, yarı zamanlı QA, kaynak dil başına (tk, ru) en az 1 anadili editör + hedef dil başına 1 içerik editörü, Aşkabat’ta yerel iş ortağı (ödeme, bayi, test).
