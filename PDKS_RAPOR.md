# PDKS (Personel Devam Kontrol Sistemi) — Planlama ve Yol Haritası Raporu

> **Konu:** Android ve iPhone cihazlarla, dinamik (süreli) QR kod okutarak personel
> giriş–çıkış ve ara (mola) çıkış–giriş takibi yapılması; web panelinden personel
> kaydı, cihaz eşleşmesi ve raporlama sağlanması.
>
> **Bu dokümanın amacı:** Sistemi kodlamaya başlamadan önce *ne yapacağımızı, nasıl
> yapacağımızı, hangi kararları sizin vermeniz gerektiğini* netleştirmektir.
> Bu bir teknik tasarım + iş planı raporudur; onayınızdan sonra geliştirme fazlarına geçilir.

---

## 1. İhtiyacın Özeti ve Kritik Yorum

İstediğiniz sistemin çekirdeği şudur:

1. Personel, **Android / iPhone** telefonundan bir işlem yaparak giriş–çıkışını kaydeder.
2. QR kod **zaman damgalı** olur ve **belirli aralıklarla değişir** (siz "saatte bir" dediniz).
3. Personel ayrıca **ara çıkış / ara giriş** (mola, dışarı çıkma vb.) kaydı da oluşturur.
4. Yöneticiler **web sayfasından** personeli kaydeder, cihazlarla eşleştirir ve
   giriş–çıkış raporlarını alır.

### 1.1. Önemli teknik uyarı: "Saatte bir değişen QR" tek başına güvenli değildir

Saatte bir değişen bir QR kod, o saat içinde **ekran görüntüsü alınıp başka bir
personelle paylaşılabilir**. Yani personel iş yerinde olmadan, evden fotoğrafı
okutarak "giriş" yapmış gibi görünebilir. Bu, PDKS'nin en klasik suistimal yoludur.

Bu yüzden raporda şu güçlendirmeleri öneriyoruz (saatlik yenilemeyi yine de
yapabiliriz, fakat tek güvenlik önlemi o olmamalıdır):

| Önlem | Ne işe yarar |
|---|---|
| QR geçerliliğini 30–60 saniyeye indirmek (önerilen) | Ekran görüntüsü paylaşımını pratik olarak engeller |
| QR içine imzalı, tek kullanımlık token koymak (HMAC/JWT + nonce) | Aynı QR ikinci kez kullanılamaz, sahte QR üretilemez |
| Telefonun o anki GPS konumunu kontrol etmek (geofence) | Sadece iş yeri çevresinden (örn. 100–200 m) okutmaya izin verir |
| Cihaz bağlama (device binding) | Her personel hesabı yalnızca kayıtlı tek telefondan işlem yapabilir |
| Sunucu saati ile doğrulama | Telefon saati değiştirilerek yapılan hileleri engeller |
| Sahte GPS (mock location) tespiti | Konum hilesi yapan uygulamaları yakalar |
| (İsteğe bağlı) Selfie / canlı doğrulama | En yüksek güvenlik gereken yerlerde kullanılır |

> **Karar noktası 1:** QR yenileme süresini 1 saat mi tutalım, yoksa önerildiği gibi
> 30–60 saniye mi yapalım? Teknik maliyet aynıdır; güvenlik farkı büyüktür.

### 1.2. İki olası kullanım modeli — hangisini seçeceğiz?

İfadeniz iki şekilde de yorumlanabildiği için iki modeli de netleştiriyoruz:

**Model A — Sabit ekran QR gösterir, personel telefonuyla okutur (ÖNERİLEN)**

```text
Girişteki tablet / TV / kiosk ekranı
  └─ Sürekli değişen QR kodu gösterir
        └─ Personel, telefonundaki PDKS uygulamasını açıp QR'ı okutur
              └─ Uygulama: kimlik + GPS + cihaz bilgisi ile sunucuya gönderir
                    └─ Sunucu doğrular → giriş/çıkış kaydı oluşur
```

- Artıları: Tek cihaz (kiosk) merkezden yönetilir; personel QR üretemez/saklayamaz;
  sahtecilik en zor modeldir; donanım maliyeti düşüktür (bir tablet yeterlidir).
- Eksileri: Giriş noktasına bir tablet/ekran koymak gerekir.

**Model B — Personelin telefonu QR üretir, yetkili cihaz/kiosk okutur**

