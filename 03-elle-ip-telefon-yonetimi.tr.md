# Discovery 3 — Elle IP Telefon Yönetimi

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Fast Lane Lab Guide'ın yanında kullanılır. Adımlar ve "neden"ler
> Türkçe; menü/düğme adları, komutlar, dosya adları, IP'ler ve kimlikler
> ekranda göründüğü gibi **birebir** bırakıldı. Tahmini süre: **60–75 dk**.

---

## Giriş — Ne Yapıyoruz, Neden?

IP telefon ve softphone'lar CUCM'deki iyi tanımlı yapılandırmalara dayanır:
doğru cihaz profili, TFTP ayarı ve standart şablonlar olmadan Cisco IP
Communicator kayıt olamaz, yanlış özellikleri gösterir ya da desteklenmeyen
firmware çalıştırır.

Bu lab'da bir CUCM yöneticisi gibi telefonları **elle** sağlarsın (provision),
doğrular ve sorun giderirsin: elle ekleme → düğme/softkey şablonları →
toplu işlem (BAT) → config temizleme → telefon web arayüzü → firmware yükleme.

> **Config yükleme notu:** Betik/config adlarının numarası Discovery
> numarasıyla örtüşmeyebilir; adımlarda yazan betiği çalıştır.

## Ortam (Job Aid)

| Cihaz | IP | Kullanıcı | Parola |
|---|---|---|---|
| PC-1 | 10.1.5.200 | student | 1234QWer |
| PC-2 | 10.1.5.201 | student | 1234QWer |
| HQ-UCM | 10.1.5.5 | admin | !234QWer |
| WIN-SERVER01 | 10.1.5.100 | administrator | 1234QWer |

Telefonlar: **HQ-CIPC-1 = 11001** (John Doe, PC-1), **HQ-CIPC-2 = 11002**
(Jane White, PC-2). Alan adı `cll-collab.internal`.

---

## Görev 1: Config Yükle

PC-1 masaüstünde **Classfiles > Discoveries** → **lab3.py**'ye çift tıkla.
Son satır **"Lab configuration successfully loaded"** olmalı. Enter ile kapat.

---

## Görev 2: Cisco IP Communicator'ı Elle Ekle

**1–7. CIPC tarafı (PC-1):** Cisco IP Communicator'ı aç → **menu button**
→ **Preferences…** → **Network** sekmesi:
- **Use this Device Name:** `HQ-CIPC-1`
- **TFTP Servers → Use these TFTP servers → TFTP Server 1:** `10.1.5.5`
- **OK**

> **SEP adı:** Gerçek cihaz adını (`SEP005056...`) görmek için geçici olarak
> "Use Network Adapter to generate Device Name"a bakabilirsin; ama bu lab'da
> adı elle `HQ-CIPC-1` veriyoruz.

**8.** PC-2'de aynısını `HQ-CIPC-2` ile yap.

**9–12. Telefon durumunu oku:** CIPC'de **Settings** → **4 Status** →
**1 Status Messages**. Henüz telefon CUCM'de tanımlı olmadığı için
**XMLDefault.cnf.xml** kullanılıyor ve **SEP….cnf.xml File Not Found** +
**CTL update failed** görürsün. (Normal — telefonu şimdi ekleyeceğiz.)

**13. (İsteğe bağlı) Varsayılan config'i gör:** Tarayıcıda
`http://10.1.5.5:6970/XMLDefault.cnf.xml` (CIPC HTTPS desteklemez). İçinde
`callManager` adı `CUCM-PUB.CLL-COLLAB.INTERNAL`, `sipPort 5060` gibi
değerleri görürsün.

**14–19. CUCM'de telefonu ekle:** `https://10.1.5.5/ccmadmin`
(`admin` / `!234QWer`) → **Device > Phone > Add New** →
**Phone Type: Cisco IP Communicator** → Next → **protocol: SIP** → Next:
- **Device Name:** `HQ-CIPC-1`
- **Description:** `Desktop 1 HQ SIP CIPC`
- **Device Pool:** `HQ`
- **Phone Button Template:** `Standard CIPC SIP`
- **Owner:** `Anonymous (Public/Shared Space)` *(sistemde kullanıcı yok)*
- **Device Security Profile:** `Cisco IP Communicator - Standard SIP Non-Secure Profile`
- **SIP Profile:** `Standard SIP Profile`

