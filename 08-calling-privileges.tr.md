# Discovery 8 — Çağrı Ayrıcalıkları (Calling Privileges)

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe; menü/alan
> adları, desen ve CSS/partition adları **birebir**. Süre: **30–45 dk**.

---

## Giriş

**Class of Control (CoC)** — eski adıyla Class of Service — kimin nereye
arayabileceğini belirler. İki yapı taşı:
- **Partition (bölüm):** Aranabilir hedeflerin (DN, route pattern, URI) konduğu
  "kutu". Numaralar/desenler bir partition'da yaşar.
- **Calling Search Space (CSS):** Arayanın **hangi kutulara bakabileceğinin**
  sıralı listesi. CSS'te olmayan partition'daki hedefe erişilemez.

**Kural:** Hedef, arayanın CSS'indeki bir partition'da yoksa → çağrı kurulmaz.

### Line/Device modeli
- **Device CSS:** Siteye özgü PSTN route pattern'lerine erişimi yönetir
  (çıkış gateway'i seçimi).
- **Line CSS:** İç hedefleri ve **engellenen PSTN desenlerini** yönetir
  (PSTN arama sınıfı: yerel/şehirlerarası/uluslararası).

### Lab ortamı
CUCM HQ'da üç CIPC: PC-1 (HQ-CIPC-1/11001), PC-2 (HQ-CIPC-2/11002),
PC-3 (HQ-CIPC-3/11003). **PC-4 = PSTN simülatörü** (CME router üzerinde,
TFTP `10.1.99.252`). PSTN'e erişim: **yerel** `408[2-9]XXXXXX`, **ulusal**
`1[2-9]XX[2-9]XXXXXX`, **uluslararası** `011!`, **acil** `911`.
CUCM `https://10.1.5.5/ccmadmin` (`admin`/`!234QWer`).

---

## Görev 1: Config Yükle

PC-1 → **lab8.py** → "successfully loaded". Ardından PC-1'den **PuTTY** ile
**her iki router**'a bağlan ve startup config'i yükle:

```
copy flash:/CLCOR-lab8.txt startup-config
```
Sonra her iki router'ı **reload** et, açılmalarını bekle. Dört CIPC'yi kaydet
(HQ-CIPC-1/2/3 → TFTP 10.1.5.5; PSTN → TFTP 10.1.99.252).

## Görev 2: Mevcut Partition ve CSS'leri İncele

Tasarıma başlamadan mevcut dial plan'ı anla.

**Call Routing > Class of Control > Partition > Find** — üç **engelleme**
partition'ı: `BLOCK_INTERNATIONAL_PT`, `BLOCK_LOCAL_PT`, `BLOCK_LONGDISTANCE_PT`.
(Ayrıca `DEVICE_HEADQUARTERS_PT`, `LINE_INTERNAL_PT`, `Directory URI` …)

**Mevcut PSTN akışı:** Route pattern'ler (`9.011!` uluslararası,
`9.1[2-9]XX[2-9]XXXXXX` ulusal, `9.408[2-9]XXXXXX` yerel, `9.911`/`911` acil,
`9.[2-8]XX` servis) → route list **HQ_SIP_ROUTE_LIST** → route group
**HQ_SIP_ROUTE_GROUP** → `10.1.5.252` SIP trunk (ROUTER-1) → PSTN.

**CSS kombinasyonları (Class of Control > Calling Search Space):**

| CSS | İçerdiği partition'lar | Anlamı |
|---|---|---|
| `LINE_INTERNATIONAL_CSS` | LINE_INTERNAL_PT | Hiç engel yok → her yere arar |
| `LINE_LONGDISTANCE_CSS` | BLOCK_INTERNATIONAL_PT + LINE_INTERNAL_PT | Uluslararası engelli |
| `LINE_LOCAL_CSS` | BLOCK_INTERNATIONAL_PT + BLOCK_LONGDISTANCE_PT + LINE_INTERNAL_PT | Sadece yerel |
| `LINE_INTERNAL_ONLY_CSS` | 3 blok + LINE_INTERNAL_PT | Sadece iç |

> **Engelleme nasıl çalışır?** Blok partition'larda ilgili route pattern'ler
> **"Block this pattern"** ile durur. CSS bu blok partition'ı içeriyorsa, en
> yakın eşleşen **blok** deseni kazanır ve çağrı reddedilir. Uluslararası
> **izin** vermek = blok partition'ı CSS'e **koymamak**.

## Görev 3: Line/Device Yaklaşımını Uygula

**Line CSS atamaları (Device > Phone > ilgili hat):**

| Uç nokta / DN | Line CSS | Sonuç |
|---|---|---|
| HQ-CIPC-1 / 11001 | `LINE_INTERNATIONAL_CSS` | Her yere arar (yönetici) |
| HQ-CIPC-2 / 11002 | `LINE_LONGDISTANCE_CSS` | Uluslararası engelli |
| HQ-CIPC-3 / 11003 | `LINE_LOCAL_CSS` | Sadece yerel (lobi) |

