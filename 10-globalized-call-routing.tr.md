# Discovery 10 — Globalized Call Routing (Küresel Çağrı Yönlendirme)

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe; desen,
> partition ve değerler **birebir**. Bu uzun ve kavram yoğun bir lab'dır
> (çok siteli). Süre: **70–90 dk**.

---

## Giriş — Temel Fikir

Çok siteli, merkezi çağrı işleme ortamında rakam düzenleme karmaşıklaşır.
Çözüm iki adımdır:

1. **Globalization (giriş/ingress):** Gelen **her** numarayı (çağrılan ve
   çağıran) tek bir evrensel biçime — **+E.164** — çevir. Böylece CUCM içinde
   herkes aynı dili konuşur.
2. **Localization (çıkış/egress):** Çıkışta, numarayı o **operatörün/ülkenin**
   beklediği yerel biçime geri çevir.

Arada tek bir **global route pattern** (`\+!`) tüm +E.164 çağrıları taşır.
Kullanıcıların alışkın olduğu yerel/ulusal/uluslararası çevirme biçimleri ise
**translation pattern**'lerle +E.164'e dönüştürülür.

### Çok önemli kural (bu lab'ın özü)
- **Transformation pattern'ler PRE-transformed (dönüştürülmemiş) numarayla
  eşleşir** — yani çağrı isteğinin **başındaki** hâliyle, route pattern/route
  list rakam düzenlemesinden **önce**.
- **Translation pattern** ise rakam düzenlemeyi uygulayıp **yeni bir arama
  isteği** üretir; sonuç, yeni isteğin pre-transformed numarasıdır.

## Ortam

Tek CUCM kümesi, iki site: **US San Jose (SJC)** ve **UK London (LON)**.
US-CIPC-1 (PC-1) `11001` / +1 408 5551001; UK-CIPC-1 (PC-3) `21001` /
+44 20 79460001. **PSTN-CIPC** (PC-4, 10.1.99.5) 8 hatlı ITSP simülatörü.

| Ülke/Şehir | PSTN (+E.164) | DN | CC | AC | Abone hane |
|---|---|---|---|---|---|
| US / SJC | +1 408 5551XXX | 11XXX | 1 | 408 | 7 |
| UK / LON | +44 20 7940XXX | 21XXX | 44 | 20 | 8 |
| US / TPA | +1 813 5550000 | — | 1 | 813 | 7 |
| UK / MAN | +44 161 4960000 | — | 44 | 161 | 7 |
| AU / SYD | +61 2 70100000 | — | 61 | 2 | 8 |

**Dial kuralları:** SJC → access 9, intl 011, national 1, local 10 hane,
emergency 991; LON → access 9, intl 00, national 0, local 8 hane,
emergency 999.

