# Discovery 9 — Toll Fraud (Çağrı Dolandırıcılığı) Önleme

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe; menü/alan
> adları, numaralar, CSS ve servis parametresi adları **birebir**.
> Süre: **35–45 dk**.

---

## Giriş

**Toll fraud**, kurumsal trunk'ların yetkisiz kullanımıyla (çoğu zaman
uluslararası veya ücretli hatlara) kuruma **fatura** çıkartılmasıdır. Kötü
niyetli kişi; ele geçirilmiş uç nokta, **çağrı yönlendirme (forward)**,
**aktarma (transfer)** veya **konferans** üzerinden dışarıdan dışarıya çağrı
kurar — fatura kuruma gelir.

Bu lab'da CUCM'i dört yöntemle korursun: (1) hatta sınıf kısıtı, (2) call
forward kısıtı, (3) dış↔dış aktarma engeli, (4) içeride kimse kalmayınca
konferansı düşürme.

### Ortam
HQ-CUCM (San Jose), PC-1 (HQ-CIPC-1/11001), PC-2 (HQ-CIPC-2/11002).
**PC-3 = RTP** (ev PSTN telefonu, dolandırıcılık denemesi), **PC-4 = PSTN**
(ITSP), ikisi de TFTP `10.1.99.252`. CUCM `https://10.1.5.5/ccmadmin`
(`admin`/`!234QWer`). PSTN hızlı aramaları: **Emergency** `9.911`, **Local**
`9.4085550000`, **National** `9.18135550000`, **International**
`9.01161270100000`.

---

## Görev 1: Config Yükle ve Uç Noktaları Kaydet

PC-1 → **lab9.py** → "successfully loaded". PuTTY ile **her iki router**:

```
copy flash:/CLCOR-lab9.txt startup-config
```
reload et. CIPC'leri kaydet: CIPC-1 (PC-1), CIPC-2 (PC-2) → TFTP 10.1.5.5;
RTP (PC-3), PSTN (PC-4) → TFTP 10.1.99.252.

## Görev 2: Kısıtsız Baz Davranışı Doğrula

CIPC-2'den hızlı aramalarla PSTN'i test et: emergency, local, national,
**international** — hepsi çalışır (henüz kısıt yok). Bu baz çizgidir.

## Görev 3: Hatta Sınıf Kısıtı (CoS) Uygula

**Device > Phone > CIPC-2 > Line [1] (11002)** → **Directory Number
Settings > Calling Search Space:** `LINE_NATIONAL_CSS` → **Save** →
**Apply Config**.

**Test (CIPC-2):** emergency / local / national **çalışır**;
**international (9.01161270100000) ARTIK ÇALIŞMAZ**. Kullanıcı artık özel
(ve iş) uluslararası arama yapamaz.

## Görev 4: Call Forward için Sınıf Kısıtı

Kullanıcı, ofis telefonunu uluslararası bir numaraya **forward** edip, ev
telefonundan kurumun gateway'i üzerinden uluslararası arama yaptırabilir.

**Dolandırıcılığı göster:**
1. PC-2'de Self-Care (`https://10.1.5.5/ucmuser`, `jwhite`/`1234QWer`) →
   **Call Forwarding** → **Forward all calls to:** `901161270100000`.
2. PC-3'ten (RTP) `14085551002`'yi ara → çağrı CIPC-2'nin CFA'sı yüzünden
   **uluslararası hatta** yönlenir; RTP ↔ PSTN kurulur, **kurum öder**.
   (Ofis telefonu sadece "Forwarded to …" gösterir, aktif çağrı görünmez.)

**Engelle:** PC-3'te çağrıyı düşür. **CIPC-2 > Line [1] (11002)** →
**Call Forward and Call Pickup Settings** → **Forward All** için
**Calling Search Space:** `LINE_INTERNATIONAL_CSS` → **`LINE_NATIONAL_CSS`**
yap → **Save**. Artık RTP → CIPC-2 denemesi **başarısız**, çünkü CFA CSS'i
uluslararasına izin vermiyor.

> **Önemli:** Forward hedefi, **arayanın** değil **forward CSS'inin**
> ayrıcalığıyla değerlendirilir. Bu yüzden CFA için ayrı, kısıtlı bir CSS
> şart.

## Görev 5: Dış↔Dış Aktarmayı (Transfer) Engelle

