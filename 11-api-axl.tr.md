# Discovery 11 — On-Premises İşbirliği Ortamında API Kullanımı (AXL)

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe; URL,
> SOAP işlemleri ve değerler **birebir**. Süre: **50–60 dk**.

---

## Giriş

Grafik arayüzün ötesinde CUCM, **AXL (Administrative XML)** arayüzüyle
**programatik** yönetim sunar: yapılandırma verisini **oku / değiştir / sil**.
Büyük ölçekli provisioning, veritabanı tutarlılığı ve otomasyon bununla yapılır.
AXL, **SOAP/XML** tabanlı bir API'dir.

Bu lab'da **Insomnia** (API istemcisi) ile AXL isteklerini göndereceksin:
telefon verisini **getPhone** ile oku, hat etiketini **updatePhone** ile
değiştir, bir cihazı **removePhone** ile sil.

## Ortam

HQ-UCM `10.1.5.5`, PC-1 `10.1.5.200`. AXL uç noktası:
**`https://10.1.5.5:8443/axl/`** (sondaki `/` şart). Kimlik: `admin` /
`!234QWer`. XML dosyaları PC-1 masaüstünde **API XML Files** klasöründe
(`get_phone.xml`, `update_phone_label.xml`, `delete_phone.xml`).

---

## Görev 1: Config Yükle ve Uç Noktayı Kaydet

PC-1 → **Classfiles > Discoveries** → **lab11.py** → "successfully loaded".
CIPC-1'i kaydet (TFTP 10.1.5.5).

## Görev 2: AXL Web Service'i Doğrula

**Cisco Unified Serviceability > Tools > Control Center – Feature Services**
→ Server **cucm-pub.cll-collab.internal** → **Go** → listede **Cisco AXL Web
Service**'in **Started** ve **Activated** olduğunu doğrula. (Değilse
**Service Activation**'dan aç.)

> AXL servisi kapalıysa hiçbir API isteği çalışmaz — ilk bakılacak yer.

## Görev 3: Insomnia Ortamı (Environment) ve Bağlantı Testi

API istemcisinde **environment**, siteye özgü değişkenleri (IP, port,
kullanıcı, parola) tutar; aynı istek farklı CUCM'lere değişkenlerle gönderilir.

**1–4. Private Environment oluştur:** Insomnia'da **Base Environment** →
**Manage Environments** → Collection Environments yanındaki **+** →
**Private Environment**. İçine değişkenleri gir:

```
{
  "ucm_host": "10.1.5.5",
  "ucm_port": "8443",
  "ucm_username": "admin",
  "ucm_password": "!234QWer"
}
```

**5. Bağlantı testi (GET):** Yeni istek → yöntem **GET**, URL
**`https://10.1.5.5:8443/axl/`** → **Auth: Basic** (`admin`/`!234QWer`) →
**Send**. Beklenen: **200 OK** ve Preview'da *"Cisco CallManager: AXL Web
Service is working. Use HTTP POST to send a request."*

> GET yalnızca bağlantıyı doğrular; gerçek AXL işlemleri **POST** ile yapılır.

## Görev 4: AXL Sorgusu — getPhone (oku)

**1.** İsteği kopyala/yeni POST isteği oluştur; yöntem **POST**, URL değişkenli:
**`https://{{ _.ucm_host }}:{{ _.ucm_port }}/axl/`**, **Auth: Basic** ile
`{{ _.ucm_username }}` / `{{ _.ucm_password }}`.

**2.** **Body > XML** seç; `get_phone.xml` içeriğini yapıştır:

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                  xmlns:ns="http://www.cisco.com/AXL/API/14.0">
  <soapenv:Header/>
  <soapenv:Body>
    <ns:getPhone>
      <name>CIPC-1</name>
    </ns:getPhone>
  </soapenv:Body>