**Hazır partition'lar:** `PSTNInternational` (global route pattern `\+!`),
`USToE164`/`UKToE164` (globalization translation pattern'leri),
`SJCGWLocalizeCd`/`SJCGWLocalizeCn` (SJC egress called/calling transform),
`LONGWLocalizeCd`/`LONGWLocalizeCn` (LON egress), `USPhLocalize`/`UKPhLocalize`
(telefonda gösterim), `USEmergency`/`UKEmergency`.
Device pool: **SJC**, **LON**. Gateway: SJC **H.323** (10.1.99.5), LON **SIP**
(LON-GW).

---

## Görev 1: Config Yükle ve Uç Noktaları Kaydet

PC-1 → **lab10.py** → "successfully loaded". US-CIPC-1 (PC-1, TFTP 10.1.5.5),
UK-CIPC-1 (PC-3), PSTN-CIPC (PC-4, 10.1.99.5) kaydet.

## Görev 2: Mevcut Dial Plan Öğelerini İncele

**Call Routing > Class of Control > Partition** ve **Calling Search Space**,
**System > Device Pool** (SJC, LON), **Device > Gateway/Trunk**'ları gez.
Route group'lar: **SJC-H323_RG** (üye 10.1.99.5), **LON-SIP_RG** (üye LON-GW).
Route list → **Standard Local Route Group** ile device pool'a göre doğru
site gateway'i seçilir.

## Görev 3: Çıkışta Küresel Yönlendirme (Global Route Pattern)

**Call Routing > Route/Hunt > Route Pattern > Add New:**
- **Route Pattern:** `\+!` (herhangi bir +E.164 numara)
- **Partition:** `PSTNInternational`
- **Gateway/Route List:** site'in route list'i (Standard Local Route Group
  ile çıkış gateway'i device pool'dan seçilir) → **Save**.

> Tüm PSTN çıkışı tek desen üzerinden gider. "Hangi gateway" sorusunu
> **Standard Local Route Group** + device pool cevaplar; böylece ayrı ayrı
> site route pattern'leri gerekmez.

## Görev 4: LON Sitesine Gelen PSTN Çağrıları (ingress — called)

LON DID `+44 20 79460XXX`, iç DN `21XXX`. ITSP, LON gateway'e çağrılan
numarayı **10 haneyle** verir (SIP olduğu için TON yok).

**Sorun:** PC-4'ten **LON-local** hattıyla `79460001` ara → **başarısız**
(gelen numara iç DN ile eşleşmiyor).

**Çözüm — LON-GW trunk ingress:** **Device > Trunk > LON-GW** → **Incoming
Called Party Settings** → **Incoming Number:** **Prefix** `21`, **Strip
Digits** `7`. Böylece `2079460001` → (7 hane at) `001` → (21 ekle) `21001` =
iç DN. Tekrar ara → **çalışır**; çağıran numara +E.164 olarak görünür
(`442079460000`).

> **Calling** tarafı da ingress'te +E.164'e normalize edilir (Incoming Calling
> Party Settings / calling party transformation CSS = `UKToE164`).

## Görev 5: Çıkışta Yerelleştirme (egress localization — US örneği)

İç çağrılar +E.164'e dönüştürüldükten sonra, SJC gateway'inden PSTN'e çıkarken
operatörün beklediği yerel biçime indirilir.

**US yerel için translation pattern** (**Call Routing > Translation
Patterns > Add New**):
- **Translation Pattern:** `9.408XXXXXXX`
- **Partition:** `USToE164` · **Description:** `globalize US local calls`
- ☑ **Use Originator's Calling Search Space**
- ☑ **Urgent Priority** · ☑ **Do Not Wait For Interdigit Timeout On Subsequent Hops**
- **Called Party > Discard Digits:** `PreDot` · **Prefix Digits (Outgoing):** `+1`
- **Calling Party > Use Calling Party's External Phone Number Mask:** ☑

Sonuç: `9 408 5551234` → (9'u at) `4085551234` → (+1 ekle) `+14085551234`
(globalized). Ardından `\+!` route pattern'i PSTN'e taşır; egress'te
**called/calling party transformation** (`SJCGWLocalizeCd`/`Cn`) numarayı
yerel operatör biçimine indirir.

> Aynı mantık ulusal (`9.1…`), uluslararası (`9.011…`) ve LON tarafı
> (`9.0…`, `9.00…`, 8 hane) için ilgili `*ToE164` partition'larında tekrarlanır.

## Görev 6: Kullanıcı Çevirme Alışkanlıklarını Koru

Translation pattern'ler sayesinde kullanıcı **yerel alışkanlığıyla** çevirir
(SJC: `9`+10 hane / `91`+... / `9011`+...; LON: `9`+8 hane / `90`+... /
`900`+...) ve CUCM bunları +E.164'e çevirir. Böylece tek global yönlendirme,
çok sayıda yerel çevirme biçimini destekler.

## Görev 7: Acil Çağrı ve Doğru Caller ID

Acil numara (US `911`/`9911`, UK `999`/`9999`) **doğru yönlenir**, ama caller
ID yanlış gidebilir (iç DN gibi). Düzeltme: egress'te bir **Calling Party
Transformation Pattern**:
- **Pattern:** `1.1XXX` · **Partition:** `SJCGWLocalizeCn`
- **Description:** `set calling number at SJC egress if not translated`
- **Discard Digits:** `PreDot` · **Prefix Digits:** `408555`
- **Calling Party Number Type:** `Unknown` · **Numbering Plan:** `Unknown`

Sonuç: iç `11001` → egress'te çağıran numara `4085551001` olur. SJC
telefonundan `911` → PSTN hattı **SJC-emergency** (line 4), caller ID
**4085551001** (doğru).

> Transformation pattern **pre-transformed** `11001` ile eşleştiği için acil
> çağrıda bile çağıran numarayı düzeltir.

---

## Doğrulama (lab tamam)

- SJC ve LON'dan yerel/ulusal/uluslararası çağrılar doğru hatta düşüyor.
- Gelen ITSP çağrıları iç DN ile eşleşiyor (LON: prefix 21 / strip 7).
- Çağıran/çağrılan numaralar içeride +E.164, PSTN'de yerel biçimde.
- Acil çağrıda caller ID doğru (SJC: 408555XXX).

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| Gelen çağrı iç DN'e düşmüyor | Ingress Incoming Called Party prefix/strip yanlış. |
| Çağrı +E.164'e çevrilmiyor | Translation pattern partition'ı (ilgili `*ToE164`) telefonun CSS'inde mi? |
| PSTN "bu numarayı tanımıyorum" | Egress localization transform'u operatör biçimine indirmiyor. |
| Acil çağrıda caller ID yanlış | Calling Party Transformation Pattern (pre-transformed) eksik. |
| Yanlış gateway'den çıkıyor | Standard Local Route Group / device pool eşlemesi yanlış. |

## Eğitmen Notu

Tek cümle: **girişte +E.164'e çevir, çıkışta yerelleştir, arada tek `\+!`.**
En kritik kavram **pre-transformed eşleşme**: transformation pattern çağrının
başındaki numarayla eşleşir; translation pattern ise yeni bir arama üretir.
Bunu tahtada bir çağrının "numara yolculuğu" olarak çizin (telefon → giriş
normalize → `\+!` → çıkış localize → PSTN). Standard Local Route Group'un
neden çok siteli tasarımı sadeleştirdiğini vurgulayın (site başına ayrı route
pattern yerine tek desen + device pool). Bu lab Modül 3'teki rakam düzenleme
ve yol seçimi konusunun ileri, çok siteli hâlidir.

---

### Sonraki

**Discovery 11: On-Premises İşbirliği Ortamında API Kullanımı** — AXL SOAP
API, API etkinleştirme, sorgu/çağrı örnekleri ve otomasyon.
