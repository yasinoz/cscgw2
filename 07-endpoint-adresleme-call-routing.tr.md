# Discovery 7 — Uç Nokta Adresleme ve Çağrı Yönlendirme

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe; menü/alan
> adları ve değerler **birebir**. Süre: **30–45 dk**.

---

## Giriş

Bu lab'da sadece telefon kaydetmekle kalmaz, çağrının ağda **nasıl aktığını**
şekillendirirsin: paylaşımlı hatlar, Directory URI'ler ve çakışan dial plan'ın
gerçek çağrı davranışına etkisi.

**İki kavramı ayır:**
- **Directory Number (DN):** Mantıksal numara — yönlendirme, yönlendirme
  (forward) ve sesli mesaj davranışını tanımlar. Sıfır veya daha çok cihaza
  atanabilir; birden çok cihazdaysa **paylaşımlı hat (shared line)** olur.
- **Line Appearance:** O DN'in bir telefondaki **fiziksel görünümü** — hangi
  düğmede, nasıl göründüğü. Aynı DN'in farklı cihazlarda farklı line
  appearance'ları olabilir.

## Ortam

CUCM `https://10.1.5.5/ccmadmin` (`admin`/`!234QWer`). Telefonlar:
**CIPC-1** (PC-1) 11001, **CIPC-2** (PC-2) 11002, **CIPC-3** (PC-3) 11003.

---

## Görev 1: Config Yükle ve Uç Noktaları Kaydet

PC-1 → **lab7.py** → "successfully loaded". Üç PC'de CIPC'leri kaydet
(CIPC-1/2/3 → 11001/11002/11003, Registered).

## Görev 2: DN ile Line Appearance Farkı (Paylaşımlı Hat)

CIPC-2 ve CIPC-3'e **aynı DN'i** vererek paylaşımlı hat kuracağız; aynı
numaraya gelen çağrıyı farklı telefonlarda yanıtlayınca **farklı caller ID**
göreceğiz.

**1–7.** **Device > Phone > CIPC-2** → **Line [2] – Add a new DN**:
- (Önce cihaza özgü parametrelerin boş olduğuna dikkat et — "Line 2 on Device
  CIPC-2" başlığı altında.)
- **Directory Number:** `11009` → başka alana tıkla → sayfa yenilenir →
  **Save**. Artık CIPC-2, 11009'un line appearance'ına sahip.

**Paylaşımlı hat yap:** Aynı `11009` DN'ini **CIPC-3**'e de bir line olarak
ekle. İki cihaz aynı DN'i paylaşır. 11009'a çağrı gelince ikisinde de çalar;
cihaza özgü **Display (Caller ID)** / **Line Text Label** ile her telefonda
farklı görünebilir.

> DN **numaranın kendisidir**; line appearance o numaranın **nerede/nasıl**
> göründüğüdür. Tek DN → çok line appearance = paylaşımlı hat.

## Görev 3: Directory URI'leri Yapılandır

URI, e-posta benzeri biçimde arama sağlar (`ad@alan`). İki yöntem:

**A) Doğrudan DN'e URI:** **Call Routing > Directory Numbers > Find > 11003**
→ **Directory URIs** bölümü → **URI:** `dgrey@cll-collab.internal`,
**Primary** işaretli → **Save**.

**B) End User üzerinden URI:** **User Management > End User > jdoe** →
**Device Associations** → **User Device Association**'da **CIPC-1**'i işaretle
→ **Save Selected/Changes** → **Directory Number Associations > Primary
Extension:** `11001` → **Save**.
Sonuç: **Call Routing > Directory Numbers > 11001** → **Directory URIs**'de
**`jdoe@cll-collab.internal`** otomatik **primary** URI olarak görünür.

> **Not:** URI bir line appearance parametresi **değildir**; DN'e aittir.
> Kullanıcı-atamalı URI silinemez, partition'ı (`Directory URI`) değiştirilemez
> ve her zaman primary URI olur. Primary URI, o hattan yapılan çağrılarda
> calling URI olarak gönderilir.

## Görev 4: Çakışan DN'ler ve Urgent Priority

Rakam-rakam (digit-by-digit) adresleme **çevirme sonrası gecikme** yaratır;
en bloc adresleme ve **Urgent Priority** bunu çözer.

**Çakışan, daha uzun bir desen:** **CIPC-3 > Line 2** DN'ini `11009` →
**`110019`** yap → **Save**. İlk dört hane 11001 (CIPC-1) ile çakışır, yeni
numarada bir hane fazla.

**1. Rakam-rakam analizi (gecikmeyi gör):** CIPC-2'de **New Call** → `11001`
çevir. CIPC-1, son haneden **10 sn sonra** çalar — çünkü CUCM "bir hane daha
gelip 110019 mu olacak?" diye **interdigit timeout**'u bekler (lab'da 10 sn;
varsayılan 15). **EndCall** ile düşür.

**2. En bloc analizi (translation pattern ile):** CIPC gerçek en bloc
göndermez; bir **translation pattern** ile üretiriz.
**Call Routing > Translation Patterns > Add New**:
- **Translation Pattern:** `11000`
- ☑ **Do Not Wait For Interdigit Timeout On Subsequent Hops**
- **Called Party Transform Mask:** `11001` → **Save**.

Artık `11000` çevrilince CUCM onu `11001`'e dönüştürüp **en bloc** yeni çağrı
kurar — beklemeden bağlanır.

**3. Urgent Priority (en temiz çözüm):** **Call Routing > Directory Numbers >
11001** → ☑ **Urgent Priority** → **Save**. Bu işaretliyken desen tam
eşleşince CUCM **interdigit timeout'u beklemez**, çağrıyı anında yönlendirir —
çakışma olsa bile 11001 gecikmesiz çalışır.

---

## Doğrulama (lab tamam)

- 11009 paylaşımlı hattı CIPC-2 ve CIPC-3'te çalıyor; caller ID cihaza göre.
- 11003'te doğrudan URI, 11001'de end-user URI (jdoe@cll-collab.internal).
- Çakışan 110019 varken 11001 digit-by-digit'te 10 sn gecikiyor;
  translation pattern (en bloc) ve **Urgent Priority** gecikmeyi kaldırıyor.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| Çağrı 10-15 sn gecikiyor | Çakışan daha uzun desen var; interdigit timeout bekleniyor. Urgent Priority ekle. |
| Paylaşımlı hatta caller ID aynı | Cihaza özgü **Display (Caller ID)** / **Line Text Label** gir. |
| URI ile arama çözülmüyor | URI primary mi, partition/CSS doğru mu, ILS gerekiyorsa advertise edildi mi? |
| User-atamalı URI silinemiyor | Normaldir; end user primary extension'dan gelir, DN'de elle silinmez. |

## Eğitmen Notu

"DN = numara, line appearance = o numaranın telefondaki yüzü" ayrımını tahtada
oturtun; paylaşımlı hat buradan çıkar. Urgent Priority'nin "neden" çalıştığını
anlatın: çakışan daha **uzun** bir desen varsa CUCM bir hane daha gelecek mi
diye bekler; Urgent Priority "bu desen tamam, bekleme" der. Üç çözümü
karşılaştırın: numarayı değiştir (çakışmayı kaldır), en bloc (translation
pattern), Urgent Priority (en hedefli).

---

### Sonraki

**Discovery 8: Çağrı Ayrıcalıkları (Calling Privileges)** — Partition, Calling
Search Space (CSS), Time-of-Day routing ve arama sınıfı mantığı.
