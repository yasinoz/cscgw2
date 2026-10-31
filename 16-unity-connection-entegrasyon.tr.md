# Discovery 16 — Cisco Unity Connection'ı CUCM ile Entegre Et

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe; menü/alan
> adları ve değerler **birebir**. Süre: **30–45 dk**.

---

## Giriş

Sesli mesaj (voicemail), **Cisco Unity Connection (CUC)** ile CUCM arasında
doğru kurulmuş bir **SIP entegrasyonuna** dayanır. Messages düğmesi, çağrı
yönlendirmeyle voicemail'e düşme, **MWI** (mesaj bekliyor ışığı) ve karşılama
anonsları bu entegrasyon doğruysa çalışır.

Bu lab'da **iki tarafı** da yapılandırırsın:
- **CUCM tarafı:** SIP trunk güvenlik profili + SIP profili, Unity'ye SIP
  trunk, route group/list/pattern, voicemail pilot/profile.
- **Unity tarafı:** Phone System, Port Group ve Portlar.

## Ortam

| Öğe | Değer |
|---|---|
| CUCM | `https://10.1.5.5/ccmadmin` (`admin`/`!234QWer`) |
| Unity Connection | `10.1.5.8` — yönetim `https://10.1.5.8/cuadmin` |
| Voicemail pilot numarası | `1999` |

---

## Görev 1: SIP Trunk Security Profile (CUCM)

**System > Security > SIP Trunk Security Profile > Add New:**
- **Name:** `Unity_CXN_Security_Profile`
- **Description:** `SIP Trunk Security Profile for Unity Connection`
- **Incoming Transport Type:** `TCP+UDP` · **Outgoing:** `TCP` · **Incoming Port:** `5060`
- ☑ **Accept out-of-dialog refer** ☑ **Accept unsolicited notification**
  ☑ **Accept replaces header**
- Diğerleri varsayılan → **Save**

> Bu üç kutu (out-of-dialog refer / unsolicited notification / replaces header)
> voicemail'in transfer, MWI ve çağrı kontrol davranışları için gereklidir.

## Görev 2: SIP Profile (CUCM)

Standart SIP profilinden PING özellikli bir profil kullan (ör. **Standard
SIP Profile+PING**) — trunk'ın canlılığını OPTIONS/PING ile izler.

## Görev 3: Unity'ye SIP Trunk (CUCM)

**Device > Trunk > Add New** → **Trunk Type:** `SIP Trunk`,
**Device Protocol:** `SIP`, **Trunk Service Type:** `None(Default)` → Next:
- **Device Name:** `Unity_Trk` · **Description:** `Trunk to Unity Connection`
- **Device Pool:** `Default`
- **Inbound Calls:** ☑ **Redirecting Diversion Header Delivery – Inbound**
- **Outbound Calls:** ☑ **Redirecting Diversion Header Delivery – Outbound**
- **Destination Address:** `10.1.5.8` · **Destination Port:** `5060`
- **SIP Trunk Security Profile:** `Unity_CXN_Security_Profile`
- **SIP Profile:** `Standard SIP Profile+PING` → **Save** → **Reset**

> **Redirecting Diversion Header** şart: voicemail'in "çağrı KİME yönlendirildi"
> bilgisini alıp **doğru kullanıcının kutusunu** ve karşılamasını seçmesi
> buna dayanır. Eksikse herkes genel karşılamaya düşer.

## Görev 4: Voicemail Yönlendirmesi (Route Group/List/Pattern)

**Route Group:** **Call Routing > Route/Hunt > Route Group > Add New** →
**Name:** `Unity_rg`, üye **Unity_Trk** → **Save**.

**Route List:** **Route List > Add New** → **Name:** `Unity_rl`,
CUCM group ata, **Add Route Group** = `Unity_rg` → **Save**.

**Route Pattern:** **Route Pattern > Add New** → **Route Pattern:** `1999`,
**Gateway/Route List:** `Unity_rl` → **Save** (auth code OK, route list reset OK).

> Route pattern'i doğrudan trunk'a da gösterebilirsin; ama route list'e
> göstermek, desen değişince **aktif çağrıların düşmesini** önler (route list
> bir yazılım katmanıdır).

## Görev 5: Voicemail Pilot ve Profile (CUCM)

