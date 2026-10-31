# Discovery 5 — Cisco Self-Provisioning ile IP Telefon Dağıtımı

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe; menü/alan
> adları, değerler ve kimlikler **birebir**. Süre: **30–45 dk**.

---

## Giriş

Telefonları tek tek elle sağlamak yerine **Self-Provisioning** ile kullanıcı,
basit bir **IVR** üzerinden kendi cihazını "sahiplenir". Bunu Autoregistration
+ evrensel şablonlarla birleştirip LDAP verisine bağlayacağız.

**Akış:** Telefon açılır → **Autoregistration** ile geçici bir numara (100X)
alır → kullanıcı **Self Prov** hızlı aramasıyla IVR'ı arar → kimliğini ve
auth kodunu girer → telefon, kullanıcının gerçek numarası/profiliyle yeniden
başlar.

**Zincir (önemli):** LDAP Sync → **Feature Group Template** → End User →
**User Profile** → (Universal Device Template + Universal Line Template +
"Allow End-User Provisioning"). Yani kullanıcıya hangi şablonun uygulanacağını
bu zincir belirler.

## Ortam

CUCM `https://10.1.5.5/ccmadmin` / `/ccmservice` (`admin`/`!234QWer`),
LDAP `10.1.5.100`, PC-1/PC-2 CIPC.

---

## Görev 1: Config Yükle

PC-1 → **Classfiles > Discoveries** → **lab5.py** → "successfully loaded".

## Görev 2: Self-Provisioning için Otomatik Kaydı Aç

Autoregistration self-provisioning'in **ön şartıdır**; autoreg olan telefonlara
bir **UDT** ve bir **ULT** atanır.

**System > Cisco Unified CM > Find > CM_CUCM-PUB**, **Auto-registration
Information**:
- **Universal Device Template:** `Auto-registration Template`
- **Universal Line Template:** `Sample Line Template with TAG usage examples`
- **Starting Directory Number:** `1000` · **Ending Directory Number:** `1009`
- **Auto-registration Disabled…:** **İŞARETSİZ** (yani autoreg açık) → **Save**

## Görev 3: Otomatik Kayıt için Evrensel Şablonlar

Varsayılan UDT'de hızlı arama düğmesi yoktur; IVR'ı kolay aramak için bir
**Speed Dial**'lı düğme şablonu yapacağız.

**2–7. Düğme şablonu:** **Device > Device Settings > Phone Button Template**
→ arama **begins with** `Universal` → **Find** → **Universal Device Template
Button Layout**'u **Copy** → ad `Universal Collab Template` → **Save** →
**button 2 = Speed Dial** → **Save**.

**8–9. UDT'ye ata:** **User Management > User/Phone Add > Universal Device
Template > Auto-registration Template** → **Phone Button Template:**
`Universal Collab Template` → **Save**.

## Görev 4: Self-Provisioning Öğelerini Kur ve İlişkilendir

Self-Provisioning IVR için: bir **CTI Route Point**, bir **dial-in numarası**,
bir **Application User** ve servis aktivasyonu gerekir.