**20.** **Save** → OK.

**21–23. Dizin numarası (DN) ekle:** Sol üstte **Line [1] – Add a new DN**:
- **Directory Number:** `11001`
- **Description:** `Desktop 1 HQ SIP CIPC Line 1`
- **Alerting Name:** `John Doe` *(çağrıda numara yerine isim görünür)*
- **Save** → Related Links **Configure Device (HQ-CIPC-1)** → **Go**

**24.** **Apply Config** → OK.

**25–26.** CIPC'yi öne al; takılırsa **Exit** x3 ya da Settings'te `**#**`.
Telefon **11001** olarak kaydolur; Status Messages'ta artık
**HQ-CIPC-1.cnf.xml** kullanılıyor.

**27–30.** İkinci telefonu ekle: `HQ-CIPC-2` / `Desktop 2 HQ SIP CIPC`,
DN `11002` / Alerting Name `Jane White`. 11001'den **11002**'yi ara, PC-2'de
yanıtla — çağrı çalışır.

---

## Görev 3: Telefon Düğme Şablonları (Phone Button Template)

Düğme şablonu, telefonun sağındaki düğmelerin ne olacağını tanımlar.
Varsayılan 2 hat + hızlı aramadır; biz **3 hat** yapacağız.

**2–5. Şablonu kopyala:** **Device > Device Settings > Phone Button
Template** → arama **contains** `CIPC` → **Find** → **Standard CIPC SIP**
satırının **kopya ikonu** → ad: `LearningAtCisco - CIPC SIP` → **Save**.

**6.** **Button 3**'ü `Line` yap → **Save**. (Button 1 her zaman Line'dır.)

**7–11. Telefona ata:** **Device > Phone > HQ-CIPC-1** → **Phone Button
Template:** `LearningAtCisco - CIPC SIP` → **Save** → **Apply Config** →
**Reset**.

**12.** Artık **Line [3]** eklenebilir durumda (1 = 11001, 2 ve 3 boş).

**13.** Aynısını **HQ-CIPC-2** için tekrarla.

---

## Görev 4: Softkey Şablonları

Softkey şablonu, ekran altı düğmelerin **çağrı durumuna göre** ne
göstereceğini belirler (On Hook ≠ Connected).

**2–5. Şablonu kopyala:** **Device > Device Settings > Softkey Template**
→ **Find** → **Cisco User with Feature Hardkeys** satırını **Copy** →
- **Name:** `LearningAtCisco - User with Feature Hardkeys`
- **Description:** `L@C Standard template for phones - features (Hold, Transfer, etc.) on hardkeys`
- **Save**

**6–7. Düzeni ayarla:** Related Links **Configure Softkey Layout** → **Go**.
Call state **On Hook** → sol listeden **Immediate Divert (iDivert)**'i sağ
oka tıklayarak ekle → **Save**.

> **iDivert**: çağrıyı anında sesli mesaja yönlendirir; yönlendirince hat
> boşalır. Bu lab'da sesli mesaj yok ama düğmeyi ileride kullanmak için ekliyoruz.