**CSS'leri genişlet:** **Class of Control > Calling Search Space >
LINE_INTERNATIONAL_CSS** → `LINE_INTERNAL_LOBBY_PT` ve
`LINE_INTERNAL_MANAGER_PT` partition'larını **Selected** listesine taşı →
**Save**. (Diğer LINE CSS'lere de gereken iç partition'ları ekle; not: ilgili
CSS'te manager partition istenmiyorsa **eklenmez**.)

**DN partition'ları (Call Routing > Directory Number):**
- `11001` → `LINE_INTERNAL_MANAGER_PT`
- `11002` → `LINE_INTERNAL_PT`
- (`*XXXXX` sesli mesaja transfer, `18070` IT paylaşımlı hat → LINE_INTERNAL_PT)

> Manager hattı kendi partition'ında; sadece onu görebilen CSS'ler erişir.
> Bu, "kim kimi arayabilir" matrisini partition/CSS ile kurar.

## Görev 4: Directory URI'lere Erişim için Ayrıcalıklar

Kullanıcı-atamalı URI'ler otomatik **`Directory URI`** partition'ına girer;
line CSS bu partition'ı içermezse URI ile arama **başarısız** olur.

**1–3.** **Call Routing > Directory Number > 11001:** URI
`jdoe@cll-collab.internal`, partition **`Directory URI`**. **11002:** URI
`jwhite@cll-collab.internal`, partition **`Directory URI`**.

**4–5. Başarısızlığı gör:** HQ-CIPC-1'den `jwhite@cll-collab.internal`'i
hızlı aramayla ara → **Reorder** (başarısız). Çünkü 11001'in line CSS'i
(`LINE_INTERNATIONAL_CSS`) **Directory URI** partition'ını içermiyor. Aynısı
HQ-CIPC-2 için de olur.

**6–8. Düzelt:** `LINE_INTERNATIONAL_CSS` ve `LINE_LONGDISTANCE_CSS`'e
**`Directory URI`** partition'ını ekle → **Save**. Artık URI çağrıları
çalışır: HQ-CIPC-2 → jdoe → **To John Doe (11001)**; HQ-CIPC-1 → jwhite →
**To Jane White (11002)**.

## Görev 5: Zaman Bazlı (Time-of-Day) Yönlendirme

Gereksinim: uluslararası aramaya **hafta sonu hariç** her zaman izin ver.
Bunu, blok partition'ını yalnızca hafta sonu **aktif** kılarak yaparız.

**Time Period (Class of Control > Time Period > Add New):**
- **Name:** `Sa-Su` · **Description:** `Limit routing calls to a working week only`
- **Time Of Day Start:** `00:00` · **Time of Day End:** `24:00`
- **Repeat Every:** `Week from Saturday to Sunday` → **Save**

> Lab testinde "bugün iş günü" varsayılır; Sa-Su zaman aralığı güncel günü
> **dışarıda** bırakır, böylece iş günü araması engellenmez. Üretimde gerçek
> hafta sonu günleri yapılandırılır.

**Time Schedule (Class of Control > Time Schedule > Add New):**
- **Name:** `Weekend` · **Description:** `Deny access only during weekend`
- İçine **`Sa-Su`** time period'ını ekle → **Save**.

**Partition'a uygula:** **Class of Control > Partition >
`BLOCK_INTERNATIONAL_PT`** → **Time Schedule:** `Weekend`, **Time Zone:**
`Originating Device` → **Save**.

**Test:** HQ-CIPC-2'den (`LINE_LONGDISTANCE_CSS`, içinde BLOCK_INTERNATIONAL_PT)
uluslararası `9011442079460132` ara. İş gününde **çağrı çalışır** — çünkü
BLOCK_INTERNATIONAL_PT yalnızca hafta sonu aktif, iş günü pasif, dolayısıyla
engel devrede değil.

---

## Doğrulama (lab tamam)

- Her telefon kendi sınıfında: 11001 her yere, 11002 uluslararası hariç,
  11003 sadece yerel.
- URI çağrıları line CSS'e Directory URI partition'ı eklendikten sonra çalışıyor.
- BLOCK_INTERNATIONAL_PT yalnızca hafta sonu aktif; iş günü uluslararası çağrı geçiyor.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| Çağrı Reorder (reddedildi) | Hedef partition arayanın CSS'inde yok; ya da blok deseni eşleşti. |
| URI çağrısı başarısız | Line CSS'e **Directory URI** partition'ı eklenmeli. |
| İzinli olması gereken arama engelli | CSS'te gereksiz blok partition var; çıkar. |
| Time-of-Day beklendiği gibi değil | Time Zone (Originating Device) ve time period aralığı doğru mu? |
| PSTN'e hiç çıkamıyor | Device CSS, DEVICE_HEADQUARTERS_PT'i (route pattern'ler) içermeli. |

## Eğitmen Notu

CoC'u iki cümleyle oturtun: **partition = hedefin kutusu**, **CSS = bakılacak
kutuların listesi**. Engellemenin mantığını vurgulayın: blok partition'da
desen "Block this pattern" ile durur; CSS onu içeriyorsa en yakın eşleşme
blok olur. Line/Device ayrımını anlatın: device CSS PSTN çıkışını, line CSS
sınıfı ve engelleri yönetir. Time-of-Day'de "partition'ı zamanla aç/kapat"
fikrini verin. Bu lab Modül 3'teki COR konusunun CUCM karşılığıdır (IOS'ta COR,
CUCM'de partition/CSS).

---

### Sonraki

**Discovery 9: Toll Fraud Önleme** — çağrı sınıflandırma, bloklama,
after-hours kısıtları ve CUBE/gateway üzerinde toll fraud koruması.