**1–3. CTI Route Point:** **Device > CTI Route Point > Add New**:
- **Device Name:** `SelfProv` · **Description:** `SelfProv` · **Device Pool:** `Default`
- **Save** → **Line [1] – Add a new DN** → **Directory Number:** `11999`
  (IVR'ı aramak için çevrilecek numara) → **Save**.

**4. Application User:** **User Management > Application User > Add New**:
- **User ID:** `SelfProv` · **Password/Confirm:** `cisco`
- **Controlled Devices:** `SelfProv`
- **Groups:** `Standard CTI Enabled` → **Save**

**5–7. Servisleri aktive et:** **Cisco Unified Serviceability > Tools >
Service Activation** → Server **CUCM-PUB...—CUCM Voice/Video** → şunları aç:
`Cisco CallManager`, `Cisco CTIManager`, `Cisco Tftp`, **`Self Provisioning
IVR`**, `Cisco DirSync` → **Save**. Sonra CM Administration'a dön.

**8–9. Self-Provisioning yapılandır:** **User Management > Self-Provisioning**:
- **Authentication Mode:** `Require Authentication` → **Allow authentication
  for users (via Password/PIN) and Administrators (via Authentication Code)**
  → **Authentication Code:** `8888`
- **CTI Route Point:** `SelfProv` · **Application User:** `SelfProv`
- **Save** → açılan pencerede **Save and Restart Now** → "Self-Provisioning
  IVR service restarted successfully".

> CTI Route Point'in kaydolması **5 dk**'ya kadar sürebilir.

## Görev 5: Kullanıcı ve Telefon Şablonları

**User Profile** hangi UDT/ULT'nin kullanılacağını ve kullanıcının kendi
telefonunu sağlamaya izinli olup olmadığını tutar.

**3–4. User Profile:** **User Management > User Settings > User Profile** →
**Find** → **Standard (Factory Default) User Profile**'ı düzenle:
- **Name/Description:** `US Users User Profile`
- **UDT > Desk Phone / Mobile and Desktop / Remote Destination:** hepsi `US Users`
- **ULT:** `US Users`
- ☑ **Allow End User to Provision their own phones** (limit 10)
- ☑ **Make this the default User Profile** → **Save**

**5. Feature Group Template:** **User Management > User/Phone Add > Feature
Group Template > Add New**:
- **Name/Description:** `US Users FT`
- **User Profile:** `US Users User Profile` → **Save**

**6+. LDAP'a bağla:** LDAP Directory yapılandırmasına bu **Feature Group
Template**'i uygula ve **Perform Full Sync Now**. Böylece senkronize kullanıcılar
US Users User Profile'ı alır (ve kendi telefonlarını sağlayabilir).

## Görev 6: Self-Provisioning'i Doğrula

**1.** PC-1'de **Cisco IP Communicator**'ı başlat → ~1 dk'da **Autoregister**
olur, numara `100X` (ör. 1001) ve altında **Self Prov** hızlı araması görünür.

**2. Self-provision et:** CIPC'de **Self Prov** (2. düğme) → IVR'ı arar:
- **1**'e bas (yeni telefon sağla)
- **Self-provisioning ID: `11001`** + **#**
- Onay için **#**
- **Authentication Code: `8888`** + **#**
- CIPC yeni ayarlarla yeniden başlar → **John Doe / 4085551001**.

PC-2'de CIPC için tekrarla (ilgili kullanıcının ID'siyle).

> Reorder tonu alırsan önceki yapılandırmayı doğrula; doğruysa Self
> Provisioning IVR servisini yeniden başlatıp tekrar dene.

---

## Doğrulama (lab tamam)

- Autoreg açık; CIPC 100X ile otomatik kaydoluyor, Self Prov hızlı araması var.
- Self-Provisioning IVR servisi çalışıyor; CTI route point (11999) kayıtlı.
- IVR'dan self-prov ID + auth code ile telefon gerçek kullanıcıya bağlanıyor.
- LDAP kullanıcıları US Users User Profile'ı alıyor.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| CIPC autoreg olmuyor | CM'de autoreg kapalı ya da DN aralığı (1000-1009) dolu/çakışıyor. |
| Self Prov araması reorder tonu | Self Provisioning IVR servisi aktif/çalışıyor mu? CTI route point kaydoldu mu (5 dk)? |
| IVR auth kodunu reddediyor | Auth Code `8888` ve self-prov ID doğru mu? Kullanıcı provision izinli mi (User Profile)? |
| Kullanıcıya şablon uygulanmıyor | Feature Group Template → User Profile zinciri kurulu mu? LDAP sync yapıldı mı? |

## Eğitmen Notu

Bu lab'ın kalbi **ilişki zinciri**: FGT → User Profile → UDT/ULT +
"Allow End-User Provisioning". Tahtaya bu zinciri çizin; öğrenciler "telefon
hangi ayarı nereden alıyor" sorusunu burada çözer. İki numarayı karıştırmayın:
**11999** IVR'ı aramak için çevrilen numara (CTI route point), **self-prov ID**
ise kullanıcıyı tanıtan kimlik. Autoreg olmadan self-provisioning olmaz —
önce o açılmalı. Üretimde "No Authentication Required" modundan kaçının.

---

### Sonraki

**Discovery 6: Batch Provisioning ile IP Telefon Dağıtımı** — telefonları CSV
ile dışa aktarma, kayıtları düzenleme, BAT ile toplu ekleme.
