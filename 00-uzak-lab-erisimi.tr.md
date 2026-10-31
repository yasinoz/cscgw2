# Discovery 0 — Uzak Lab Ortamına Erişim

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Bu kılavuz resmi Fast Lane Lab Guide'ın **yanında** kullanılmak üzere
> hazırlanmıştır. Adımlar ve açıklamalar Türkçe anlatılır; arayüz
> öğelerinin adları (düğme, menü) ve komutlar İngilizce orijinaliyle
> birebir bırakılır — ekranda onları o şekilde görürsünüz.

---

## Etkinliğin Amacı

Lab etkinliklerini yapabilmek için uzaktaki lab ortamına bağlanmak.
Bu etkinliğin sonunda:

- Uzak lab ortamına bağlanabilecek,
- Lab topolojisini görüntüleyebilecek,
- Cihazlara (yönlendirici, CUCM, PC) oturum açabileceksiniz.

## Önkoşul

- İnternet erişimi olan bir bilgisayar ve güncel bir tarayıcı.
- Eğitmenin verdiği **erişim kodu** (access code). Her katılımcının/pod'un
  kodu farklıdır.

> **Ortam bilgisi (örnek):** Pod adı `CI-CLFNDU`, kaynak `voicelab01_1`.
> Kartta ayrıca **Start Time / End Time** (oturumun açık kalacağı aralık)
> yazar. Kendi kartındaki değerler farklı olacaktır.

---

## Görev 1: Uzak Lab Ortamına Bağlan

### Adımlar

**1. Portalı aç.**
Tarayıcıda şu adrese git: **https://portal.remotelabs.io**

**2. Giriş yap.**
Hesabın yoksa ücretsiz bir hesap oluştur (*Create account*), varsa
giriş yap (*Login*).

**3. Lab'ı ekle.**
**Add Lab** düğmesine tıkla. Açılan **ADD NEW LAB** penceresinde
eğitmenin verdiği erişim kodunu **Please enter your access code** alanına
gir ve **ADD** de. (Pencereyi kapatmak için **CLOSE**.)

**4. Lab'ı aç.**
Kod eklenince beliren kartta pod bilgilerini görürsün (Access Code, POD,
Start/End Time). **VIEW LAB** düğmesine tıklayarak pod kaynaklarına eriş.

**5. İstemciyi/arayüzü aç ve topolojiyi gör.**
**RemoteLabsClient** açılır. Ortada **LAB TOPOLOGY**, sağda ise
**LAB GUIDE** ve varsa ek bilgiler görünür. Sağ taraftaki ek bilgileri
mutlaka kontrol et (IP planı, parolalar, uyarılar burada olabilir).

> **Tarayıcıdan bağlanmak istiyorsan (indirme yapmadan):**
> Sağ üstteki kullanıcı adına tıkla → **Settings** → **Connection** sekmesi
> → **Connection Technology** açılır menüsünden **Guacamole** seç, uygun
> bir **Resolution** (çözünürlük) belirle ve **Update** de. Guacamole,
> oturumu doğrudan tarayıcıda (HTML5) açar; ayrı bir RDP istemcisi gerekmez.

**6. Bir cihaza oturum aç.**
Topolojideki herhangi bir cihazın **RDP** ya da **console** ikonuna
tıklayarak oturumu başlat. Örneğin **PC-1** (`10.1.5.200`) üzerinde
*Double-click for Rdp*.

> **Önemli — bant genişliği:** Ortamdaki bazı PC'lerde **çift yönlü ses**
> açıktır ve bu ciddi bant genişliği harcar. Mümkünse **yalnızca PC-1'e**
> bağlan. PC-1 masaüstünde diğer PC'lere bağlanmak için kısayollar vardır:
> `PC-2.rdp`, `PC-3.rdp`. Diğer PC'lere oradan geç.

### Doğrulama

Aşağıdakiler sağlanıyorsa erişim tamamdır:

- RemoteLabsClient'te lab topolojisi görünüyor.
- PC-1'e RDP/Guacamole ile masaüstü açılıyor.
- Yönlendirici/CUCM gibi cihazların konsol ya da web arayüzüne
  erişebiliyorsun.

---

## İpuçları ve Sık Karşılaşılan Sorunlar

| Belirti | Olası sebep / çözüm |
|---|---|
| Kart "süresi doldu" gösteriyor | **End Time** geçmiş. Eğitmenden kodu/oturumu yenilet. |
| Oturum açılmıyor / siyah ekran | Önce **Guacamole**'yi dene (tarayıcı içi); yerel RDP istemcisi engelliyse bu yol çalışır. |
| Ses yok ya da kesik | Çift yönlü sesli PC'ye bağlan (PC-1); diğer PC'ler bant yüzünden tekler. |
| Topoloji boş görünüyor | **VIEW LAB** ile pod'un gerçekten başladığından ve Start Time'ın geçtiğinden emin ol. |
| Kopyala-yapıştır çalışmıyor | Guacamole'de pano menüsünü kullan; yerel RDP'de istemci ayarından panoyu aç. |

---

## Eğitmen Notu

Bu etkinlik "sıfırıncı" adımdır: içerik değil, **erişim** kazandırır.
Derse başlamadan önce herkesin PC-1'e bağlanabildiğini tek tek doğrulatın
— en çok zaman kaybı burada yaşanır. Çift yönlü ses uyarısını baştan
vurgulayın; 15 kişi aynı anda tüm PC'lere bağlanırsa pod yavaşlar.
Sağ paneldeki **LAB GUIDE / Introduction** bölümünde pod'a özel IP planı
ve parolalar olabilir; öğrencilere oradan okumalarını söyleyin.

---

### Sonraki

**Discovery 1: Cisco Unified Communications Manager'da CA-İmzalı
Sertifikaların Uygulanması** — Tomcat servisi için CSR üretme, sertifikayı
alma, CUCM'e yükleme ve servisleri yeniden başlatma.