**8–12. Connected durumu:** Call state **Connected** seç → **Hold (Hold)**'u
sağ listeye ekle, **yukarı ok** ile en üste taşı; **Undefined**'ı **aşağı ok**
ile en alta taşı → **Save**. (Hold yalnızca Connected'ta geçerlidir.)

**13–17. Telefona ata:** **Device > Phone > HQ-CIPC-1** → **Softkey
Template:** `LearningAtCisco - User with Feature Hardkeys` → **Save** →
**Apply Config**. HQ-CIPC-2 için tekrarla.

**18–20.** Telefon listesinde iki CIPC'yi işaretle → **Reset Selected** →
**Restart** (Reset değil; Restart telefonu baştan başlatmaz, config'i yeniden
indirir). CIPC'de **iDivert** softkey'inin göründüğünü doğrula.

---

## Görev 5: Toplu İşlem — Bulk Administration Tool (BAT)

BAT ile birden çok telefonu tek seferde değiştirirsin. Burada iki telefonun
**device pool**'unu toplu değiştireceğiz.

**1.** **Device > Phone**: ikisinin de device pool'u şu an `HQ`.

**2–6. Servisi etkinleştir:** **Cisco Unified Serviceability** → **Tools >
Service Activation** → Server **CUCM-PUB.CLL-COLLAB.INTERNAL—CUCM
Voice/Video** → **Go** → **Database and Admin Services** altında **Cisco
Bulk Provisioning Service**'i işaretle → **Save** → durum **Activated**.

**7–12. Toplu güncelle:** **Cisco Unified CM Administration**'a dön →
**Bulk Administration > Phones > Update Phones > Query** → **Find** →
**Next** → **Device Pool** kutusunu işaretle, `BRANCH 1` seç →
**Apply Config** radyo düğmesi → **Submit**.

**13–14. İşi çalıştır:** **Bulk Administration > Job Scheduler** → **Find**
→ işi işaretle → **Activate Selected**. İş **Completed / Success** olmalı
(olmazsa yenile).

**15.** **Device > Phone > Find**: iki telefonun device pool'u artık
**BRANCH 1**.

---

## Görev 6: Telefon Config Dosyasını Temizle (CTL)

En sık bootup sorunlarından biri **yanlış CTL dosyası**: telefon bir
kümeden başka kümeye taşınınca CTL uyuşmazlığı kaydı engeller. Çözüm:
CTL'yi telefondan silmek.

**1.** PC-1 CIPC → **Settings** → **5 Security Configuration**.

**2.** **4 CTL File** (`10.1.5.5`). Config **kilitli** (sağ üstte kapalı
kilit); değiştiremezsin.

**3.** Kilidi aç: telefon tuş takımında **`* * #`** bas. ~1 sn sonra kilit
açılır ve **Erase** softkey'i belirir.

> 78XX/88XX/98XX telefonlarda menü farklıdır; bunlarda silmeden önce kilit
> açmak gerekmez.

**4. Erase** → telefon config dosyasını siler, yeniden başlar ve yeni CTL
dosyasını alır.

---

## Görev 7: Telefon GUI'sine Tarayıcıdan Eriş

Telefon ayarlarına telefon menüsü dışında **yerleşik web sunucusundan** da
bakabilirsin; durum ve hata mesajları sorun gidermede işe yarar.

**1–3.** **Find and List Phones** → **IPv4 Address** sütunundaki IP'ye
(HQ-CIPC-1 → `10.1.5.200`) tıkla. **Device Information** sayfası:
- Host Name `HQ-CIPC-1`, Phone DN `11001`, Version `8.6.0`,
  Model `Cisco Communicator`.

**4. Network Configuration:**
- DHCP Server, Host Name `HQ-CIPC-1`, IP `10.1.5.200`,
  TFTP Server 1 `10.1.5.5`, Default Router 1 `10.1.5.250`,
  Unified CM 1 `CUCM-PUB.CLL-COLLAB.INTERNAL  Active`.

> Sol menüde **Status Messages**, **Device Logs**, **Streaming Statistics**
> de var — çağrı kalitesi ve kayıt sorunlarında ilk bakılacak yer.

---

## Görev 8: CUCM'e Telefon Firmware'i (Device Package) Yükle

Firmware dosyaları genelde bir **SFTP/FTP** sunucusunda tutulur; bu
"device package" dosyaları telefonlara dağıtılmadan önce CUCM'e kurulmalı.

**1–2.** PC-1 masaüstünde **Firmware** klasörü → **cmterm-PHONEOS** dosyasına
sağ tık → **Copy**. (Tam ad: `cmterm-PHONEOS.3-2-1-0003-28.cop.sha512`.)

**3–6. FTP sunucusuna koy:** **Remote Desktop Connection** → `10.1.5.100`
(`administrator` / `1234QWer`) → **C:\firmware\upgrade** → sağ tık **Paste**
→ RDP'yi kapat.

**7–9. CUCM'e kur:** `https://10.1.5.5/cmplatform` (`admin` / `!234QWer`)
→ **Software Upgrades > Install/Upgrade**:
- **Source:** `Remote Filesystem`
- **Directory:** `/upgrade`
- **Server:** `10.1.5.100`
- **User Name:** `anonymous` · **User Password:** `anonymous`
- **Transfer Protocol:** `FTP`
- ☑ **continue with upgrade after download** → **Next**

**8–9.** **Options/Upgrades** açılır menüsünde
`cmterm-PHONEOS.3-2-1-0003-28.cop.sha512`'i seç → **Next** → ~8 dk bekle →
durum **Complete**.

**12–13. Doğrula:** **CM Administration > Device > Device Settings >
Device Defaults**: yeni **Load Information** (`PHONEOS.3-2-1-0003-28`)
9841/9851/9861/9871 modellerinde görünür.

**14–16. TFTP'yi yeniden başlat:** **Cisco Unified Serviceability > Tools >
Control Center—Feature Services** → publisher seç → **Cisco TFTP** servisini
**Restart**. (Yeni telefon yükleri ancak TFTP restart'ından sonra dağıtılır.)

> Bundan sonra eski sürümlü 9800 serisi telefonlar bir sonraki yeniden
> başlatmada yükseltilir. Üretimde bunu **mesai dışı** yapın; yükseltme
> sırasında telefon çağrı yapamaz (aktif çağrı biter bitmez reset olur).

---

## Doğrulama (lab tamam)

- HQ-CIPC-1=11001, HQ-CIPC-2=11002 kayıtlı; aralarında çağrı çalışıyor.
- Üçüncü hat eklenebiliyor (düğme şablonu); iDivert softkey'i görünüyor.
- İki telefonun device pool'u BAT ile **BRANCH 1**.
- Telefon web arayüzü Device/Network bilgisini gösteriyor.
- Device Defaults'ta yeni telefon yükü var; Cisco TFTP yeniden başlatıldı.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| CIPC "Configuring IP"de takılı | Settings'te `* * #`, ya da CIPC'yi kapat-aç. |
| Telefon kayıt olmuyor, CTL failed | Yanlış/eski CTL; Görev 6 ile CTL'yi sil. |
| Şablon değişikliği görünmüyor | **Apply Config** + **Reset/Restart** yapıldı mı? |
| CIPC web arayüzü (6970) HTTPS'te açılmıyor | CIPC HTTPS desteklemez; `http://…:6970/…` kullan. |
| BAT işi çalışmıyor | **Cisco Bulk Provisioning Service** Activated mı? Job Scheduler'da **Activate Selected**. |
| Firmware yüklenince telefonlar eski sürümde | **Cisco TFTP** restart edilmeli; sonra telefon reset. |

## Eğitmen Notu

Bu lab provisioning'in bütün yollarını tek seferde gösterir: elle (tek
telefon), şablonla (düğme/softkey), toplu (BAT). Öğrencilere "aynı işi üç
yoldan yaptık" dedirtin. **Apply Config + Reset/Restart** refleksini
oturtun — en çok "neden değişmedi" sorusu buradan çıkar. CTL silmeyi
(`* * #` → Erase) kümeler arası taşımanın klasik çözümü olarak anlatın.
Firmware'de sıra önemli: FTP'ye koy → CUCM'e kur → **TFTP restart** →
telefon reset. BAT'ta önce **servisi aktive etmeyi** unutmayın; bu adım
atlanınca iş hiç çalışmaz.

---

### Sonraki

**Discovery 4** — guide'ın bir sonraki etkinliği. (Başlığı o lab'a
geçerken teyit edeceğim.)
