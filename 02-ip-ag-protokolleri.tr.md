# Discovery 2 — Destekleyici IP Ağ Protokollerinin Yapılandırılması

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Fast Lane Lab Guide'ın yanında kullanılır. Adımlar ve "neden"ler
> Türkçe; menü/düğme adları, komutlar, dosya adları, IP'ler ve kimlikler
> ekranda göründüğü gibi **birebir** bırakıldı. Tahmini süre: **40–60 dk**.

---

## Giriş — Ne Yapıyoruz, Neden?

Bir işbirliği (collaboration) ortamı üç temel ağ servisine dayanır:
**DHCP** (otomatik IP), **DNS** (ad çözümleme) ve **NTP** (zaman eşitleme).
Bunlar doğru değilse IP telefonlar kayıt olamaz, isimler çözülmez, zaman
kayarsa sertifikalar geçersizleşir ve CUCM veritabanı replikasyonu bozulur.

Bu lab'da bir ağ yöneticisi gibi DHCP, DNS ve NTP'yi yapılandırıp,
doğrulayıp sorun giderirsin.

> **Önemli not (config yükleme):** Yükleme betiklerinin ve config adlarının
> numarası, o anki Discovery numarasıyla **örtüşmeyebilir**. Her zaman
> adımlarda yazan betiği çalıştır; numaraya güvenme.

## Ortam (Job Aid)

| Cihaz | IP | Kullanıcı | Parola |
|---|---|---|---|
| PC-1 | 10.1.5.200 | student | 1234QWer |
| PC-2 | 10.1.5.201 | student | 1234QWer |
| HQ-UCM (DHCP sunucu) | 10.1.5.5 | admin | !234QWer |
| WIN-SERVER01 (NTP/DNS/LDAP) | 10.1.5.100 | cll-collab\administrator | 1234QWer |
| HQ-Router (NTP) | 10.1.5.252 | admin | 1234QWer |
| PSTN-GW | 10.1.99.252 | admin | 1234QWer |

Telefonlar: **CIPC-1 = 11001** (PC-1), **CIPC-2 = 11002** (PC-2).
Alan adı: `cll-collab.internal`.

---

## Görev 1: Config Yükle ve Uç Noktaları Kaydet

**1.** PC-1 masaüstünde **Classfiles > Discoveries** → **lab2.py**'ye çift tıkla.
Açılan pencerede en alt satır **"Lab configuration successfully loaded"**
yazmalı. (Hata verirse eğitmene başvur.) Enter ile kapat.

**4.** PC-1'de **Cisco IP Communicator**'ı başlat ve ayarla:
- **Device Name:** `CIPC-1`
- **TFTP Servers:** `10.1.5.5`

> SCCP→SIP kontrol protokolü değişimi nedeniyle CIPC bir kez reset olur;
> açılan uyarıda **OK** de. Telefon **11001** (John) olarak HQ-UCM'e kaydolur.

---

## Görev 2: CUCM DHCP Servisini Yapılandır

Küçük ağlarda CUCM'in kendisi IP telefonlara **DHCP sunucusu** olabilir.

**1.** PC-1'de `https://10.1.5.5/ccmadmin` → `admin` / `!234QWer`.

**2.** **Device > Phone > Find:** CIPC-1 **registered**, IP `10.1.5.200`;
CIPC-2 **unregistered**, IP `10.1.5.201` (son başarılı kayıttan kalma).

**3–5. DHCP Sunucusu:** **System > DHCP > DHCP Server** → **Add New**:
- **Host Server:** `cucm-pub.cll-collab.internal`
- **Primary DNS IPv4:** `10.1.5.100`
- **Primary TFTP (Option 150):** `10.1.5.5` → **Save**

> **Option 150** telefona TFTP sunucusunun IP'sini verir. CIPC bu seçeneği
> farklı yorumlar; onda TFTP'yi ağ ayarından **elle** girmek gerekebilir.

**6–7. DHCP Alt Ağı:** **System > DHCP > DHCP Subnet** → **Add New**:
- **DHCP Server:** `cucm-pub.cll-collab.internal`
- **Subnet IPv4:** `10.1.5.0`
- **Primary Start:** `10.1.5.210`
- **Primary End:** `10.1.5.220`
- **Primary Router:** `10.1.5.250`
- **IPv4 Subnet Mask:** `255.255.255.0`
- **Primary DNS:** `10.1.5.100`
- **Primary TFTP (Option 150):** `10.1.5.5`
- Diğer alanlar varsayılan → **Save**

---

## Görev 3: PC-2'yi DHCP İstemcisi Yap ve Test Et

**1–2.** PC-2 masaüstünde **Network Connections** kısayolu. İki adaptör var:
- **lab** → statik `10.1.5.201` — **DEĞİŞTİRME**, dokunma.
- **dhcp** → bunun IP'yi DHCP ile aldığını doğrulayacağız.