```text
Personelin telefonu
  └─ Oturum açmış uygulama, zaman damgalı kişisel QR üretir
        └─ Girişteki kiosk / görevli telefonu QR'ı okutur
              └─ Sunucu doğrular → kayıt oluşur
```

- Artıları: Personel telefonu dışında ek ekran gerekmez (görevli cihazı yeterli).
- Eksileri: Okutan cihazın başında biri/kiosk olması gerekir; kuyruk oluşabilir;
  personelin şarjı/interneti yoksa işlem aksar.

> **Karar noktası 2:** Model A mı, Model B mi? (İki model de aynı altyapıyı kullanır;
> fark mobil uygulamadaki ekrandadır. İstenirse ikisi birden de desteklenebilir,
> fakat ilk sürümde tek modelle başlamak daha sağlıklıdır.)

---

## 2. Sistem Mimarisi

Sistem **üç ana parçadan** oluşur; hepsi tek bir merkezi sunucu (backend) üzerinden
haberleşir. Telefonlar birbirleriyle doğrudan konuşmaz; her şey sunucu üzerinden ve
kayıt altına alınarak yapılır.

```text
┌────────────────────┐        ┌────────────────────┐
│  MOBİL UYGULAMA    │        │   WEB YÖNETİM      │
│  (iOS + Android)   │        │   PANELİ           │
│                    │        │                    │
│ • Giriş yapma      │        │ • Personel kaydı   │
│ • QR okutma/üretme │        │ • Cihaz eşleştirme │
│ • Giriş/Çıkış      │        │ • Vardiya tanımları│
│ • Ara çıkış/giriş  │        │ • Raporlar / Excel │
│ • Kendi hareketleri│        │ • Yetki yönetimi   │
└─────────┬──────────┘        └─────────┬──────────┘
          │  HTTPS / REST API           │
          └──────────────┬──────────────┘
                         ▼
          ┌──────────────────────────────┐
          │        BACKEND (SUNUCU)      │
          │ • Kimlik doğrulama (JWT)     │
          │ • QR token üretimi/doğrulama │
          │ • Giriş-çıkış iş kuralları   │
          │ • Rapor hesaplama motoru     │
          │ • Bildirimler (push/e-posta) │
          └──────────────┬───────────────┘
                         ▼
          ┌──────────────────────────────┐
          │     VERİTABANI (PostgreSQL)  │
          │ Personel, cihaz, lokasyon,   │
          │ vardiya, hareket kayıtları   │
          └──────────────────────────────┘
```

### 2.1. Önerilen teknoloji seçimleri

