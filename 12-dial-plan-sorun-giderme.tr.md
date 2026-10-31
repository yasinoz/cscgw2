# Discovery 12 — İzleme Araçlarıyla Dial Plan Sorun Giderme

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe; araç adları,
> desen ve değerler **birebir**. Süre: **40–50 dk**.
> **Ön koşul:** Dial plan, Discovery 8 (Calling Privileges) üzerine kuruludur.

---

## Giriş

Dial plan arızaları (yanlış CSS, çakışan desen, hatalı translation pattern)
çağrı bağlantısını bozar. CUCM bunları teşhis için iki güçlü araç verir:
- **DNA (Dialed Number Analyzer):** Bir numara çevrildiğinde CUCM'in **hangi
  deseni/partition'ı seçtiğini**, hangi dönüşümleri uyguladığını ve çağrının
  nereye gittiğini (ya da neden engellendiğini) adım adım gösterir. Gerçek
  çağrı kurmadan "kağıt üzerinde" analiz.
- **RTMT (Real-Time Monitoring Tool):** Canlı çağrı kayıtlarını (CDR/Real Time
  Data) ve **termination cause code**'larını gösterir; gelen çağrı arızalarını
  ve SIP merdiven diyagramını incelersin.

## Ortam

HQ-UCM (v15), PC-1 (HQ-CIPC-1/11001), PC-2 (HQ-CIPC-2/11002). PSTN access `9`.
ROUTER-1 SIP trunk, PC-4 PSTN simülatörü. Reach: local `408[2-9]XXXXXX`,
national `1[2-9]XX[2-9]XXXXXX`, intl `011!`, emergency `911`.

---

## Görev 1: Uç Noktaları Kaydet ve Doğrula

PC-1 → **lab12.py** → "successfully loaded". CIPC-1/CIPC-2 kaydet.

## Görev 2: Telefon Arama Sorununu DNA ile Teşhis Et

**Senaryo:** Müşteri "CIPC-1 şehirlerarası (national) arayamıyor" diyor.

**1–3. Sorunu doğrula ve kapsamını daralt:**
- CIPC-1'den national `914554551000` ara → **Reorder** (başarısız, **beklenmedik**).
- CIPC-1'den international `9011...#` ara → başarısız (**beklenen**, izinli değil).
- CIPC-2'den national `914554551000` ara → **çalışır** → sorun tek telefonda
  (genel bir arıza değil).

**4–8. DNA analizi (çalışmayan telefon):** **Cisco Unified CM Administration >
Device > Phone > HQ-CIPC-1** → üstteki **Analysis** (DNA) ikonu:
- **Association Information:** `Line [1] – 11001 in LINE_INTERNAL_MANAGER_PT` seç
- **Analyzer Input > Dialed Digits:** `914554551000` (ya da `911`) → **Do Analysis**

DNA çıktısı çevrilen numaranın hangi route pattern/partition ile eşleştiğini,
uygulanan CSS'i ve sonucu gösterir. Burada **Match Result** ya da eşleşen
desen, hattın CSS'inin national'a izin vermediğini ortaya koyar (kök neden:
CIPC-1 Line 1'in CSS'i yanlış/kısıtlı).

**Karşılaştır (çalışan telefon):** HQ-CIPC-2 → `Line [1] – 11002 in
LINE_INTERNAL_PT` → `914554551001` → **Do Analysis**. İkisini yan yana koyunca
fark (CSS/partition) netleşir.

**Düzelt:** CIPC-1 Line 1'in **Calling Search Space**'ini national'a izin veren
doğru CSS'e çevir → **Save** → **Apply Config**. National arama artık çalışır.

## Görev 3: Çakışan Desen ve Interdigit Sorunu (DNA + Urgent Priority)

Çakışan route pattern'ler çevirme sonrası gecikme (interdigit timeout) yaratır.
DNA ile çakışmayı görüp düzeltirsin.

**911 acil için Urgent Priority:** **Call Routing > Route/Hunt > Route Pattern
> `911`** (partition `DEVICE_HEADQUARTERS_PT`) → ☑ **Urgent Priority** → **Save**.
Böylece `911` tam eşleşince CUCM beklemez, anında yönlendirir.

**Çakışan deseni sil:** **Route Pattern > Find** → `911` ile çakışan
`9.1XXXX` desenini işaretle → **Delete Selected**. (Çakışma kalkınca `911`
gecikmesiz çalışır.)

> DNA'da acil numarayı analiz ederek, hangi desenin "önce" eşleştiğini ve
> gecikmeye yol açtığını görebilirsin.