**3.** PC-2'de **Command Prompt** → `ipconfig`. **dhcp** adaptörü
`10.1.5.210` almış olmalı (aralığın ilk boş adresi). *(DHCP sürecinin
tamamlanması birkaç dakika sürebilir.)*

**4.** **Wireshark**'ta DHCP el sıkışmasını izle — filtre: `bootp`.
Dört mesaj görürsün: **Discover → Offer → Request → ACK** (DORA).
PC-2, `10.1.5.210`'u alır.

**5–8.** Wireshark'ı durdur. Cisco IP Communicator'ı başlat → **11002**
olarak kaydolur. PC-1'de **Device > Phone**'da CIPC-2 artık **registered**.
11001'den 11002'ye çağrı denemesi çalışır.

> **Not (lab kısıtı):** `lab` adaptörü (10.1.5.201) sürekli aktif kalmak
> zorunda olduğundan IP gerçekte 10.1.5.210'a geçmeyebilir. Gerçek ortamda
> tek adaptör olur ve bu sorun yaşanmaz.

---

## Görev 4: İstemci DNS Ayarlarını Doğrula

Hatırlatma: **A kaydı** ad→IP, **PTR kaydı** IP→ad (ters DNS) çözümler.

**1.** PC-1'de Command Prompt → `nslookup`. Default Server
**win-server01.cll-collab.internal** (`10.1.5.100`) görünmeli.

**2.** `www.cisco.com` → çözülür (dış A kaydı çalışıyor, DNS sağlıklı).

**3.** `cucm-pub.cll-collab.internal` → **başarısız** ("Non-existent domain").
Yani CUCM için **A kaydı yok**.

**4.** `10.1.5.5` → **başarısız**. Yani **PTR kaydı da yok**.

> Sonraki görevde bu iki kaydı ekleyeceğiz. `nslookup`'tan çıkmak için
> `exit`.

---

## Görev 5: DNS Servisi — A ve PTR Kayıtları

**1.** PC-1'de **Remote Desktop Connection** → `10.1.5.100`,
`Administrator` / `1234QWer` (sertifika uyarısında **Yes**).

**2.** Sunucuda **DNS** (DNS Manager) aç → **Forward Lookup Zones >
cll-collab.internal**. Birçok kayıt var (ör. `pc-2` → 10.1.5.201) ama
**cucm-pub** yok.

