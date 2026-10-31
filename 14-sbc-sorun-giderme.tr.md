# Discovery 14 — Session Border Controller (CUBE) Sorun Giderme

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe; IOS
> komutları, SIP alanları ve değerler **birebir**. Süre: **60–75 dk**.
> *(CSCGW'nin CUBE sorun giderme konusuyla doğrudan örtüşür.)*

---

## Giriş

CUBE arızaları genelde beş yerden çıkar: **SIP sinyali**, **DTMF relay**,
**voice translation**, **codec anlaşması** ve **dial-peer eşleşmesi**.
Bu lab'da `debug` ve `show` komutlarıyla SIP mesaj akışını okuyup her birini
teşhis edersin.

**Temel refleks:** Önce sinyal kuruldu mu (`debug ccsip messages`), sonra
hangi dial-peer/translation seçildi (`show`/`debug voice translation`), sonra
medya anlaştı mı (SDP, codec). Hata kodu hangi katmanı gösteriyor ona bak.

## Ortam (Topoloji)

HQ-UCM `10.1.5.5`, **ROUTER-1 = CUBE** `10.1.5.252`, ROUTER-2 (PSTN)
`10.1.99.252`, SITE1-UCM `10.1.15.5`. PC-1 (HQ 11001), PC-2 (müşteri 22001),
PC-3 (PSTN-CIPC). **Test çağrıları:** HQ→PSTN `91 813 555 0123`,
HQ→müşteri `22001`, müşteri→HQ `1 408 555 1001`, PSTN→HQ `408 555 1001`.

---

## Görev 1: Uç Noktaları Kaydet, Çağrıları Doğrula

PC-1 → **lab14.py** → "successfully loaded". Telefonları kaydet, yukarıdaki
test çağrılarını yap (yanıtlamana gerek yok, "Ring out" yeterli).

## Görev 2: SIP Mesaj Akışını Analiz Et (debug ccsip messages)

ROUTER-1'de:

```
debug ccsip messages
```
Bir çağrı yap, sonra `no debug all`. Çıktıda INVITE / 100 Trying / 200 OK /
ACK / BYE dizisini oku. Önemli alanlar:
- **Request-URI / To / From** → çağrının kime/kimden gittiği.
- **Via / Contact** → yol ve geri dönüş adresi.
- **SDP** (200 OK'te delayed offer'da) → `m=audio … RTP/AVP`, `a=rtpmap`,
  `telephone-event`.
- **Server:** başlığı (`Cisco-CUCM14.0`) → karşı tarafın kim olduğu.

## Görev 3: DTMF Relay Uyuşmazlığı

Tuşlar karşıya gitmiyorsa DTMF yöntemi uyuşmuyordur. SDP'de **NTE-RTP** olup
olmadığına bak: `a=rtpmap:101 telephone-event/8000` ve `m=audio … 101`.

**Teşhis:** `debug ccsip messages` ile bir çağrıda (ör. konferans `4085551009`)
200 OK / ACK SDP'sini incele; **telephone-event (101)** payload'ı var mı?
Çağrı sırasında CIPC'den birkaç tuş (`135`) girip gidip gitmediğini gör.

**Düzelt:** İki ucun DTMF yöntemini eşitle — giden dial-peer'da
`dtmf-relay rtp-nte` (gerekirse `sip-notify`/`digit-drop`), karşı taraf da
RFC2833 (rtp-nte) desteklemeli. Uyuşmazsa tonlar bozulur/gitmez.

## Görev 4: Voice Translation Rule / Profile

Numara karşı tarafa yanlış biçimde gidiyorsa translation kuralı hatalıdır.

```
debug voice translation
```
Müşteri sitesine (`22001`) bir çağrı yap. Çıktıda (RXRULE):
- **Çağıran** `11001` → **ruleset 200**, `sed_subst: pattern=11001
  matchPattern=^1 replacePattern=1408555` → sonuç `14085551001`.
- **Çağrılan** `22001` → **ruleset 201**, `matchPattern=^2
  replacePattern=1202555` → sonuç `12025552001`.
- `no rule for callback-number number` satırı normaldir (callback için kural yok).

**show ile doğrula:**
```
show call active voice compact
```
Aktif çağrıda `pid:200` / `pid:201` uygulanan translation profillerini gösterir.
Yanlış sonuç varsa ilgili `voice translation-rule`'un `rule` satırındaki
match/replace desenini düzelt.

## Görev 5: Codec Anlaşması

Ortak codec bulunamazsa çağrı kurulmaz.

**Teşhis:** `debug ccsip messages` → 200 OK SDP'sinde CUCM `m=audio … RTP/AVP
0 18`, `a=rtpmap:0 PCMU/8000`, `a=rtpmap:18 G729/8000` teklif ediyor; ama CUBE
bunları desteklemiyorsa CUBE bir **BYE** gönderir: **`Q.850;cause=65`**
(*bearer capability not implemented*) = ortak codec yok.

**Düzelt (CUBE codec listesini uyumla):**
```
config t
voice class codec 1
 no codec preference 1
 no codec preference 2
 codec preference 1 ilbc
end
```
(Örnekte liste iLBC'ye ayarlanıyor; gerçek çözümde iki tarafın **ortak** bir
codec'te buluşması gerekir — ya CUCM ilbc'yi destekler ya CUBE CUCM'in
tercih ettiğini.)

## Görev 6: İnaktif / Yanlış Yapılandırılmış Dial-Peer

```
show dial-peer voice summary
```
Giden/gelen peer'ların **durumunu** (up/down, admin), destination-pattern,
session target, codec ve bind'ini kontrol et. İnaktif ya da yanlış
`destination-pattern`/`session target` olan peer çağrıyı eşlemez. Düzelt,
sonra çağrıyı tekrarla.

## Görev 7: SIP Profili — Başlık Davranışı

ITSP belirli bir başlık biçimi istiyorsa (ör. Diversion, From, PAI) bir
**SIP profili** ile giden INVITE'taki başlığı yeniden yaz ve dial-peer'a
uygula (`voice class sip-profiles N` → `voice-class sip profiles N`).
`debug ccsip messages` ile başlığın düzeldiğini doğrula.

---

## Doğrulama (lab tamam)

- SIP akışı debug'da temiz (INVITE→200 OK→ACK); hata kodu yok.
- DTMF için NTE-RTP (telephone-event/8000) iki uçta anlaşılıyor; tuşlar gidiyor.
- Translation rule'lar doğru biçim üretiyor (pid:200/201).
- Ortak codec anlaşılıyor; BYE Q.850 cause=65 kalkıyor.
- Dial-peer'lar up ve doğru eşleşiyor.

## Sık Karşılaşılan Sorunlar

| Belirti | Teşhis komutu / çözüm |
|---|---|
| Tuşlar gitmiyor | `debug ccsip messages` SDP'de telephone-event var mı; DTMF yöntemini eşitle. |
| Numara yanlış biçimde gidiyor | `debug voice translation` (match/replace); kuralı düzelt. |
| BYE `Q.850;cause=65` | Ortak codec yok; `voice class codec` listesini uyumla. |
| Çağrı hiç eşleşmiyor | `show dial-peer voice summary`; destination-pattern/target/durum. |
| ITSP başlığı reddediyor | SIP profili ile başlığı yeniden yaz (`voice class sip-profiles`). |

## Eğitmen Notu

Sorun gidermeyi katmanlara bölün: **sinyal → eşleşme → medya**.
`debug ccsip messages` SIP'i, `debug voice translation` numara dönüşümünü,
`show call active voice compact` ve `show dial-peer voice summary` durumu verir.
Öğrencilere hata kodlarını okumayı öğretin: 403 (yetki/alan), 488 (codec),
BYE Q.850 cause=65 (ortak codec yok). NTE-RTP (RFC2833, telephone-event/8000)
DTMF'in SDP'deki imzasıdır. Bu lab Modül 5'in (CUBE) ve Modül 2'nin (DTMF,
codec, bind) uygulamalı sorun giderme hâlidir.

---

### Sonraki

**Discovery 15: Media Resources** — konferans köprüsü, transcoder, MTP, MOH
(music on hold) ve media resource group / list.
