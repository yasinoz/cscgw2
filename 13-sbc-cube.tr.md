# Discovery 13 — Session Border Controller (CUBE) Uygulama

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe; IOS
> komutları, menü/alan adları ve değerler **birebir**. Süre: **60–75 dk**.
> *(Bu lab, CSCGW kursunun CUBE konusuyla doğrudan örtüşür.)*

---

## Giriş

**Cisco Unified Border Element (CUBE)**, kurumun SIP sınırında duran **Session
Border Controller (SBC)**'dir: iç UC ile dış operatör (ITSP) / başka siteler
arasında SIP sinyalini **normalleştirir**, adresi gizler, codec/DTMF uyumunu
yönetir ve çağrıyı **B2BUA** olarak iki ayrı bacağa böler.

Bu lab'da HQ CUCM'i hem bir **ITSP** (PSTN) hem bir **müşteri sitesi** ile
CUBE üzerinden birleştirirsin.

## Ortam (Topoloji)

| Cihaz | IP | Rol |
|---|---|---|
| HQ-UCM | 10.1.5.5 | Merkez CUCM |
| ROUTER-1 | 10.1.5.252 | **CUBE** (SBC) |
| ROUTER-2 | 10.1.99.252 | PSTN gateway (ITSP ucu) |
| SITE1-UCM | 10.1.15.5 | Müşteri sitesi CUCM (VPN) |
| PC-1 | 11001 | HQ telefonu |
| PC-3 | PSTN-CIPC | PSTN telefonu |
| PC-2 | 22001 | Müşteri sitesi telefonu |

**ITSP SIP trunk gereksinimleri:** sunucu `10.1.99.252`, port `5060`,
transport **UDP**, codec **G711**, faks **T.38**, DTMF **RFC2833 + SIP
NOTIFY**, early offer: hayır (ama önerilir), CUBE'un ITSP'ye dönük IP'si
`10.1.5.252`.
**Müşteri SIP trunk:** sunucu `10.1.15.5`, port `5060`, UDP/TCP, codec
G711 u/a-law, G729, iLBC; DTMF RFC2833 + SIP NOTIFY.

---

## Görev 1: CUCM Tarafı — SIP Trunk'ı CUBE'a Kur

**1. SIP Trunk Security Profile:** **System > Security > SIP Trunk Security
Profile** → "Non Secure SIP Trunk Profile" kopyala →
- **Name:** `Non Secure SIP Trunk Profile for Cisco UBE`
- **Incoming Transport Type:** `TCP+UDP` · **Outgoing:** `TCP`
- **Incoming Port:** `5060` → **Save**

> Her SIP trunk'a ayrı profil iyi pratiktir; ileride bir trunk'ta değişiklik
> diğerlerini etkilemez.

**2. SIP Trunk:** **Device > Trunk > Add New** → **Trunk Type:** `SIP Trunk`,
**Device Protocol:** `SIP`, **Trunk Service Type:** `None(Default)` → Next:
- **Device Name:** `toCUBE` · **Device Pool:** HQ
- **Destination Address:** `10.1.5.252` (CUBE), **Port:** 5060
- **SIP Trunk Security Profile:** `Non Secure SIP Trunk Profile for Cisco UBE`
- **SIP Profile:** Standard → **Save** → **Reset**.

## Görev 2: CUCM Tarafı — Route Group / Route List / Route Pattern

**Route Group:** **Call Routing > Route/Hunt > Route Group > Add New** →
**Name:** `US-SIP-RG`, üye **toCUBE (10.1.5.252)** (All Ports) → **Save**.

**Route List:** **Route List > Add New** → **Name:** `US-ITSP-RL`,
**CUCM Group:** `HQ-CUCM-GROUP-1`, ☑ Device is trusted → **Save** →
**Add Route Group** = `US-SIP-RG` → **Save**.