- **Advanced Features > Voice Mail > Voice Mail Pilot:** pilot numarası `1999`
  (Unity'ye route pattern ile ulaşan numara).
- **Voice Mail Profile:** bu pilotu içeren profil; kullanıcı/hat **Messages**
  düğmesi bu profili kullanır.
- Hat/DN'lerde **Voice Mail Profile** ve **Call Forward > Voicemail** ayarları
  yapılır (meşgul/yanıtsız → voicemail).

## Görev 6: Unity Connection Tarafı — Phone System, Port Group, Ports

Unity yönetimi: `https://10.1.5.8/cuadmin`.

**Phone System:** **Telephony Integrations > Phone System** → `CUCM SIP`
(Default TRAP Phone System işaretli).

**Port Group:** Phone System içinde **Related Links > Add Port Group** →
- **Phone System:** `CUCM SIP` · **Port Group Type:** `SIP`
- **Display Name:** `CUCM SIP-1`
- **SIP Security Profile:** `5060` · **SIP Transport Protocol:** `TCP`
- **Primary Server > IPv4 Address or Host Name:** `10.1.5.5` · **Port:** `5060`
- **Save**

**Ports:** Uyarı "ports need to be added" → **Related Links > Add Ports** →
port sayısını (ör. istenen eşzamanlı voicemail oturumu kadar) ekle → **Save**.

> Unity tarafı CUCM'e (10.1.5.5) bakar; CUCM tarafı Unity'ye (10.1.5.8).
> İki uç birbirini işaret etmezse portlar kaydolmaz.

## Görev 7: Doğrulama ve RTMT ile İzleme

**Test:** CIPC'de **Messages** düğmesine bas (ya da `1999` çevir) → Unity
karşılaması gelmeli. Bir telefonu meşgul/yanıtsız bırakıp voicemail'e düşmeyi
ve **MWI**'yi kontrol et.

**RTMT (Unity portlarını izle):** PC-1'de **Cisco Unified Real-Time Monitoring
Tool** → Host **`10.1.5.8`** (Unity) → `Accept` (self-signed sertifika) →
**Port Monitor** ile canlı voicemail port etkinliğini izle: calling/called/
**redirect** numaraları ve çağrının hangi karşılama/handler'a düştüğünün
sebebi.

---

## Doğrulama (lab tamam)

- Unity'ye SIP trunk (Unity_Trk → 10.1.5.8) kurulu; route pattern 1999 ona yönlendiriyor.
- Voicemail pilot/profile atanmış; Messages düğmesi Unity'yi açıyor.
- Unity'de Phone System (CUCM SIP) + Port Group (10.1.5.5) + portlar kayıtlı.
- RTMT Port Monitor'da gelen voicemail çağrıları görünüyor.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| Messages'a basınca reorder | Route pattern 1999 / route list / trunk zinciri eksik; voicemail profile atanmamış. |
| Herkes genel karşılamaya düşüyor | **Redirecting Diversion Header** (in/out) işaretli değil. |
| Unity port kaydolmuyor | Port Group primary server (10.1.5.5) / port sayısı / transport yanlış. |
| MWI yanmıyor | Unsolicited notification kapalı ya da MWI on/off numaraları yanlış. |
| Trunk down | SIP Profile PING/OPTIONS ile canlılık; hedef 10.1.5.8:5060 erişilebilir mi? |

## Eğitmen Notu

Entegrasyonu "iki yön" olarak çizin: CUCM → (SIP trunk) Unity (10.1.5.8),
Unity → (Port Group) CUCM (10.1.5.5). Voicemail'e ulaşma zinciri:
**Messages/forward → voicemail profile → pilot 1999 → route pattern → route
list → Unity_Trk.** **Redirecting Diversion Header**'ın neden kritik olduğunu
vurgulayın: doğru kutu/karşılama bu bilgiyle seçilir. RTMT **Port Monitor**
voicemail sorun gidermenin ana aracıdır (yanlış karşılama, başarısız transfer,
beklenmedik yönlendirme). Bu lab Modül 1'deki UC mimarisinin sesli mesaj
ayağını tamamlar.

---

### Sonraki

**Discovery 17: Unity Connection Kullanıcılarını Yönet** — kullanıcı şablonları,
toplu kullanıcı içe aktarma (LDAP/CSV), sesli mesaj kutusu ayarları ve sınıflar.
