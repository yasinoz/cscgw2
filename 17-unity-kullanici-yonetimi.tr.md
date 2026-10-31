# Discovery 17 — Unity Connection Kullanıcılarını Yönet

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe; menü/alan
> adları ve değerler **birebir**. Süre: **60–75 dk**.
> **Ön koşul:** Discovery 16 tamamlanmış olmalı.

---

## Giriş

Güvenilir sesli mesaj, **Unity Connection kullanıcılarının** doğru
yapılandırılmasına dayanır: kimlik doğrulama kuralları, zaman çizelgeleri,
şablonlar ve kutu ayarları. Bu lab'da sistem genelini ayarlar, kullanıcı
şablonu özelleştirir, kullanıcı oluşturur/CUCM'den içe aktarır, yönetici rolü
atar, çağrı giriş (caller input) ve alternatif numaraları yapılandırır, MWI
ve kutu işlevini doğrular, sonra kullanıcıları tekil ve toplu silersin.

## Ortam

| Öğe | Değer |
|---|---|
| Unity Connection (HQ-UCX) | `https://10.1.5.8/cuadmin` |
| CUCM | `10.1.5.5` |
| Telefonlar | PC-1 CIPC-1 `11001`, PC-2 CIPC-2 `11002` |

---

## Görev 1: Kimlik Doğrulama Kuralı (Authentication Rule)

**System Settings > Authentication Rules > Recommended Web Application
Authentication Rule**:
- **Failed Sign-In:** `7` Attempts · **Reset Every Failed Sign-In:** `30` Minutes
- **Lockout Duration:** `30` Minutes
- **Minimum Credential Length:** `8`
- **Check for Trivial Passwords:** işaretsiz (lab; üretimde **işaretli** olmalı)
- **Save**