## Görev 4: Gelen Çağrı Arızasını RTMT ile Teşhis Et

**Senaryo:** PSTN'den gelen bir çağrı başarısız. Canlı kayıtlara bakalım.

**1–2.** PC-4'ten (PSTN) başarısız çağrıyı tetikle (ör. CIPC-2'yi ara).
**Cisco Unified Real-Time Monitoring Tool** → Host `10.1.5.5`,
`admin`/`!234QWer`.

**3. CDR/Real Time Data:** İlgili bölümden **Real Time Data** aç → Search
Criteria:
- **Start Time:** test çağrının **UTC** saati (CUCM'in bulunduğu dilim)
- **Duration:** `60` · **Time Zone:** `(GMT +0:0) Coordinated Universal Time-UTC`
→ **Run**.

**4.** Dönen kayıtlarda **Termination Cause Code = (1) Unallocated
(unassigned) number** olan çağrı **başarısız** olandır. (Başarılı çağrı
`(16) Normal call clearing` gösterir.)

**5.** O kaydı aç → **SIP merdiven diyagramı**: PSTN'den gelen ilk **INVITE**
sonrası pek bir şey olmadığını, çağrının kurulamadığını görürsün → numara
CUCM'de karşılık bulmuyor.

## Görev 5: Translation Pattern'i Doğrula (DNA ile)

Gelen çağrının neden "unallocated" olduğunu DNA ile kesinleştir.

**DNA — gelen çağrı:** **Analysis** → **Calling Party:** `4085551234`,
**Called party:** `4085551002` → **Do Analysis**:
- **Match Result = `BlockThisPattern`** ve **Call Flow: Information Not
  Available** → veritabanında eşleşme yok/engelli; sorun **11002 ile ilgili
  bir translation pattern**.

**Karşılaştır (çalışan):** `4085551001` analiz edilince DNA sağlıklı bir akış
gösterir: `TranslationPattern Pattern=4085551001` (partition
`GWY_HEADQUARTERS_PT`) → Called Number `11001` → `Directory Number DN=11001`
(partition `LINE_INTERNAL_MANAGER_PT`) → Device `HQ-CIPC-1`.

**Düzelt:** 11002 için gelen çağrıyı iç DN'e çeviren translation pattern'i
oluştur/düzelt (çalışan 11001 örneğini model al) → gelen PSTN çağrısı CIPC-2'ye
ulaşır.

---

## Doğrulama (lab tamam)

- CIPC-1 CSS düzeltildi; national arama çalışıyor (DNA ile doğrulandı).
- 911 Urgent Priority + çakışan desen silindi; acil çağrı gecikmesiz.
- RTMT'de başarısız gelen çağrı termination cause (1) ile bulundu.
- 11002 translation pattern'i düzeltildi; gelen PSTN çağrısı ulaşıyor.

## Sık Karşılaşılan Sorunlar

| Belirti | Araç / teşhis |
|---|---|
| Telefon belli sınıfı arayamıyor | **DNA**: hattın CSS'i ve eşleşen desen; yanlış CSS'i düzelt. |
| Çevirme sonrası gecikme | **DNA**: çakışan daha uzun desen; **Urgent Priority** / deseni sil. |
| Gelen PSTN çağrısı düşüyor | **RTMT** Real Time Data: termination cause **(1)** = unallocated. |
| "BlockThisPattern" / akış yok | **DNA**: eşleşme yok ya da engel; translation pattern eksik/yanlış. |
| Hangi gateway/pattern seçildi | **DNA** çıktısında TranslationPattern/Route Pattern + partition. |

## Eğitmen Notu

İki aracı rolüne göre öğretin: **DNA = "çağrı kurulmadan kağıt üzerinde analiz"**
(hangi desen, hangi CSS, hangi dönüşüm, sonuç), **RTMT = "canlı ne oldu"**
(termination cause, SIP merdiveni). Sorun giderme refleksini verin: önce
**kapsamı daralt** (tek telefon mu, herkes mi), sonra **DNA ile kök neden**,
gerekirse **RTMT ile canlı doğrulama**. `BlockThisPattern` ve termination
cause `(1)` iki klasik parmak izidir. Bu lab Discovery 8'in dial plan'ı
üzerine kuruludur; önce onu yapmış olmak gerekir.

---

### Sonraki

**Discovery 13: Session Border Controller (CUBE) Uygulama** — CUBE/SBC
yapılandırması, SIP trunk, dial-peer'lar ve PSTN entegrasyonu.