| Katman | Öneri | Gerekçe |
|---|---|---|
| Mobil uygulama | **Flutter** (tek kod, iOS + Android) | İki platforma ayrı ekip gerekmez; QR kamera, GPS, push bildirim desteği olgun |
| Web paneli | React / Next.js | Hızlı, raporlama ve tablo bileşenleri güçlü |
| Backend | Node.js (NestJS) veya .NET — mevcut ekibinize göre seçilir | REST API + JWT; ikisi de uygundur, ekip tecrübesi belirleyici olmalı |
| Veritabanı | PostgreSQL | Raporlama sorguları ve güvenilirlik için ideal |
| QR token | Kısa ömürlü imzalı JWT / HMAC token | İçinde lokasyon ID, zaman damgası, nonce; sunucu imzası olmadan üretilemez |
| Sunucu | Bulut VPS (örn. Hetzner/AWS/Azure) + Docker | HTTPS (Let's Encrypt), günlük yedek, kolay ölçekleme |
| Bildirim | Firebase Cloud Messaging (FCM + APNs) | Geç kalan / kayıt oluşturmayan personele ve yöneticiye uyarı |
| Dosya çıktısı | Excel (xlsx) + PDF | Bordro dönemine uygun rapor teslimi için |

> Teknoloji seçimi kilit değildir; önemli olan mimarinin bu üç katmanlı yapıda
> kurulmasıdır. Ekibinizde hâlihazırda bir dil varsa backend o dille yazılmalıdır.

---

## 3. Kullanıcı Rolleri ve Yetkiler

| Rol | Yapabilecekleri |
|---|---|
| **Süper Admin** | Tüm şirket/lokasyon ayarları, kullanıcı ve yetki yönetimi, sistem parametreleri |
| **İK / Admin** | Personel ekleme-düzenleme, cihaz eşleştirme/sıfırlama, vardiya ve izin tanımları, tüm raporlar |
| **Yönetici / Amir** | Sadece kendi departmanının raporlarını görme, geç gelme/onay işlemleri, düzeltme talepleri |
| **Personel** | Mobil uygulamadan giriş-çıkış/ara işlemi, kendi hareket geçmişini ve aylık özetini görme |

---

## 4. Temel İş Kuralları (Giriş–Çıkış Mantığı)

Sistem, her personelin gün içindeki durumunu bir **durum makinesi** olarak takip eder.
Bu, hatalı kayıtların (örn. iki kere üst üste "giriş") önüne geçer:

```text
        ┌──────────┐   ARA ÇIKIŞ   ┌──────────┐
        │          │ ────────────► │          │
  GİRİŞ │ İÇERİDE  │               │  ARADA   │
 ─────► │ (ÇALIŞMA)│ ◄──────────── │  (MOLA)  │
        │          │  ARA GİRİŞ    │          │
        └────┬─────┘               └──────────┘
             │ ÇIKIŞ
             ▼
        ┌──────────┐
        │ DIŞARIDA │  (gün kapanır / mesai hesaplanır)
        └──────────┘
```

Kurallar:

1. İlk okutma **GİRİŞ** olur; içerideyken sıradaki işlem **ARA ÇIKIŞ** veya **ÇIKIŞ** olabilir.
2. Ara çıkıştan sonra yalnızca **ARA GİRİŞ** yapılabilir; ara süreleri toplam çalışma
   süresinden düşülür (veya şirket politikanıza göre düşülmez — parametrik olmalıdır).
3. Gün sonunda çıkış yapmayan personel için sistem **"eksik kayıt"** uyarısı üretir;
   İK panelden düzeltme yapabilir ve düzeltme **kim tarafından, ne zaman** yapıldığıyla
   birlikte loglanır (denetim izi).
4. **Geç gelme, erken çıkma, fazla mesai**; personelin vardiyasına göre otomatik hesaplanır.
5. Resmî tatiller, yıllık izin, rapor/izin günleri çalışma hesabına işlenir.
6. Aynı QR token **ikinci kez kullanılamaz**; süresi dolmuş QR reddedilir ve deneme loglanır.

---

## 5. Veri Modeli (Ana Tablolar)

| Tablo | Önemli alanlar |
|---|---|
| `personel` | Ad-soyad, sicil no, TC (isteğe bağlı, maskeli), departman, unvan, işe giriş, durum (aktif/pasif) |
| `kullanicilar` | E-posta/telefon, şifre (hash), rol |
| `cihazlar` | Personel ID, cihaz ID (UUID), model, platform, kayıt tarihi, aktif mi |
| `lokasyonlar` | Ad, adres, GPS merkez, izin yarıçapı (m), kiosk bilgisi |
| `vardiyalar` | Ad, giriş-çıkış saatleri, tolerans (dk), ara hakkı, fazla mesai kuralı |
| `personel_vardiya` | Personel ↔ vardiya ataması, geçerlilik tarihleri |
| `hareketler` | Personel, tip (GİRİŞ/ÇIKIŞ/ARA_ÇIKIŞ/ARA_GİRİŞ), zaman, lokasyon, GPS, cihaz, QR token ID, kaynak (mobil/kiosk/panel düzeltme) |
| `izinler` | Personel, tip (yıllık/rapor/ücretsiz), başlangıç-bitiş, onaylayan |
| `tatiller` | Tarih, açıklama |
| `qr_token_log` | Üretilen/kullanılan/ reddedilen token denemeleri (güvenlik takibi) |
| `duzeltme_log` | Kim, kimi, hangi kaydı, eski→yeni değer, açıklama |

---

## 6. Web Panelinden Beklenen Ekranlar ve Raporlar

### Ekranlar
- **Dashboard:** Bugün gelen/gelmeyen/geç kalan sayısı, anlık içeridekiler listesi
- **Personel yönetimi:** Ekleme, düzenleme, toplu içe aktarma (Excel), pasife alma
- **Cihaz yönetimi:** Personelin telefonunu eşleştirme, telefon değişince sıfırlama, kayıp cihazı devre dışı bırakma
- **Lokasyon & vardiya tanımları**
- **Canlı hareket akışı:** Kim, ne zaman, nereden giriş yaptı (anlık liste)
- **Düzeltme/onay ekranı:** Eksik ve hatalı kayıtların İK tarafından düzeltilmesi

### Raporlar
| Rapor | İçerik | Çıktı |
|---|---|---|
| Günlük devam raporu | Kim geldi/gelmedi, giriş-çıkış saatleri | Ekran + Excel |
| Aylık puantaj | Gün gün çalışma, ara toplamı, geç kalma, fazla mesai | Excel + PDF |
| Geç kalma / erken çıkma | Kişi ve departman bazlı listeler | Excel |
| Fazla mesai raporu | Onaylı/onaysız mesai saatleri | Excel |
| Eksik kayıt raporu | Çıkış yapmayanlar, yarım günler | Ekran + Excel |
| Departman özetleri | Departman bazlı devamlılık yüzdeleri | Ekran + PDF |
| **Bordro entegrasyonu** (isteğe bağlı) | Logo, Mikro, Zirve, Luca vb. aktarım formatı | Programa özel export |

---

## 7. Geliştirme Planı (Fazlar)

Takvim günü yerine **faz ve teslim çıktısı** bazında planlama öneriyoruz; her faz
kendi içinde test edilip onaylandıktan sonra sıradakine geçilir.

### Faz 0 — Analiz ve karar (bu raporun devamı)
- Karar noktalarının netleşmesi (Model A/B, QR süresi, lokasyon sayısı, vardiya kuralları)
- Ekran taslakları (wireframe) ve onayınız
- **Çıktı:** Onaylı kapsam dokümanı

### Faz 1 — Çekirdek altyapı ve web paneli (MVP temeli)
- Sunucu, veritabanı, kullanıcı/rol sistemi
- Personel kaydı, departman, lokasyon, vardiya tanımları
- **Çıktı:** Personelin webden kaydedilebildiği çalışan panel

### Faz 2 — QR motoru ve mobil uygulama
- İmzalı, süreli QR token üretimi/doğrulaması (sunucu tarafı)
- Flutter mobil uygulama: giriş, QR okutma/üretme, giriş-çıkış/ara işlemleri,
  cihaz bağlama, GPS (geofence) kontrolü, kendi hareketlerini görme
- Kiosk ekranı (Model A seçilirse): sürekli yenilenen QR gösteren web sayfası
- **Çıktı:** Uçtan uca giriş-çıkış yapılabilen sistem

### Faz 3 — Raporlama ve düzeltme
- Günlük/aylık puantaj, geç kalma, fazla mesai, eksik kayıt raporları
- Excel/PDF export, İK düzeltme ve onay akışları, denetim logları
- **Çıktı:** Bordroya esas aylık puantaj çıkarılabilen sistem

### Faz 4 — Pilot kullanım ve sertleştirme
- Küçük bir personel grubuyla canlı pilot; sahte GPS/ekran görüntüsü senaryoları dahil
  güvenlik testleri; iOS/Android cihaz çeşitliliği testi
- App Store / Google Play yayın süreçleri (Apple geliştirici hesabı gerekir)
- **Çıktı:** Tüm personele açılmaya hazır sürüm

### Faz 5 — (İsteğe bağlı) Genişletmeler
- Bordro programı entegrasyonu, turnike/kapı entegrasyonu, yüz tanıma seçeneği,
  çok şubeli yapı, ziyaretçi modülü

---

## 8. Güvenlik, KVKK ve Yasal Notlar

1. **KVKK:** GPS konumu yalnızca **okutma anında** alınmalı, personel sürekli takip
   edilmemelidir. Bu hem yasal hem de personel güveni açısından kritiktir. Personelden
   aydınlatma metni ile açık rıza alınmalı, veriler amaç dışında kullanılmamalıdır.
2. Şifreler hash'li saklanır; tüm trafik HTTPS'tir; API anahtarları telefonda açık tutulmaz.
3. Telefon değişimi/satışı durumunda İK, panelden eski cihazı tek tıkla devre dışı bırakabilmelidir.
4. Kayıtlar (özellikle düzeltmeler) silinemez olmalı; düzeltme yeni kayıt + log ile yapılır.
   Puantaj uyuşmazlıklarında bu loglar delil niteliği taşır.
5. Çalışma süresi hesapları (ara düşümü, fazla mesai) şirket politikanıza ve İş Kanunu'na
   göre parametrik tanımlanmalıdır; sistem "tek doğru"yu dayatmamalı, kuralı siz girmelisiniz.
6. Günlük otomatik yedek ve farklı lokasyonda yedek kopyası planlanmalıdır.

---

## 9. Riskler ve Dikkat Edilecekler

| Risk | Etki | Önlem |
|---|---|---|
| Ekran görüntüsüyle suistimal | Başkası adına giriş | Kısa ömürlü + tek kullanımlık token, GPS kontrolü |
| Sahte GPS uygulamaları | Evden giriş yapma | Mock-location tespiti, cihaz bütünlük kontrolü (Play Integrity / App Attest) |
| Telefon saati değiştirme | Zaman hilesi | Tüm doğrulama sunucu saatiyle yapılır |
| İnternet kesintisi | Girişte kayıt alınamaması | Mobil uygulamada çevrimdışı kuyruk; internet gelince imzalı kaydı gönderme (kural onayı gerekir) |
| Şarj/telefon yokluğu | Personel kayıt yapamaz | Kiosk/yetkili cihazından manuel kayıt + İK onayı akışı |
| iOS yayın süreci | Mağaza onay gecikmesi | Apple Developer hesabının erken açılması, TestFlight ile erken dağıtım |
| Personel direnci / eğitim | Hatalı kullanım | Pilot grup, kısa kullanım kılavuzu, ilk günlerde destek |

---

## 10. Başlamadan Önce Sizden Netleştirmemizi İstediğimiz Kararlar

Lütfen bu soruları cevaplayın; cevaplar Faz 0'ın çıktısı olacak ve kapsamı kilitleyecektir:

1. **Model:** QR'ı kiosk ekranı mı göstersin (Model A), personelin telefonu mu üretsin (Model B)?
2. **QR yenileme süresi:** 1 saatte ısrar mı, 30–60 saniye önerisini kabul mü?
3. **Kapsam:** Kaç personel, kaç farklı lokasyon/şube, kaç giriş noktası olacak?
4. **GPS zorunluluğu:** Konum kontrolü şart mı, sadece kayıt amaçlı mı tutulsun?
5. **Vardiya yapısı:** Herkes sabit saatte mi çalışıyor; vardiyalı/nöbetli düzen var mı?
   Ara (mola) süreleri maaşa esas çalışma süresinden düşülecek mi?
6. **Raporlama:** Aylık puantaj hangi formatta, hangi bordro/muhasebe programına aktarılacak?
7. **Cihaz politikası:** Personel kendi telefonunu mu kullanacak (BYOD), şirket telefonu mu verilecek?
   Telefonu olmayan personel için alternatif (kart/kiosk PIN) isteniyor mu?
8. **Kimlik doğrulama:** Mobil uygulamaya giriş; e-posta+şifre mi, telefon+SMS kodu mu, personel no+PIN mi?
9. **Onay akışı:** Geç kalma/fazla mesai için amir onayı isteniyor mu?
10. **Barındırma:** Veriler kendi sunucunuzda mı, bulutta mı tutulacak? (KVKK tercihiniz)

---

## 11. Özet ve Önerilen Sonraki Adım

- Sistem; **mobil uygulama + web paneli + merkezi backend** olmak üzere üç parçadır ve
  tamamı tek veritabanı üzerinden haberleşir. Bu yapı hem Android hem iPhone'u tek
  mobil kod tabanıyla (Flutter) karşılar.
- En kritik tasarım kararı QR güvenliğidir: **saatlik değişen QR tek başına yetersizdir**;
  kısa ömürlü, imzalı, tek kullanımlık token + cihaz bağlama + GPS kontrolü önerilir.
- İşe **Faz 0** ile başlanır: Yukarıdaki 10 sorunun cevabı + ekran taslakları onaylanır,
  ardından Faz 1'den itibaren geliştirme yapılır.

**Sonraki adım:** Bölüm 10'daki sorulara vereceğiniz cevaplarla kapsamı kilitleyip,
ekran taslaklarını ve teknik tasarım detayını (API sözleşmesi, veritabanı şeması)
hazırlamamızdır. Onayınız hâlinde geliştirmeye Faz 1'den başlanabilir.

---

*Bu rapor planlama amaçlıdır. Kapsam, Bölüm 10'daki kararlar netleştikten sonra
kesinleşir; kararlardaki değişiklikler faz içeriklerini etkileyebilir.*
