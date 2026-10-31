# Discovery 1 — CUCM'de CA-İmzalı Sertifikaların Uygulanması

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Fast Lane Lab Guide'ın yanında kullanılır. Adımlar ve "neden"ler
> Türkçe; menü/düğme adları, komutlar, dosya adları, IP'ler ve kimlik
> bilgileri ekranda göründüğü gibi **birebir** bırakıldı.
> Tahmini süre: **30–45 dk**.

---

## Giriş — Ne Yapıyoruz, Neden?

CUCM varsayılan olarak servislerini **kendinden imzalı (self-signed)**
sertifikalarla sunar. Tarayıcı bu sertifikaya güvenmez; "Your connection
is not private" uyarısı tam da bundan çıkar. Bu lab'da **Cisco Tomcat**
servisinin (CUCM'in HTTPS yüzü) kendinden imzalı sertifikasını, kurumun
**güvenilen CA sunucusunun** imzaladığı bir sertifikayla değiştiriyoruz.

CUCM bir **küme (cluster)** olduğundan, sertifika servis başına **bir kez**
üretilir ve aynı servisi veren diğer düğümlerle paylaşılır.

| Sertifika | Ne işe yarar |
|---|---|
| **Tomcat** | CUCM HTTPS sunucusu. CUCM/IM&P web servislerine erişen herkes bununla sunucuyu doğrular. |
| **CallManager** | Uç noktalar/Jabber, kayıt öncesi CUCM kimliğini bununla doğrular. |
| **CAPF** | LSC için CA politikası; kök yerine ara CA rolü. |
| **IPSec** | Gateway ↔ CUCM IPsec tüneli. |
| **TVS** | Trust Verification Service bağlantısının doğrulanması. |

> **Not (CUCM 14+):** Multi-SAN Tomcat sertifikası CallManager servisi için
> de yeniden kullanılabilir.

Bu lab'da sırasıyla: **(1)** Tomcat için CSR üret, **(2)** CSR'ı özel CA'ya
imzalat, **(3)** CA + imzalı sertifikayı CUCM'e yükle, **(4)** ilgili
servisleri yeniden başlat.

## Ortam Bilgisi (pod)

| Öğe | Değer |
|---|---|
| CUCM (OS Admin) | `https://10.1.5.5/cmplatform` |
| CA sunucusu (AD CS web) | `https://10.1.5.100/certsrv` |
| İş istasyonu | PC-1 (topolojide) |
| Alan adı | `CLL-COLLAB.INTERNAL` |
| CUCM OS Admin / CLI | kullanıcı `admin` · parola `!234QWer` |
| CA web oturumu | kullanıcı `cll-collab\administrator` · parola `1234QWer` |

> Kimlik bilgileri **büyük/küçük harfe duyarlı**. Kendi pod'unda değerler
> farklı olabilir; ekrandakini esas al.

---

## Görev 1: Cisco Tomcat Servisi için CSR Üret

**1.** Topolojide **PC-1**'e bağlan, masaüstünden tarayıcıyı aç ve
`https://10.1.5.5/cmplatform` adresine git.

**2–3.** Kendinden imzalı sertifika uyarısında **Advanced** → **Proceed to
10.1.5.5 (unsafe)**. (Henüz güvenilen sertifika yok; zaten onu kuracağız.)

**4.** **Cisco Unified Operating System Administration** ekranında
`admin` / `!234QWer` ile **Login**.

**5.** Menüden **Security > Certificate Management**.

**6.** Araç çubuğundan **Generate CSR**.

**7.** Açılan pencerede:
- **Certificate Purpose:** `tomcat`
- **Distribution:** `Multi-server(SAN)` *(küme genelinde tek sertifika — tüm düğümler kullanır; tek düğüm istersen düğüm adını seçebilirsin)*
- **Common Name:** otomatik `CUCM-PUB-ms.CLL-COLLAB.INTERNAL`
- **SAN (otomatik):** `CUCM-PUB.CLL-COLLAB.INTERNAL`, `IMP-PUB.cll-collab.internal`
- **Parent Domain:** `CLL-COLLAB.INTERNAL`
- **Key Type:** `RSA` · **Key Length:** `2048` · **Hash Algorithm:** `SHA256`

**Generate** de. (Gerçek kurumda Key Length / Hash kurum politikana göre değişebilir.)

**8.** "Success: Certificate Signing Request Generated" görünce **Close**.

**9–11.** **Download CSR** → **Certificate Purpose: tomcat** → **Download CSR**
→ masaüstündeki **Certificates** klasörüne `tomcat.csr` olarak **Save**.

✅ *Çıktı:* `Desktop\Certificates\tomcat.csr`

---

## Görev 2: CSR'ı Özel CA'da İmzalat

**1.** Tarayıcıda **yeni sekme** → `https://10.1.5.100/certsrv`
(uyarıda **Advanced → Proceed**).

**2.** Oturum aç: `cll-collab\administrator` / `1234QWer` → **Sign In**.

**3.** **Request a certificate** → **advanced certificate request**.

**5–6.** PC-1'de `Desktop\Certificates\tomcat.csr` dosyasını **Notepad++**
ile aç; **Ctrl+A** (tümünü seç), **Ctrl+C** (kopyala).
(`-----BEGIN CERTIFICATE REQUEST-----` … `-----END …-----` bloğunun tamamı.)

**7.** CA sekmesine dön, **Saved Request** kutusuna **Ctrl+V** ile yapıştır.

**8.** **Certificate Template:** `Web Client and Server` seç.
> **Neden bu şablon?** Tomcat hem istek yapan (client) hem isteği karşılayan
> (server) taraf olduğundan sertifikanın **her iki** işlevi de desteklemesi
> gerekir. Yanlış şablon (ör. yalnız Server) seçilirse istemci kimlik
> doğrulaması aksar.

**9.** **Submit**.

**10–11.** "Certificate Issued" sayfasında **Base 64 encoded** → **Download
certificate** → **Certificates** klasörüne `CM_Tomcat.cer` adıyla kaydet.
*(Chrome indirince "Keep" de.)*

**12–15.** CA sertifikasını da indir: sağ üst **Home** → **Download a CA
certificate, certificate chain, or CRL** → **Base 64** seçili → **Download
CA certificate** → `CA.cer` adıyla **Certificates** klasörüne kaydet.
> Lab'daki CA **kök (root)** olduğundan tek bir CA sertifikası yeter.
> Gerçek/genel CA ara (intermediate) ise onun yerine **Download CA
> certificate chain** ile kök+ara zincirini tek dosyada indirirsin.

✅ *Çıktı:* `CM_Tomcat.cer` (imzalı Tomcat) + `CA.cer` (CA kök)

---

## Görev 3: Sertifikaları CUCM'e Yükle

**1.** CUCM sekmesine dön (**Security > Certificate Management** →
**Certificate List**), araç çubuğundan **Upload Certificate/Certificate chain**.

**2–5. CA sertifikasını güven deposuna yükle:**
- **Certificate Purpose:** `tomcat-trust`
- **Choose File:** `CA.cer` → **Open** → **Upload**
- Status'ta "Success: Certificate Uploaded" görünür.

**6. CA'yı CallManager güven deposuna da yükle:**
- **Certificate Purpose:** `CallManager-trust` → `CA.cer` → **Upload**
> İleride Cisco Meeting Server de CA-imzalı sertifika kullanacağı için CA'nın
> CallManager-trust'a da eklenmesi gerekir.

**7. İmzalı Tomcat sertifikasını yükle:**
- **Certificate Purpose:** `tomcat` → `CM_Tomcat.cer` → **Upload**
> Önce CA güven deposunda olmalı (adım 2); CUCM imzalı sertifikayı ancak
> imzalayan CA'ya güveniyorsa kabul eder. Bu yüzden sıra: **önce CA, sonra
> imzalı sertifika.**

**8.** Başarı mesajından sonra pencereyi **Close** et. (Servis restart'ı
Görev 4'te yapacağız.)

**8–9. Doğrula:** Certificate List'te arama kutusuna `tomcat` yazıp **Find**:
- `tomcat-trust` altında CA: **cll-collab-WIN-SERVER01-CA**
- `tomcat` altında imzalı: **CUCM-PUB-ms.CLL-COLLAB.INTERNAL** (Issued By: CA)

---

## Görev 4: İlgili Servisleri Yeniden Başlat

Tomcat sertifikası değişince şu servisler yeniden başlatılmalı:

| Nerede | Servis |
|---|---|
| **CLI** | Cisco Tomcat (tüm düğümlerde), Cisco UDS Tomcat, Cisco AXL Tomcat |
| **Control Center (Serviceability)** | Cisco DRF Master, Cisco DRF Local |

> Tek sertifika değişiyorsa tek tek servis; çok sertifika değişiyorsa düğümü
> baştan başlatmak daha hızlı olabilir.

**1–4. CLI (yayıncı — HQ-UCM):** PC-1 masaüstünde **PuTTY** → **Saved
Sessions** içinde `HQ-UCM` (10.1.5.5) → **Load** → **Open**. Güvenlik
uyarısında **Yes/Accept**. Giriş: `admin` / `!234QWer`.

**5–6.** Şu komutları sırayla çalıştır:

```
utils service restart Cisco Tomcat
utils service restart Cisco UDS Tomcat
utils service restart Cisco AXL Tomcat
```

> Servis `[STOPPING] … [STARTING] … [STARTED]` aşamalarından geçer.
> **Restart sırasında Ctrl+C'ye basma**; takılırsa aynı komutu tekrar çalıştır.

**7–9. CLI (IM&P düğümü — HQ-IMP):** PuTTY penceresinde sağ tık →
**Saved Sessions > HQ-IMP** → `admin` / `!234QWer` → aynı
`utils service restart Cisco Tomcat` (ve gerekli diğerleri) bu düğümde de.

**10–11. Web'e dön:** Tarayıcıda **Cisco Unified Operating System
Administration**'a dön.
> Tomcat restart'ından sonra web sayfaları **5 dk veya daha fazla**
> yanıtsız kalabilir — sabret. Sayfa açılınca **yeni** Tomcat sertifikası
> için tarayıcı tekrar uyarır; **kabul et** (Advanced → Proceed). Gerçekte
> PC'nin güven deposuna CA eklenirse bu adım gerekmez.

Navigation menüsünden **Cisco Unified Serviceability** → **Go**; istenirse
yeniden `admin` / `!234QWer` ile giriş.

**12.** **Tools > Control Center – Network Services**.

**13.** **Server:** `CUCM-PUB.CLL-COLLAB.INTERNAL—CUCM Voice/Video` → **Go**.

**14–15.** **Backup and Restore Services** bölümünde:
- `Cisco DRF Master` seç → **Restart**
- Tamamlanınca `Cisco DRF Local` seç → **Restart**

---

## Doğrulama (lab tamam)

- `https://10.1.5.5/cmplatform` yeni sertifikayla açılıyor; sertifikayı
  inceleyince **Issued By** artık `cll-collab-WIN-SERVER01-CA`.
- Certificate List → `tomcat`: imzalı sertifika (Issued By = CA) ve
  `tomcat-trust` altında CA kök görünüyor.
- CLI'da üç Tomcat servisi `[STARTED]`; DRF Master/Local restart edilmiş.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| İmzalı Tomcat yüklenmiyor / reddediliyor | Önce `CA.cer`'i **tomcat-trust**'a yükle; imzalayan CA güvende değilse imzalı sertifika kabul edilmez. |
| Yükleme sonrası tarayıcı hâlâ uyarıyor | Servisler restart edilmedi ya da yeni sertifika henüz kabul edilmedi; Görev 4'ü tamamla, 5 dk bekle, uyarıyı kabul et. |
| Web 5 dk yanıt vermiyor | Normal — Tomcat restart'ından sonra OS Admin/CM Admin geç açılır. |
| CA web'e girilemiyor | Kullanıcıyı `cll-collab\administrator` biçiminde (alan adıyla) gir. |
| CSR yapıştırınca şablon listesi boş | `advanced certificate request` yolundan gittiğinden emin ol; User Certificate değil. |
| Yanlış şablon seçildi | `Web Client and Server` olmalı (client+server); yeniden submit gerekir. |

## Eğitmen Notu

Bu lab'ın özü **sıra ve güven zinciri**: önce CSR, sonra CA imzası, sonra
**önce CA'yı güven deposuna** sonra imzalı sertifikayı yükleme, en son
**restart**. Öğrencilerin en çok takıldığı iki yer: (1) imzalı sertifikayı
CA'dan önce yüklemeye çalışmak, (2) restart'ı atlayıp "neden hâlâ uyarıyor"
demek. Tahtaya güven zincirini (CA → imzalı Tomcat) bir kez çizin.
Multi-server(SAN) seçiminin anlamını vurgulayın: tek sertifika tüm kümede.
Restart sonrası web'in 5 dk gelmemesi normaldir; sınıfı buna hazırlayın
(panik restart dalgası olmasın).

---

### Sonraki

**Discovery 2: Destekleyici IP Ağ Protokollerinin Yapılandırılması** —
CUCM DHCP servisi, PC-2'yi DHCP istemcisi yapma, DNS (A/PTR/SRV kayıtları)
ve Cisco yönlendiricide NTP.