> İki ayrı kural vardır: **Web Application** (yönetim/Self-Care parolası) ve
> **Voicemail (telefon PIN'i)**. Sayısal PIN için ayrı kuralı ayarla.

## Görev 2: Class of Service (CoS)

**Class of Service > Class of Service > Voice Mail User COS** (ya da Add New):
kutu boyutu, mesaj uzunluğu, dış arama (transfer), TTS, IMAP erişimi gibi
**yeteneği** belirler. Kullanıcılar bir CoS'a bağlanır.

> CoS = "bu kullanıcı neler yapabilir" (kutu kotası, dış erişim). Partition/CSS'in
> voicemail dünyasındaki karşılığı gibi düşün.

## Görev 3: Zaman Çizelgeleri (Schedules) ve Tatiller

**System Settings > Schedules** → bir çizelge (ör. **Weekdays**) ve
**Schedule Detail** ekle:
- Örnek **Saturday Hours**: **Start Time** `08:00 AM`, **End Time** `12:00 PM`
  (End Of Day işaretsiz), **Active Saturday** işaretli → **Save**.
- **Holiday** ayarlarıyla tatil günleri tanımlanır.

> Çizelge, otomatik karşılamaların "mesai içi/dışı" davranışını belirler
> (açık/kapalı karşılama, operatöre aktarma vb.).

## Görev 4: Kullanıcı Şablonu (User Template)

**Templates > User Templates** → bir şablonu düzenle/kopyala. Şablon;
**CoS**, **şablon**-bazlı uzantı/partition, zaman çizelgesi, dil, kutu
varsayılanlarını taşır. Yeni kullanıcılar bu şablondan **standart** oluşur.

> Şablon, CUCM'deki Feature Group/User Profile mantığının Unity karşılığıdır:
> tek yerden tutarlı kutu üretimi.

## Görev 5: Kullanıcı Oluştur ve CUCM'den İçe Aktar

- **Tekil:** **Users > Users > Add New** → şablon seç, Alias/uzantı (ör. 11001)
  gir → kutu oluşturulur ve **initialize** edilir.
- **İçe aktar (CUCM):** **Users > Import Users** → kaynak **Cisco Unified
  Communications Manager (AXL)** → CUCM'den kullanıcıları listele → şablon seç →
  seçilenleri **Import**. (CUCM↔Unity AXL bağlantısı Phone System üzerinden
  kurulur.)

> İçe aktarma, kullanıcıyı CUCM verisiyle (ad, uzantı) oluşturur; böylece
> voicemail kutusu doğru uzantıya bağlanır.

## Görev 6: Yönetici Rolü Ata

**Users > Users >** ilgili kullanıcı **> Roles** → gereken yönetim rolünü
(ör. System Administrator, Help Desk) ata. Rol, Unity yönetim arayüzünde neyi
görüp değiştirebileceğini belirler.

## Görev 7: Caller Input ve Alternatif Numaralar

- **Caller Input:** Kullanıcının karşılaması çalarken arayanın bastığı tuşların
  (0–9, *, #) ne yapacağını tanımla (ör. `0` → operatör, `1` → başka uzantı).
- **Alternate Extensions:** Kullanıcıya ek numaralar bağla; bu numaralardan
  gelen/giden çağrılar da bu kutuya/kullanıcıya eşlenir (cep, DID vb.).

## Görev 8: MWI ve Kutu İşlevini Doğrula

1. CIPC-1'den (11001) **CIPC-2'yi (11002, Jane White)** ara; yanıtlama,
   karşılama çalınca **mesaj bırak**, kapat.
2. CIPC-2'de **MWI** (zarf/kırmızı ışık) yanar, ekranda "You Have VoiceMail".
3. CIPC-2'de **Messages** düğmesi → PIN (ör. `0301`) ile gir, mesajı dinle.
4. **3**'e basıp mesajı sil → MWI ışığı ve zarf **söner**.

## Görev 9: Kullanıcı Silme (Tekil ve Toplu)

- **Tekil:** **Users > Users >** kullanıcıyı seç → **Delete Selected**.
- **Toplu:** **Bulk Administration Tool** ile CSV üzerinden toplu sil (ya da
  listede çoklu seçip Delete Selected).

> Silme kutuyu da siler (mesajlar dahil) ve **geri alınamaz** — üretimde önce
> yedek/arşiv.

---

## Doğrulama (lab tamam)

- Auth rule, CoS ve schedule tanımlı; kullanıcılar şablondan tutarlı oluşuyor.
- CUCM'den içe aktarılan kullanıcıların kutusu doğru uzantıya bağlı.
- Mesaj bırakınca MWI yanıyor, dinleyip silince sönüyor.
- Caller input / alternate extension çalışıyor; kullanıcılar silinebiliyor.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| MWI yanmıyor/sönmüyor | CUCM↔Unity MWI (unsolicited notification) ve port ayarı; Discovery 16'yı kontrol et. |
| Kullanıcı kutusuna düşmüyor | Uzantı/alias yanlış ya da import eksik; CoS kutu kotası dolu olabilir. |
| PIN kabul edilmiyor | Voicemail auth kuralı (uzunluk/trivial) ve kullanıcı PIN'i. |
| Karşılama yanlış (mesai) | Schedule/holiday ayarı; açık/kapalı karşılama. |
| Import boş geliyor | Phone System/AXL bağlantısı (CUCM 10.1.5.5) kurulu değil. |

## Eğitmen Notu

Unity kullanıcı dünyasını CUCM'e eşleyin: **CoS ≈ yetki (kota/dış erişim)**,
**User Template ≈ Feature Group/User Profile**, **Schedule ≈ Time Schedule**,
**Auth Rule ≈ kimlik/parola politikası**. İçe aktarmanın neden tercih
edildiğini vurgulayın (CUCM verisiyle tutarlı kutu). MWI test döngüsünü canlı
yaptırın (bırak → yanar → dinle+sil → söner); voicemail sorun gidermenin
görünür ipucu budur. Silmenin geri alınamazlığını hatırlatın. Ön koşul:
Discovery 16 (entegrasyon) yapılmadan bu lab tamamlanamaz.

---

### Sonraki

**Discovery 18: Unity Connection Call Handler'ları Yapılandır** — call handler
ağacı, karşılama (greetings), caller input menüleri, directory handler ve
otomatik santral (auto-attendant) akışı.