Bir kullanıcı dışarıdan gelen çağrıyı bir dış uluslararası numaraya
**transfer** edip iki PSTN bacağını kurumun üzerinden bağlayabilir.

**Dolandırıcılığı göster:**
1. CIPC-1'den `919195550000` (RTP) ara, PC-3 yanıtla.
2. CIPC-1'de **Transfer** → `901161270100000` (International hızlı arama),
   PC-4 yanıtla → CIPC-1'de tekrar **Transfer**. CIPC-1'de artık **aktif
   çağrı yok** ama iki PSTN birbirine bağlı.
3. PSTN-GW'de (Router 2, PuTTY `10.1.99.252`) bacakları gör:

```
show call active voice compact
```
→ **4 call-leg**: her bağlantı için biri CUCM'e (VOIP) biri PSTN'e; her iki
VOIP bacağının peer adresi CIPC-1'in numarası.

**Engelle:** PC-3'te düşür. **System > Service Parameters** → Server
**CUCM-PUB.CLL-COLLAB.INTERNAL** → Service **Cisco CallManager (Active)** →
**Clusterwide Parameters (Feature - General)** → **Block OffNet to Offnet
Transfer = `True`** → **Save**. Artık dış→dış transfer reddedilir.

## Görev 6: İçeride Kimse Kalmayınca Ad Hoc Konferansı Düşür

Benzer dolandırıcılık konferansla da yapılır: kullanıcı iki dış tarafı
konferansa alıp **ayrılır**; konferans (dolayısıyla iki PSTN bağlantısı)
açık kalır.

**Dolandırıcılığı göster:**
1. CIPC-1'den `919195550000` (RTP) ara, yanıtla.
2. CIPC-1'de **Confrn** (gerekirse **more**) → `901161270100000`, yanıtla →
   **Confrn** → konferans kuruldu. **ConfList** ile üyeleri gör (sahibi `*`
   ile işaretli, 11001).
3. CIPC-1'de çağrıyı düşürüp **ayrıl** → iki PSTN tarafı konferansta kalır
   (PSTN-GW'de yine 4 call-leg).

**Engelle:** PC-3'te düşür. **System > Service Parameters** → **Cisco
CallManager** → **Clusterwide Parameters (Feature - Conference)** → **Drop Ad
Hoc Conference = `When No OnNet Parties Remain in the Conference`** → **Save**.
Tekrar denendiğinde, sahip ayrılınca (içeride kimse kalmayınca) **tüm çağrılar
düşer**.

---

## Doğrulama (lab tamam)

- CIPC-2 hattı ve CFA'sı uluslararası aramayı engelliyor.
- Block OffNet to OffNet Transfer = True → dış→dış transfer yok.
- Drop Ad Hoc Conference → içeride kimse kalmayınca konferans düşüyor.
- `show call active voice compact` dolandırıcı senaryolarda 4 bacak gösteriyor.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| Kısıtlı hat hâlâ uluslararası arıyor | Hat CSS'i (ve **CFA CSS**'i) kısıtlı mı? Apply Config yapıldı mı? |
| Forward ile fatura kaçağı | CFA için ayrı, kısıtlı CSS ata; arayanın CSS'i yeterli değil. |
| Transfer kaçağı sürüyor | **Block OffNet to OffNet Transfer = True** mu? |
| Konferans kaçağı sürüyor | **Drop Ad Hoc Conference** doğru değerde mi? |
| Hangi çağrı gerçek | `show call active voice compact` ile gateway'deki bacakları say. |

## Eğitmen Notu

Dört vektörü birlikte gösterin: hat CoS, **forward**, **transfer**,
**konferans**. Ortak mantık: "dış↔dış köprü kurulursa kurum öder." En çok
unutulan: forward ve transfer **arayanın** değil, **özelliğin kendi CSS'inin/
parametresinin** ayrıcalığıyla değerlendirilir. `show call active voice
compact` çıktısında **4 call-leg** görmek dolandırıcılığın parmak izidir —
öğrencilere bacak saymayı öğretin. Bu lab Modül 5'teki toll-fraud dört
katmanının CUCM tarafıdır (gateway/CUBE tarafı ayrı).

---

### Sonraki

**Discovery 10: Globalized Call Routing** — E.164 globalization, +E.164
pattern'ler, yerelleştirme (localization) ve çağrı yönlendirme normalizasyonu.