**3–4.** `cll-collab.internal` zonuna sağ tık → **New Host (A or AAAA)…**:
- **Name:** `cucm-pub`
- **IP address:** `10.1.5.5`
- ☑ **Create associated pointer (PTR) record** (PTR'ı da otomatik üretir)
- **Add Host**

**5.** **Reverse Lookup Zones > 5.1.10.in-addr.arpa**'yı yenile; `10.1.5.5`
için **cucm-pub.cll-collab.internal** PTR kaydının oluştuğunu gör.

**6–7.** RDP'yi küçült, PC-1'de tekrar `nslookup`:
- `cucm-pub.cll-collab.internal` → artık `10.1.5.5` (A çalışıyor)
- `10.1.5.5` → artık `cucm-pub.cll-collab.internal` (PTR çalışıyor)

**8.** PC-2'de Chrome → `https://cucm-pub.cll-collab.internal` açılır.

> Çözüm hâlâ gelmiyorsa yönetici Command Prompt'ta `ipconfig /flushdns`
> ile DNS önbelleğini temizle.

---

## Görev 6: DNS Servisi — SRV Kayıtları

**SRV kaydı**, istemcinin bir servisi veren sunucuyu (host + port + öncelik
+ ağırlık) **otomatik** bulmasını sağlar; SIP ortamlarında yaygın.

**1–3.** DNS Manager'da `cll-collab.internal` → sağ tık **Other New
Records…** → **Service Location (SRV)** → **Create Record…**:
- **Service:** `_sip`
- **Protocol:** `_tcp`
- **Priority:** `0`
- **Weight:** `10`
- **Port:** `5060`
- **Host offering this service:** `cucm-pub.cll-collab.internal` → **OK**

**4.** İkinci SRV kaydı:
- **Service:** `_sip` · **Protocol:** `_tcp` · **Priority:** `0`
- **Weight:** `20`
- **Port:** `5060`
- **Host:** `cucm-sub1.cll-collab.internal` → **OK**

> **Ağırlık = yük dağılımı.** cucm-pub'a 10, cucm-sub1'e 20 vererek **1:2**
> oran kurarsın: pub'a giden her 1 INVITE'a karşı sub1'e 2 INVITE yönlenir.
> (cucm-sub1 yalnızca DNS'te var; kayıt mantığını göstermek için.)

**5.** `cll-collab.internal > _tcp` klasöründe iki `_sip` SRV kaydını gör.

**6–9.** PC-1'de Command Prompt → `nslookup`:

```
set typ=srv
_sip._tcp.cll-collab.internal
```

İki SRV kaydı dökülür: weight 10 → cucm-pub, weight 20 → cucm-sub1,
port 5060; ayrıca çözülen adresler (cucm-pub = 10.1.5.5).

---

## Görev 7: Cisco Yönlendiricide NTP

Zaman eşitlemesi kritiktir: saat kayarsa sertifika geçersizleşir,
replikasyon bozulur.

**1.** PC-1'de **PuTTY** → **Router 1** (`10.1.5.252`, saved session
**HQ-Router**) → **Load** → **Open**.

**2.** Giriş `admin` / `1234QWer`. Durumu gör:

```
show ntp status
```
→ `%NTP is not enabled.`

**3.** Yönlendiriciyi NTP **master** yap (stratum 15):

```
configure terminal
ntp master 15
```

**4.** Çık ve tekrar bak:

```
exit
show ntp status
```
→ `Clock is unsynchronized, stratum 15, reference is 127.127.1.1`
(kendi iç saatini kaynak alıyor; drift ölçülünce "synchronized" olur,
15 dk sürebilir — beklemeden devam edebilirsin).

**5.** Birliktelikleri gör:

```
show ntp associations
```
→ `127.127.1.1 .LOCL. st 14 … reach 377` (reach 377 = son 8 paketin
tamamı başarılı).

**6.** Dış zaman kaynağını ekle (lab'daki 10.1.5.100):

```
configure terminal
ntp server 10.1.5.100
```

**7–8.** `exit` → `show ntp associations` ve `show ntp status`:
→ saat `10.1.5.100` ile **synchronized**, **stratum 7** (kaynağa bir hop).
Daha düşük stratum'lu kaynak (10.1.5.100) referans seçilir.

**9–10.** PC-1'de Chrome → `https://10.1.5.5/cmplatform` → `admin` /
`!234QWer`. **Settings > NTP Servers**: mevcut NTP sunucusu `10.1.5.250`
**Accessible** görünür. (CUCM'in NTP sunucusu kurulumda eklenmiştir.)

---

## Doğrulama (lab tamam)

- PC-2 **dhcp** adaptörü DHCP ile `10.1.5.210` aldı; Wireshark'ta DORA görüldü.
- `nslookup`: `cucm-pub.cll-collab.internal` ↔ `10.1.5.5` (A + PTR çalışıyor).
- `set typ=srv` ile iki `_sip._tcp` SRV kaydı (weight 10/20) dönüyor.
- Yönlendirici `show ntp status` → synchronized; CUCM NTP sunucusu Accessible.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| PC-2 DHCP ile IP almıyor | DHCP Subnet aralığı/maske/sunucu yanlış; `lab` adaptörü aktifse (lab kısıtı) IP geçmeyebilir. |
| `cucm-pub` çözülmüyor | A kaydı yok ya da önbellek bayat; Görev 5'i yap, `ipconfig /flushdns`. |
| Ters arama (IP→ad) başarısız | PTR kaydı yok; New Host'ta "Create associated PTR" işaretli miydi? |
| SRV sorgusu boş dönüyor | `set typ=srv` yazıp sonra `_sip._tcp.cll-collab.internal` sorgula; kayıtlar `_tcp` klasöründe mi? |
| `show ntp status` hep unsynchronized | Drift ölçülüyor (15 dk'ya kadar); dış kaynak (`ntp server`) erişilebilir mi? |
| CIPC TFTP'yi almıyor | Option 150 CIPC'de elle girilmeli (ağ ayarları → TFTP 10.1.5.5). |

## Eğitmen Notu

Bu lab'ın bütünü "altyapı sağlam değilse collab çalışmaz" mesajıdır.
Üç servisi birbirine bağlayın: DHCP telefona IP+TFTP verir, DNS isimleri
çözer (SRV ile sunucuyu bulur), NTP zamanı tutar (sertifika/replikasyon).
DORA'yı Wireshark'ta canlı gösterin — öğrenciler dört mesajı ilk kez
görünce kavram oturur. SRV ağırlığındaki 1:2 yük dağılımını tahtada
anlatın. NTP'de `127.127.1.1` iç saat referansının ve stratum
mantığının (düşük stratum kazanır) altını çizin. Lab adaptörü kısıtını
baştan söyleyin ki "IP neden geçmedi" tartışması çıkmasın.

---

### Sonraki

**Discovery 3: Elle IP Telefon Yönetimi** — IP Communicator'ı elle ekleme,
telefon düğme/softkey şablonları, BAT ile toplu işlem, telefon GUI'si.