</soapenv:Envelope>
```

> `ns` namespace'indeki sürüm (`14.0`/`15.0`) **CUCM sürümünle** eşleşmeli.

**3. Send** → **200 OK**. Preview'da `getPhoneResponse` içinde CIPC-1'in tüm
yapılandırması XML olarak döner (name, description, product, model, protocol,
devicePoolName=HQ, vb.). 4xx/5xx alırsan namespace sürümü ve XML biçimini
kontrol et.

## Görev 5: AXL Güncelleme — updatePhone (hat etiketini değiştir)

`update_phone_label.xml` gövdesi hattın **label** (hat metni) değerini değiştirir:

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                  xmlns:ns="http://www.cisco.com/AXL/API/14.0">
  <soapenv:Header/>
  <soapenv:Body>
    <ns:updatePhone>
      <name>CIPC-1</name>
      <lines>
        <line>
          <index>1</index>
          <label>Anita - 11001</label>
          <dirn>
            <pattern>11001</pattern>
            <routePartitionName>DN</routePartitionName>
          </dirn>
        </line>
      </lines>
    </ns:updatePhone>
  </soapenv:Body>
</soapenv:Envelope>
```

**POST** → **200 OK**. CUCM'de **Device > Phone > CIPC-1 > Line [1]** aç:
**Line Text Label** artık `Anita - 11001`. (GUI'de dakikalarca sürecek bir
değişiklik API ile saniyeler içinde — ölçeklenince gücü buradan gelir.)

## Görev 6: AXL Silme — removePhone (cihazı sil)

`delete_phone.xml` gövdesi bir cihazı veritabanından siler:

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                  xmlns:ns="http://www.cisco.com/AXL/API/14.0">
  <soapenv:Header/>
  <soapenv:Body>
    <ns:removePhone>
      <name>CIPT-2</name>
    </ns:removePhone>
  </soapenv:Body>
</soapenv:Envelope>
```

**POST** → **200 OK**. CUCM'de **Device > Phone > Find**: `CIPT-2` artık yok.

> `removePhone` **geri alınamaz** — üretimde cihaz adını iki kez kontrol et.

---

## Doğrulama (lab tamam)

- AXL Web Service Started/Activated; GET testi 200 OK.
- getPhone CIPC-1 verisini XML döndürüyor.
- updatePhone hat etiketini değiştiriyor (GUI'de görünüyor).
- removePhone cihazı siliyor.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| 401 Unauthorized | Auth Basic kullanıcı/parola yanlış; AXL yetkili rol gerek. |
| 403 / "service not activated" | Cisco AXL Web Service kapalı; Serviceability'den aç. |
| 500 / SOAP Fault | XML biçimi bozuk ya da namespace sürümü CUCM'le uyuşmuyor. |
| Bağlantı yok / sertifika | URL `https://…:8443/axl/` (sondaki `/`), sertifikaya güven/izin. |
| 404 | URL yanlış; `/axl/` yolu ve port 8443 doğru mu? |

## Eğitmen Notu

AXL'in üç fiilini vurgulayın: **get / update / remove** (ve add). SOAP
zarfının üç sabitini gösterin: `soapenv:Envelope`, namespace **sürümü**
(CUCM ile eşleşmeli), işlem etiketi (`ns:getPhone` vb.). Environment
değişkenlerinin neden önemli olduğunu anlatın: aynı istek, farklı CUCM'lere
değişkenlerle gider (çok siteli otomasyon). HTTP durum kodu okuma refleksini
verin: 200 başarı, 401/403 yetki/servis, 500 SOAP/biçim. Bu lab, GUI
yerine **programatik** yönetimin kapısıdır; büyük provisioning işleri
(BAT'ın ötesi) buradan otomatikleşir.

---

### Sonraki

**Discovery 12: İzleme Araçlarıyla Dial Plan Sorun Giderme** — Dialed Number
Analyzer (DNA), RTMT ve çağrı izleme ile dial plan arızalarını teşhis.
