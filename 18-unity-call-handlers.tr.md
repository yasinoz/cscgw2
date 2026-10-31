# Discovery 18 — Cisco Unity Connection Call Handler'ları Yapılandır

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe; menü/alan
> adları ve değerler **birebir**. Süre: **100–120 dk**.
> **Ön koşul:** Discovery 16 ve 17 tamamlanmış olmalı.

---

## Giriş

**Call handler**'lar, gelen çağrının voicemail sisteminde **nasıl
karşılandığını ve yönlendirildiğini** belirler — karşılamalar (greetings),
tuş menüleri (caller input), transfer kuralları. Bunlarla bir **otomatik
santral (auto-attendant / audiotext)** uygulaması kurarsın.

Handler türleri:
- **System Call Handler** (ör. **Opening Greeting**, **Operator**): karşılama
  + caller input + transfer.
- **Directory Handler**: arayanın isim/numarayla kullanıcı bulmasını sağlar.
- **Interview Handler**: arayana sorular sorup yanıtları **tek bir voicemail**
  olarak kaydeder.
- **Routing Rules** (direct/forwarded): çağrının hangi handler'a gideceğini
  arayan numara/konuma göre seçer.

## Ortam

Unity `https://10.1.5.8/cuadmin` (HQ-UCX), CUCM `10.1.5.5`. PC-1 CIPC-1
`11001`, PC-2 CIPC-2 `11002`. (16 voicemail portu Discovery 16'dan geliyor.)

---

## Görev 1: Routing Rule (çağrıyı handler'a yönlendir)

**Call Management > Routing Rules > Direct Routing Rules > Test**'i düzenle:
- **Status:** `Active` · **Search Scope:** `cucx-pub Search Space`
- **Send Call to > Call Handler:** `Opening Greeting`, **Attempt Transfer** →
  **Save**.

**Test:** CIPC-1'den `18070` ile Unity'yi ara → **Opening Greeting** karşılaması
duyulur ("…messaging system…" anonsu).

> **Direct** kural doğrudan gelen çağrılara, **Forwarded** kural
> yönlendirilerek gelenlere uygulanır. Routing Rule Conditions ile **Calling
> Number**'a göre farklı handler'lara dallanabilirsin.

## Görev 2: RTMT Port Monitor ile İzle

**Cisco Unified Real-Time Monitoring Tool** → Host `10.1.5.8` → **Unity
Connection > Port Monitor** → **Start Polling**. Discovery 16'da oluşturulan
**16 port** idle görünür; çağrı geldikçe caller/called/**reason/redirect** ve
hangi handler'a düştüğü canlı izlenir.

## Görev 3: Mevcut System Call Handler'ları Yapılandır (Opening Greeting / Operator)

**Call Management > System Call Handlers > Opening Greeting** (ve **Operator**):
- **Greetings:** Standart karşılamayı düzenle — **Callers Hear**, **After
  Greeting** (Call Action), transfer davranışı.
- **Caller Input:** Tuşların ne yapacağını ata (ör. `0` → Operator, `#` → sign-in).
- **Transfer Rules:** Operatör çağrısının nereye (uzantı/kullanıcı) aktarılacağı.

> Opening Greeting = santralin "ilk sesi". Operator = `0`'a basınca ulaşılan
> canlı/handler hedefi. İkisi doğru değilse arayan ya kaybolur ya operatöre
> ulaşamaz.

## Görev 4: Özel Call Handler'larla Audiotext Uygulaması

Bir **İK (Human Resources)** menüsü kuralım: arayan `1` ile iş ilanlarını
dinlesin, `2` ile İK temsilcisine ulaşsın.

**Handler'ları oluştur:** **System Call Handlers > Add New** ile:
`Human Resources`, `Job Listings`, `Representative in Human Resources`.

**Her handler'ın karşılaması (Greetings > Standard):**
- **Callers Hear:** `My Personal Recording` (ve "Play the Record Your Message
  at the Tone" kutusunu **işaretsiz** bırak).
- **After Greeting:** `Call Action` → `Restart Greeting` (menü tekrar çalsın).
- **Recordings:** ilgili `.wav` dosyasını **yükle** (ör. `Desktop\Audio Files`
  altından uygun kayıt) → **Save**.

**Caller Input (Human Resources handler'ında) — Edit > Caller Input:**
- **`*`** → **Call Action** = `Ignore` → Save
- **`1`** → **Call Handler** = `Job Listings`, **Go directly to Greetings** → Save
- **`2`** → **Call Handler** = `Representative in Human Resources`, **Go directly
  to Greetings** → Save

> Böylece bir **ağaç** kurulur: Opening Greeting → (bir tuşla) Human Resources
> → (1) Job Listings / (2) Representative. Audiotext = iç içe call handler'lar.

## Görev 5: Directory Handler (isimle/numarayla bulma)

**Call Management > Directory Handlers**: arayanın dizinde kullanıcı aramasını
sağlar. **Partition** ve **Search Space** ile hangi kullanıcıların bulunabilir
olduğunu sınırla. Çok konumlu ortam için **site-özel** directory handler'lar
oluştur (her site kendi search space'iyle yalnız kendi kullanıcılarını bulsun).

> Directory handler, voicemail'in "rehberi"dir; partition/search space onun
> görünürlük kapsamını belirler (CUCM'deki partition/CSS mantığı).

## Görev 6: Interview Handler (yanıt toplama)

**Call Management > Interview Handlers > Add New**: arayana sırayla **sorular**
(kaydedilmiş prompt'lar) sor, her yanıtı kaydet. Tüm yanıtlar **tek bir
voicemail** olarak belirlenen kullanıcı/dağıtım listesine düşer (ör. destek
talebi formu). Bir call handler caller input'undan buna yönlendir ve test et.

---

## Doğrulama (lab tamam)

- Routing rule çağrıyı Opening Greeting'e yönlendiriyor; RTMT'de port aktivitesi görünüyor.
- Opening Greeting/Operator caller input ve transfer doğru.
- İK audiotext ağacı: `1` → Job Listings, `2` → temsilci çalışıyor.
- Directory handler kullanıcıları (search space kapsamında) buluyor.
- Interview handler yanıtları tek voicemail olarak teslim ediyor.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| Çağrı yanlış karşılamaya düşüyor | Routing rule hedefi/koşulu ya da handler zinciri yanlış. |
| Tuş bir şey yapmıyor | Caller Input'ta o tuş tanımlı değil ya da "Ignore"/yanlış hedef. |
| Karşılama çalmıyor | Greeting "My Personal Recording" seçili ama kayıt yüklenmemiş. |
| Directory kullanıcı bulamıyor | Handler'ın search space'i o kullanıcıların partition'ını içermiyor. |
| Menü tekrar etmiyor | After Greeting "Restart Greeting" değil; arayan menüyü bir kez duyup düşüyor. |

## Eğitmen Notu

Call handler'ı bir **akış şeması** olarak çizin: Routing Rule → Opening
Greeting → caller input dalları → (call handler / directory / interview /
user). Audiotext = iç içe call handler ağacı. "Go directly to Greetings"
ile "Attempt Transfer" farkını vurgulayın. Directory handler'ın search
space'inin partition/CSS mantığıyla aynı olduğunu bağlayın. RTMT Port
Monitor'da **reason/redirect** sütunları hangi handler'a neden düştüğünü
gösterir — sorun gidermenin kalbi budur. Bu lab tüm kursu taçlandırır:
arayan deneyiminin baştan sona (routing → greeting → menü → kutu) tasarımı.

---

### Kurs Sonu

Discovery 18 ile CLCOR-OP 2.0 lab serisi tamamlanır. Bu kılavuz dizisi
(Discovery 0–18) kurulumdan (erişim, sertifika, DHCP/DNS/NTP) telefon
sağlamaya, dial plan ve çağrı ayrıcalıklarına, toll fraud'a, globalizasyona,
API'lere, sorun gidermeye, CUBE/SBC'ye, media resource'lara ve Unity
Connection'a kadar tüm core işbirliği teknolojilerini kapsar.
