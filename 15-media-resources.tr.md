# Discovery 15 — Media Resources (Medya Kaynakları)

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe; menü/alan
> adları ve değerler **birebir**. Süre: **50–60 dk**.
> **Ön koşul:** Discovery 1 tamamlanmış olmalı (CA sertifikası
> **CallManager-trust**'ta kurulu olmalı).

---

## Giriş

Güvenilir bir kullanıcı deneyimi çağrı yönlendirmeyle bitmez; **medya
kaynakları** da doğru kurulmalı: **MOH** (müzik/sessizlik beklemede),
**Annunciator (ANN)** (sesli anonslar), **Conference Bridge (CFB)**
(konferans), **MTP/Transcoder** (codec/medya uyumu). Yanlış yapılandırılırsa
kullanıcı beklemede ton, eksik anons ya da konferans arızası yaşar.

Bu lab'da yazılım medya servislerini aktive eder, MOH kaynaklarını
yapılandırır, Meet-Me ve ad-hoc konferansı test eder ve **MRG/MRGL** ile
cihazların medya kaynaklarına erişimini önceliklendirirsin.

## Ortam

HQ-UCM `https://10.1.5.5/ccmadmin` ve `/ccmservice` (`admin`/`!234QWer`).
Üç telefon: HQ-CIPC-1 (PC-1), HQ-CIPC-2 (PC-2), HQ-CIPC-3 (PC-3).

---

## Görev 1: Config Yükle ve Kaydet

PC-1 → **lab15.py** → "successfully loaded". Üç CIPC'yi kaydet.

## Görev 2: Yazılım Medya Servisini Aktive Et

**Cisco IP Voice Media Streaming App** servisi; MOH, Annunciator, yazılım
Conference Bridge ve yazılım MTP'yi **tek başına** sağlar.

**Önce sorunu gör:**
- **Media Resources > Music On Hold Server > Find:** `MOH_2` durumu **None /
  Unregistered**.
- **Media Resources > Annunciator > Find:** ANN **None / Unregistered**.
- HQ-CIPC-1'den geçersiz bir DN ara (ör. `11020`) → kayıtlı anons yerine
  **Reorder** tonu (annunciator çalışmıyor; "Your call cannot be completed as
  dialed" anonsu yok).

**Aktive et:** **Cisco Unified Serviceability > Tools > Service Activation**
→ Server **CUCM-PUB...—CUCM Voice/Video** → **Cisco IP Voice Media Streaming
App**'i işaretle → **Save**. Birkaç dk sonra MOH Server ve Annunciator
**Registered** olur; `11020` araması artık **sesli anons** verir.

## Görev 3: MOH Ses Kaynaklarını Yapılandır

**Media Resources > Music On Hold Audio Source.**

**Sessizlik kaynağı oluştur (Add New):**
- **MOH Audio Stream Number:** `4`
- **MOH Audio Source Name:** `NoMOH`
- **Use MOH WAV file source** · **MOH Audio Source File:** `SilenceAudioSource`
- **Save**

> Bazı senaryolarda (kayıt, uyumluluk) beklemede **müzik yerine sessizlik**
> istenir; bu kaynak onu sağlar. Özel müzik için **Upload File** ile WAV yükle.

**Sunucuya uygula:** Related Links **Music On Hold Server Configuration** →
**Go** → **Find** → **MOH_2**'yi işaretle → **Apply Config to Selected** →
**OK**. (Ses kaynağı değişiklikleri sunucuda uygulanınca etkinleşir.)

## Görev 4: MOH Davranışını Ata (Common Device Config + override)

- **Küme/varsayılan:** Service Parameters'ta veya device pool'da varsayılan MOH
  kaynağı (user hold / network hold) atanır.
- **Grup bazında:** **Common Device Configuration** ile bir cihaz grubuna
  belirli MOH kaynağını (ör. `NoMOH`) uygula.
- **Cihaz bazında override:** Telefon yapılandırmasında (ya da hattında) MOH
  kaynağı cihaz düzeyinde ezilebilir.

> Hiyerarşi: **cihaz > common device config > device pool > küme varsayılanı**.
> Kullanıcı "beklemede yanlış müzik/ton" diyorsa bu zincire bak.

## Görev 5: Meet-Me ve Ad-Hoc Konferans

**Meet-Me (randevulu konferans):**
1. **Call Routing > Meet-Me Number/Pattern > Add New:** Meet-Me numarası/aralığı
   (ör. `11980`), partition uygun → **Save**.
2. Telefona **Meet Me Conference** düğmesi ekle: **Device > Phone > HQ-CIPC-1 >
   Modify Button Items** → boş bir düğmeye (ör. button 3) **Meet Me
   Conference** → **Save** → **Apply Config**.
3. **Test:** HQ-CIPC-1'de **NewCall** (2. softkey) → **Meet-Me** (button 3) →
   `11980` çevir → telefon **"To Conference (11980)"** gösterir. Diğer
   telefonlar aynı `11980`'i arayarak katılır.

**Ad-hoc konferans:** Aktif çağrıda **Confrn** → yeni numarayı ara → tekrar
**Confrn** ile birleştir (konferans köprüsü CFB kaynağı kullanılır).

## Görev 6: Media Resource Group (MRG) ve MRG List (MRGL)

MRG, medya kaynaklarını (CFB/MTP/MOH/ANN) **gruplar**; MRGL bu grupları
**sıralı** bir liste yapar. Cihaz, MRGL'deki sırayla ilk uygun kaynağı seçer.

**1. MRG:** **Media Resources > Media Resource Group > Add New** → ad ver,
istenen kaynakları (ör. yazılım CFB, MTP, MOH, ANN) **Selected**'a taşı →
**Save**. (Donanım ve yazılım kaynakları ayrı MRG'lerde toplanıp
önceliklendirilebilir.)

**2. MRGL:** **Media Resources > Media Resource Group List > Add New** → ad ver,
MRG'leri **sıraya** koy (önce tercih edilen) → **Save**.

**3. Uygula:**
- **Device Pool** düzeyinde (grup geneli) → **System > Device Pool >** ilgili
  pool → **Media Resource Group List** ata.
- **Telefon** düzeyinde (override) → **Device > Phone >** MRGL ata.

**4. Doğrula (RTMT):** **Cisco Unified Real-Time Monitoring Tool** ile bir
konferans/hold sırasında hangi medya kaynağının seçildiğini ve kaynak
kullanımını izle.

> MRGL **yoksa** cihaz **Default** (Null) MRG'yi kullanır — tüm kaynaklara
> erişir. MRGL ile erişimi kısıtlar/önceliklendirirsin (ör. şube yalnızca
> yerel MTP'yi kullansın).

---

## Doğrulama (lab tamam)

- Cisco IP Voice Media Streaming App aktif; MOH Server ve Annunciator Registered.
- Geçersiz DN'de reorder yerine sesli anons duyuluyor.
- NoMOH (sessizlik) kaynağı oluşturuldu ve MOH sunucusuna uygulandı.
- Meet-Me (11980) ve ad-hoc konferans çalışıyor.
- MRG/MRGL oluşturulup device pool/telefona uygulandı; RTMT ile doğrulandı.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| Beklemede ton (müzik yok) | MOH Server kayıtlı değil (IP Voice Media Streaming aktif mi) ya da MRGL'de MOH yok. |
| Geçersiz numarada anons yok | Annunciator kayıtlı değil; servisi aktive et. |
| Konferans kurulamıyor | CFB kaynağı yok ya da cihazın MRGL'inde yer almıyor. |
| Yanlış/eksik MOH kaynağı | Common Device Config / cihaz override / device pool zincirini kontrol et. |
| Meet-Me "conference full/not available" | Meet-Me numarası tanımlı mı, CFB kapasitesi var mı? |

## Eğitmen Notu

Tek servisin (Cisco IP Voice Media Streaming App) dört kaynağı birden
sağladığını vurgulayın: MOH, ANN, yazılım CFB, yazılım MTP. MRG/MRGL
mantığını partition/CSS'e benzetin: **MRG = kaynak kutusu, MRGL = bakılacak
kutular sırası, yoksa Default (hepsi).** MOH kaynak seçim hiyerarşisini
(cihaz > common device config > device pool > küme) tahtaya yazın. Meet-Me
(randevulu, sabit numara) ile ad-hoc (anında, Confrn) farkını gösterin.
Ön koşulu hatırlatın: Discovery 1 (CallManager-trust'ta CA) yapılmamışsa MOH
sunucusu düzgün kaydolmayabilir.

---

### Sonraki

**Discovery 16: Cisco Unity Connection'ı CUCM ile Entegre Et** — sesli mesaj
portları, SIP trunk, voicemail pilot/profile ve MWI.