**Route Pattern'ler:**
- PSTN desenleri (408…, national, intl, 911) → `US-ITSP-RL` (CUBE üzerinden PSTN)
- Müşteri sitesi: **Route Pattern** `2200X`, **Partition** `DEVICE_US_PT`,
  **Description** `Calls to Customer Site 1`, **Gateway/Route List** `US-ITSP-RL`,
  **Provide Outside Dial Tone** işaretsiz → **Save**. (22000–22009 çağrıları
  CUBE'a gider.)

## Görev 3: CUBE Tarafı — IOS Yapılandırması (ROUTER-1)

PuTTY ile ROUTER-1 (`10.1.5.252`). CUBE, CUCM'den gelen SIP'i ITSP'ye SIP
olarak taşır (SIP-to-SIP, B2BUA).

**Gelen (CUCM'den) dial-peer** (örnek 101): `session protocol sipv2`,
`voice-class codec` listesi, `voice-class sip bind all source-interface …`,
`incoming called-number …` / uygun eşleşme.

**Giden (ITSP'ye) dial-peer:**

```
config t
dial-peer voice 7 voip
 description *Outgoing DP for Subscriber Calls to ITSP*
 session protocol sipv2
 session target ipv4:10.1.99.252
 session transport udp
 voice-class sip bind all source-interface GigabitEthernet 1
 destination-pattern 408.......$
 codec g711ulaw
 dtmf-relay rtp-nte digit-drop sip-notify
 ip qos dscp cs3 signaling
 no vad
end
```

Satır satır:
- `session target ipv4:10.1.99.252` → ITSP'nin (ROUTER-2) adresi.
- `session transport udp` → global TCP'yi bu peer için **UDP**'ye ezer (ITSP UDP ister).
- `voice-class sip bind all source-interface Gi1` → ITSP'ye dönük arayüzle konuş.
- `destination-pattern 408.......$` → `408` + 7 hane; `$` ile 10 haneyi aşma.
- `codec g711ulaw` → ITSP tek codec; liste yerine tek codec (varsayılan G.729r8'i ezer).
- `dtmf-relay rtp-nte digit-drop sip-notify` → ITSP'nin istediği DTMF yöntemleri.

> İyi pratik: tek codec bile olsa `voice class codec` listesi kullanmak;
> sonra codec eklerken tüm dial-peer'ları değil tek listeyi güncellersin.

## Görev 4: Doğrulama — SIP Debug ve Çağrı

**1.** ROUTER-1'de: `debug ccsip messages` (CUCM↔CUBE↔ITSP SIP izini gör).
**2.** PC-1'den (CIPC) PSTN'i ara: `9 408 555 0123` → telefon "Ring out",
PC-3 (PSTN) çalar. Ayrıca `911`/`9911`, `918135550123`, `901161270100123#`
test et (bu lab'da PSTN yalnızca listelenen numaraları çalar).

## Görev 5: SIP Profili — Diversion Header Düzeltmesi

Çağrı yönlendirme (forward) senaryosunda ITSP, CUBE'un gönderdiği INVITE'ı
**403 Forbidden** ile reddedebilir.

**Teşhis (SIP merdiveni / debug):** ITSP `100 Trying` sonra **403 Forbidden**
döner. Giden INVITE'ta:
```
INVITE sip:18135550123@10.1.99.252:5060 SIP/2.0
```
çağıran numara `12025552001`, **Diversion** numarası `11001` — ikisi de ITSP
tarafından **tanınmıyor**. En az biri düzeltilmeli.

**Çözüm (önerilen — Diversion'ı düzelt):** Giden dial-peer'a bir **SIP profili**
uygula ve **Diversion** başlığını ITSP'nin kabul ettiği geçerli bir PSTN/+E.164
numaraya yeniden yaz. Böylece tanınmayan çağıran numara bile yönlendirilen
numaraya ITSP tarafından geçirilir ve uzak tarafta asıl arayan görünür
(kullanıcı beklentisi budur).

```
voice class sip-profiles 100
 request INVITE sip-header Diversion modify "<tanınmayan 11001>" "<geçerli PSTN numarası>"
!
dial-peer voice 7 voip
 voice-class sip profiles 100
```

> Çağıran numarayı PSTN numaranla değiştirirsen yönlendirmeden sonra "kim
> aradı" bilgisi kaybolur; bu yüzden **Diversion**'ı düzeltmek tercih edilir.

---

## Doğrulama (lab tamam)

- CUCM'den CUBE'a SIP trunk kurulu; route list/pattern CUBE'a yönlendiriyor.
- CUBE giden dial-peer ITSP'ye UDP/G711/RFC2833 ile çağrı kuruyor.
- `debug ccsip messages` çağrı akışını gösteriyor; PSTN çalıyor.
- Diversion header düzeltmesiyle yönlendirilen çağrı 403 almıyor.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| ITSP 403 Forbidden | Calling/Diversion numarası tanınmıyor; SIP profili ile düzelt. |
| 488 Not Acceptable | Codec uyuşmazlığı; ITSP'nin istediği codec'i (G711) dial-peer'a koy. |
| Tek yönlü / sessiz ses | `bind` yanlış arayüz; SDP'de ulaşılamaz adres. |
| Çağrı CUBE'a gitmiyor | CUCM route list/pattern `US-ITSP-RL`/`toCUBE`'a işaret etmiyor. |
| Tuşlar gitmiyor | DTMF yöntemi uyuşmuyor (`rtp-nte` / `sip-notify`). |

## Eğitmen Notu

CUBE'u tek cümleyle: **iki SIP ağının birleştiği yerde duran, her iki tarafa
ayrı uç nokta gibi görünen B2BUA.** Akışı tahtaya çizin: CUCM → (SIP) CUBE →
(SIP) ITSP; CUBE iki ayrı bacak kurar, adres gizler, codec/DTMF/transport
uyumunu sağlar. `debug ccsip messages` + SIP merdiveni okuma refleksini verin:
403/488/500 kodları ve hangi alan (Diversion, From, SDP) sorunlu. Bu lab
Modül 5'in (CUBE) ve Modül 4'ün (SIP) uygulamalı karşılığıdır.

---

### Sonraki

**Discovery 14: Session Border Controller Sorun Giderme** — CUBE SIP trunk ve
çağrı akışı arızalarını debug ve show komutlarıyla teşhis.
