# PDKS (Personel Devam Kontrol Sistemi) — Planlama ve Yol Haritası Raporu

> **Sürüm:** 2 — Kullanıcı kararlarıyla güncellenmiştir.
>
> **Konu:** Android ve iPhone cihazlarla, kiosk ekranında değişen QR kod okutarak
> personel giriş–çıkış ve ara (mola) çıkış–giriş takibi; web panelinden personel
> kaydı ve raporlama; bordro için puantaj verisinin sistem tarafından hazırlanması
> (dış bordro programına entegrasyon yoktur).

---

## 1. Kilitlenen Kararlar (Sizin Onayınızla)

| # | Konu | Karar |
|---|---|---|
| 1 | Kullanım modeli | **Model A:** Girişteki tablet/kiosk ekranı sürekli değişen QR gösterir, personel telefonundaki uygulamayla okutur |
| 2 | Mobil uygulama | **Flutter** — tek kod tabanı, iOS + Android |
| 3 | Backend | **Go / Fiber** (REST API) |
| 4 | Veritabanı | **MariaDB** |
| 5 | Web paneli | Fiber backend'e bağlı web yönetim paneli (personel kaydı, vardiya, raporlar); panel arayüzü aynı Fiber servisi üzerinden sunulur |
| 6 | Zaman doğrulaması | Telefon saati **kullanılmaz**; tüm doğrulama **sunucu saatine** göre yapılır |
| 7 | GPS kontrolü | **Yok** — konum alınmaz, geofence uygulanmaz |
| 8 | Cihaz bağlama | **Yok** — personel hesabı belirli bir telefona kilitlenmez (bkz. Bölüm 2'deki not) |
| 9 | Geç kalma toleransı | **15 dakikaya kadar** tolere edilir; 15 dakikayı aşan gecikme "işlem yapılacak" olarak işaretlenir (işlemin ne olduğu Bölüm 8'de sorulmuştur) |
| 10 | Fazla mesai / eksik çalışma | Otomatik hesaplanır; **fazla mesai ile eksik çalışma arasındaki fark (net)** de hesaplanır (netleştirme periyodu Bölüm 8'de sorulmuştur) |
| 11 | İK düzeltmeleri | Panelden yapılır ve **kim/ne zaman/eski→yeni değer** ile loglanır; kayıt silinmez |
| 12 | Bordro | Dış programa **entegrasyon yok**; ancak bordroya esas puantaj verisi sistemde hazırlanır, Excel/PDF çıktısı alınır |

---

## 2. QR Motoru Seçimi (Faz 2) — Tarafımca Seçilen En Güvenli Tasarım

Seçim görevi bana bırakıldığı için, Model A + sunucu saati kararlarınıza uygun
**en güvenli** tasarımı seçtim:

### "Dönen İmzalı Tek Kullanımlık QR Token" (Server-Authoritative Rotating QR)

```text
1. Kiosk, kendi kiosk kimliğiyle sunucuya bağlanır (HTTPS).
2. Sunucu (Go/Fiber) her 30 saniyede bir yeni token üretir:
      { kiosk_id, lokasyon_id, iat, exp, jti }  →  Ed25519 (EdDSA) ile imzalanır
   - iat/exp : sunucu saatine göre düzenlenme ve bitiş anı
   - jti     : her token'a özel benzersiz kimlik (nonce)
3. Kiosk token'ı QR olarak ekranda gösterir (içinde personel verisi YOKTUR).
4. Personel, uygulamaya hesabıyla giriş yapmış olarak QR'ı okutur ve
   sıradaki işlemi (GİRİŞ / ARA ÇIKIŞ / ARA GİRİŞ / ÇIKIŞ) onaylar.
5. Uygulama token'ı HTTPS ile sunucuya gönderir. Sunucu sırayla doğrular:
   a. İmza geçerli mi? (Sunucu özel anahtarı olmadan sahte QR üretilemez)
   b. Süresi dolmuş mu? (sunucu saatiyle, exp kontrolü)
   c. Bu personel bu jti'yi daha önce kullanmış mı? (kullanmışsa reddet)
   d. Kiosk aktif mi, personel aktif mi?
   e. İstenen işlem, personelin o anki durumuna uygun mu? (Bölüm 4 durum makinesi)
6. Token kullanımı, hareket kaydıyla aynı veritabanı işleminde (transaction)
   kaydedilir → aynı personel aynı QR ile ikinci kez kayıt oluşturamaz.
7. Başarısız denemeler (süresi dolmuş, tekrar kullanım, sahte imza)
   güvenlik loguna yazılır; kiosk panelden tek tıkla devre dışı bırakılabilir.
```

**Teknik detaylar**

- İmza: **EdDSA (Ed25519) JWT (JWS)**; özel anahtar yalnızca sunucuda (ortam
  değişkeni / gizli anahtar kasası), anahtar rotasyonu için `kid` alanı kullanılır.
- Token ömrü: varsayılan **30 saniye** (20–60 sn arası parametrik). Kiosk ekranında
  kalan süre göstergesi bulunur; kiosk saat kayması yaşamaz çünkü token'ı sunucu üretir.
- "Tek kullanımlık" kuralı **personel bazında** uygulanır: MariaDB'de
  `(jti, personel_id)` çifti benzersizdir. Böylece:
  - Aynı personel, aynı QR ile ikinci kez kayıt oluşturamaz (replay engellenir);
  - Farklı personeller, aynı 30 saniyelik pencerede aynı kiosk QR'ıyla sırayla
    işlem yapabilir (girişte kuyruk tıkanmaz; aksi hâlde ilk okutan token'ı
    "tüketir", diğerleri giremezdi).
- Token, hareket kaydıyla aynı transaction içinde "bu personel tarafından
  kullanıldı" olarak işaretlenir.
- Hız sınırlama (rate limit) ve arka arkaya hatalı deneme alarmı uygulanır.

### ⚠️ Çelişki notu ve sizden tek cümlelik teyit isteği

Birinci mesajınızdaki şu ifade ile "en güvenli QR motorunu seç" talimatınız
birbiriyle kısmen çelişiyor:

> "QR ömrünü 30–60 saniyeye indirmek, imzalı ve tek kullanımlık token, cihaz
> bağlama ve GPS kontrolü **istemiyorum**."

Ben bunu şöyle yorumladım ve raporu böyle güncelledim:

- **GPS kontrolü: yok** ✔ (net kararınız)
- **Cihaz bağlama: yok** ✔ — personel yeni telefondan da hesabıyla giriş yapabilir;
  dolayısıyla paneldeki "cihaz eşleştirme/sıfırlama" ekranı da kapsamdan çıkar.
- **Ancak** kısa ömürlü + imzalı + tek kullanımlık (replay korumalı) token,
  QR motorunun *kendi iç mekanizmasıdır*; bunlar olmadan "en güvenli" bir motor
  teknik olarak mümkün değildir. İmzasız ve uzun ömürlü bir QR, düz bir metinden
  ibarettir: fotoğrafı çekilen kod saatler sonra bile herkeste çalışır.

> **Teyit isteği:** Bu yorumumu onaylıyor musunuz?
> Onaylamıyorsanız (yani imzasız, saatlik değişen basit QR istiyorsanız) bunu
> açıkça belirtin; o seçeneği de geliştiririz ancak ekran görüntüsüyle suistimal
> riskinin tarafınızca kabul edildiğini rapora işleriz.

**Kabul edilen artık risk (GPS ve cihaz bağlama olmadan):** Kioskun fotoğrafı
o an çekilip uzaktaki personele gönderilirse, personel 30 saniyelik pencere
içinde uzaktan da okutabilir. Kısa ömür + pencere başına sınırlı süre bu riski
küçültür ama sıfırlamaz. Riski daha da azaltmak isterseniz sonradı eklenebilecek
opsiyonlar: kiosk ekranında saniyelik değişen görsel doğrulama deseni, anormal
okutma raporu (aynı dakikada çakışan işlemler vb.).

---

## 3. Sistem Mimarisi (Güncel)

```text
┌────────────────────┐        ┌────────────────────┐
│  MOBİL UYGULAMA    │        │   WEB YÖNETİM      │
│  Flutter (iOS+And.)│        │   PANELİ           │
│ • Hesapla giriş    │        │ • Personel kaydı   │
│ • QR okutma        │        │ • Lokasyon/kiosk   │
│ • Giriş/Çıkış      │        │ • Vardiya tanımları│
│ • Ara çıkış/giriş  │        │ • Raporlar / Excel │
│ • Kendi hareketleri│        │ • Düzeltme + log   │
└─────────┬──────────┘        └─────────┬──────────┘
          │                             │
          │        HTTPS / REST API     │
          └──────────────┬──────────────┘
                         ▼
          ┌──────────────────────────────┐
          │   BACKEND — Go / Fiber       │
          │ • JWT kimlik doğrulama       │
          │ • QR token üretimi/doğrulama │
          │   (EdDSA, sunucu saati, jti) │
          │ • Durum makinesi + puantaj   │
          │ • Bordro puantaj hazırlama   │
          └──────────────┬───────────────┘
                         ▼
          ┌──────────────────────────────┐
          │        MariaDB               │
          │ Personel, kiosk, vardiya,    │
          │ hareket, token, düzeltme log │
          └──────────────────────────────┘

   Kiosk ekranı: Fiber'ın sunduğu web sayfası; sunucudan dönen QR'ı gösterir.
```

| Katman | Seçim | Not |
|---|---|---|
| Mobil | Flutter | QR tarama, push bildirim, güvenli token saklama |
| Backend | Go + Fiber | Tek binary, düşük kaynak, hızlı; REST + JWT |
| Veritabanı | MariaDB | Puantaj sorguları, token tek-kullanım için benzersiz indeks + transaction |
| Kiosk | Tarayıcı tabanlı sayfa | Tablette tarayıcı tam ekran açılır; uygulama kurulumu gerekmez |
| Çıktı | Excel + PDF | Bordro dönemine uygun puantaj |
| Barındırma | VPS/bulut + Docker, HTTPS, günlük yedek | MariaDB yedeği farklı lokasyona kopyalanır |

---

## 4. İş Kuralları

```text
        ┌──────────┐   ARA ÇIKIŞ   ┌──────────┐
        │          │ ────────────► │          │
  GİRİŞ │ İÇERİDE  │               │  ARADA   │
 ─────► │          │ ◄──────────── │  (MOLA)  │
        └────┬─────┘  ARA GİRİŞ    └──────────┘
             │ ÇIKIŞ
             ▼
        ┌──────────┐
        │ DIŞARIDA │  (gün kapanır, mesai hesaplanır)
        └──────────┘
```

1. Personelin o anki durumuna göre uygulama yalnızca **geçerli sıradaki işlemleri**
   önerir (örn. içerideyse ARA ÇIKIŞ veya ÇIKIŞ); sunucu uygun olmayan işlemi reddeder.
2. **Geç kalma:** Planlanan giriş + **15 dk** içinde giriş → normal. 15 dk sonrası →
   "tolerans dışı geç" olarak işaretlenir ve rapora işlenir. (Bu gecikmelerin
   ücretten kesilip kesilmeyeceği vb. işlem detayı Bölüm 8, Soru 2'dedir.)
3. Ara süreleri toplamı gün sonunda hesaplanır; çalışma süresinden düşülüp
   düşülmeyeceği vardiya tanımında parametredir.
4. **Fazla mesai** = planlanan çıkış sonrası çalışılan süre; **eksik çalışma** =
   planlanan − gerçekleşen (pozitifse). İkisi de gün gün hesaplanır.
5. **Net fark** = toplam fazla mesai − toplam eksik çalışma (periyot bazında;
   bordro puantajında hem brüt değerler hem net ayrı sütunlarda gösterilir —
   fazla mesainin ücret karşılığı ile eksik çalışmanın kesintisi hukuken aynı
   katsayıya tabi olmadığından, net tutar "bilgi amaçlı" üretilir, nihai bordro
   hesabını İK/muhasebe yapar; bkz. Soru 4).
6. Çıkışsız kapanan gün → "eksik kayıt"; İK panelden düzeltir, düzeltme loglanır.
7. İzin, rapor ve resmî tatiller çalışma hesabında mazeret olarak işlenir.

---

## 5. Veri Modeli (MariaDB — ana tablolar)

| Tablo | Önemli alanlar |
|---|---|
| `personel` | ad_soyad, sicil_no, departman_id, unvan, ise_giris, telefon, eposta, durum |
| `kullanicilar` | personel_id, kullanici_adi, sifre_hash (bcrypt/argon2), rol |
| `lokasyonlar` | ad, adres |
| `kioskler` | lokasyon_id, ad, kiosk_token_hash, aktif, son_gorülme |
| `vardiyalar` | ad, giris_saat, cikis_saat, gec_tolerans_dk (15), ara_dusulsun_mu |
| `personel_vardiya` | personel_id, vardiya_id, baslangic, bitis |
| `hareketler` | personel_id, tip, zaman (sunucu), kiosk_id, qr_jti, kaynak |
| `qr_tokenlar` | jti (UNIQUE), kiosk_id, iat, exp |
| `qr_kullanimlar` | jti + personel_id (birlikte UNIQUE), hareket_id, zaman |
| `izinler` / `tatiller` | personel/tarih, tip, onaylayan |
| `puantaj_gunluk` | personel, tarih, normal_sure, ara_sure, gec_dk, fazla_mesai_dk, eksik_dk |
| `duzeltme_log` | kim, personel, alan, eski, yeni, aciklama, zaman |

*GPS alanı ve cihaz bağlama tablosu, kararlarınız gereği yoktur.*

---

## 6. Web Paneli Ekranları ve Raporlar

**Ekranlar:** Dashboard (bugün gelen/gelmeyen/geç kalan, anlık içeridekiler),
Personel yönetimi (Excel'den toplu aktarım dahil), Lokasyon/Kiosk yönetimi
(kiosk aktif/pasif, son görülme), Vardiya tanımları, Canlı hareket akışı,
Düzeltme/onay ekranı, Kullanıcı ve rol yönetimi.

**Raporlar:** Günlük devam listesi, aylık puantaj (gün gün), geç kalma
(15 dk toleranslı/tolerans dışı ayrımıyla), fazla mesai, eksik kayıt,
departman özetleri — ekran + Excel/PDF.

**Bordro hazırlığı (entegrasyonsuz):** Dönem sonunda personel bazında
"bordroya esas puantaj özeti" üretilir: çalışılan gün, normal süre, ara toplamı,
fazla mesai, eksik çalışma, **net fark**, izin/tatil günleri. Bu özet Excel
olarak bordroyu hazırlayan kişiye teslim edilir. (Sistemin net maaş, SGK ve
vergi hesaplayan tam bordro modülü olup olmayacağı ayrı bir kapsam kararıdır —
Soru 6.)

---

## 7. Geliştirme Fazları (Güncel)

| Faz | Kapsam | Çıktı |
|---|---|---|
| **Faz 0** Analiz | Bu raporun onayı, kalan soruların cevabı, ekran taslakları | Onaylı kapsam |
| **Faz 1** Web paneli | Go/Fiber + MariaDB iskeleti, giriş/roller, personel-lokasyon-vardiya tanımları | Personelin webden kaydedilebildiği panel |
| **Faz 2** QR motoru + mobil | Bölüm 2'deki token motoru, kiosk sayfası, Flutter uygulama (giriş, QR okutma, 4 işlem tipi, kendi hareketleri) | Uçtan uca giriş-çıkış |
| **Faz 3** Raporlama/puantaj | Günlük puantaj motoru, 15 dk tolerans, fazla mesai/eksik/net hesabı, bordro özeti, düzeltme+log | Bordroya esas aylık puantaj |
| **Faz 4** Pilot + yayın | Küçük grupla pilot, güvenlik testleri (sahte/tekrar QR denemeleri), App Store / Google Play | Canlı sürüm |
| **Faz 5** Genişletme (isteğe bağlı) | Turnike entegrasyonu, çok şube, tam bordro modülü vb. | İhtiyaca göre |

---

## 8. Sizden Hâlâ İhtiyacım Olan Bilgiler

Evet, birkaç bilgi daha gerekiyor. **Faz 1'e başlamadan önce şart olanlar**
(1–6) ve **sonra da netleşebilecekler** (7–10) diye ayırdım:

### Başlamadan önce şart
1. **QR teyidi:** Bölüm 2'deki yorumumu onaylıyor musunuz? (GPS yok, cihaz
   bağlama yok; ama QR'ın içi imzalı/kısa ömürlü/replay-korumalı olacak.)
2. **15 dakikayı aşan gecikmede "yapılacak işlem" nedir?** Örn: sadece raporlama
   mı, ücret kesintisi mi (kaç dk üzerinden — aşan kısım mı tamamı mı?),
   uyarı/disiplin süreci mi? Ayrıca 15 dk içindeki gecikme "geç" sayılmazken
   o süre çalışılmış mı sayılır?
3. **Vardiya düzeni:** Giriş-çıkış saatleri kaçtır, haftada kaç gün, hafta sonu
   ve resmî tatil çalışması var mı? Herkes tek vardiya mı, dönen vardiya var mı?
   Öğle arası sabit mi (örn. 12:30–13:30 otomatik düşüm) yoksa ara çıkış/giriş
   yalnızca personelin okutmasıyla mı hesaplanacak?
4. **Fazla mesai–eksik netleştirmesi:** Net fark günlük mü aylık mı hesaplanacak?
   Fazla mesai için amir onayı gerekli mi? Süreler yuvarlanacak mı (örn. 15 dk'lık
   bloklar)? Net eksi çıkarsa ücretten kesinti yapılacak mı?
5. **Kapsam:** Kaç personel, kaç lokasyon ve kaç kiosk olacak? Kiosk olarak ne
   kullanılacak (Android tablet / iPad / TV+tarayıcı)? Giriş noktalarında
   internet durumu nedir?
6. **"Bordro hazırlanacak" kapsamı:** Sistemden beklenen yalnızca bordroya esas
   **puantaj özeti** mi, yoksa brüt→net maaş, SGK, gelir vergisi hesaplayan
   **tam bordro** mu? (Bu, kapsamı en çok büyüten kalemdir; tam bordro ise
   personel kartına ücret tipi, aylık/saatlik ücret gibi alanlar da gerekir.)

### Geliştirme sırasında netleşebilir
7. **Mobil giriş yöntemi:** Sicil no + şifre mi, telefon + SMS kodu mu,
   e-posta + şifre mi? Şifreyi ilk kim oluşturacak, unutan nasıl yenileyecek?
8. **Personel kartında hangi alanlar olacak?** (TC kimlik no isteniyor mu,
   IBAN/ücret bilgisi girilecek mi — 6. soruya bağlı)
9. **İzin/onay akışları:** İzin talebini personel uygulamadan mı girecek,
   İK mı tanımlayacak? Fazla mesai onayı amirde mi olacak?
10. **Barındırma ve hesaplar:** Sunucu sizde mi bulutta mı? Apple Developer
    (yıllık ücretli) ve Google Play hesaplarınız var mı, açılacak mı?
    Bildirim kanalı: uygulama içi push yeterli mi, e-posta da isteniyor mu?

---

## 9. Özet

- Mimari kilitlendi: **Flutter + Go/Fiber + MariaDB**, Model A (kiosk QR'ı gösterir).
- GPS ve cihaz bağlama yok; zaman doğrulaması sunucu saatine göre.
- QR motoru olarak en güvenli tasarım seçildi: **EdDSA imzalı, 30 saniyelik,
  replay-korumalı dönen token** (Bölüm 2 — teyidiniz bekleniyor).
- İş kuralları: durum makinesi, **15 dk geç toleransı**, fazla mesai / eksik /
  net fark hesabı, loglu İK düzeltmesi, bordro için puantaj özeti (entegrasyonsuz).

Bölüm 8'deki 1–6. soruları cevapladığınızda Faz 0 tamamlanır; ekran taslaklarını
çıkarıp Faz 1 (web paneli) geliştirmesine başlayabiliriz.
