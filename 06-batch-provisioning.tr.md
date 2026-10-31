# Discovery 6 — Batch Provisioning ile IP Telefon Dağıtımı

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe; menü/alan
> adları, dosya adları ve değerler **birebir**. Süre: **30–45 dk**.

---

## Giriş

Büyük uç nokta dağıtımlarında **Bulk Administration Tool (BAT)** verimliliğin
anahtarıdır: doğru CSV hazırlamak, doğru servisleri açmak ve **iş (job)
sonuçlarını** doğru yorumlamak. Bu lab'da mevcut telefonları dışa aktarır,
CSV'yi düzenler, geri içe aktarır ve gerçek dünyada sık görülen iki tuzağı
loglardan teşhis edersin.

## Ortam

CUCM `http://10.1.5.5/ccmadmin` (`admin`/`!234QWer`). Telefonlar:
**HQ-CIPC-1** (John Doe, 4085551001), **HQ-CIPC-2** (Jane White, 4085551002).

---

## Görev 1: Config Yükle

PC-1 → **Classfiles > Discoveries** → **lab6.py** → "successfully loaded".

## Görev 2: Telefonları Dışa Aktar (Export)

**1–2.** PC-1/PC-2'de **Cisco IP Communicator**'ları aç; HQ-CIPC-1 John Doe,
HQ-CIPC-2 Jane White olarak kayıtlı olsun.

**3–4.** `http://10.1.5.5/ccmadmin` → **Device > Phone > Find**: iki CIPC görünür.

**Dışa aktar:** **Bulk Administration > Phones > Export Phones** →
(All Details) → CIPC'leri bulan bir sorgu → dosya adı **`EXP_CIPC`** →
**Run Immediately** → **Job Scheduler**'da iş **Completed** olunca
**Bulk Administration > Upload/Download Files** → **EXP_CIPC...** dosyasını
**Download Selected** ile PC-1 masaüstüne indir.

## Görev 3: Dışa Aktarılan Kayıtları Düzenle

CSV'yi herhangi bir araçla düzenleyebilirsin; ama **dosya biçimini bozma**,
yoksa içe aktarma işi başarısız olur.

**1.** Masaüstünde **EXP_CIPC** dosyasına sağ tık → **Edit with Notepad++**.

**2–3.** **Search > Find** → `408555XXX` (her iki telefonun dış numara
maskesi) → maskeyi **`12345678X`** yap (ikisi için) → dosyayı masaüstüne
**`IMP_CIPC.txt`** olarak kaydet.

**Hata dosyası (doğrulama pratiği için):** Aynı içeriği kopyalayıp bir
kaydın **Device Pool** alanını geçersiz bir değere (`HQ` — sistemde böyle
bir device pool yok) ayarla ve **`IMP_CIPC_mistake.txt`** olarak kaydet.

> Amaç: Geçersiz device pool'lu bir kaydın işi nasıl etkilediğini ve logda
> nasıl göründüğünü görmek.

## Görev 4: Telefonları Batch Provisioning ile Ekle (Insert)

**1–2. CSV'leri yükle:** **Bulk Administration > Upload/Download > Add New**:
- **File:** `IMP_CIPC.txt` · **Select The Target:** `Phones`
- **Select Transaction Type:** `Insert Phones - All Details` → **Save**

**3.** Aynı yolla **`IMP_CIPC_mistake.txt`**'i de yükle.

**4.** **Upload/Download Files > Find**: iki CSV + `bat.xlt` görünür.

**5–7. Çakışmayı önle:** **Device > Phone > Find** → iki telefonu **sil**;
PC-1/PC-2'de CIPC'ler artık **kayıtsız**. (Autoreg de kapalı olmalı.)

**8–11. Önce HATALI dosyayla iş:** **Bulk Administration > Phones > Insert
Phones**:
- ☑ **Insert Phones All Details** · **File Name:** `IMP_CIPC_mistake.txt`
- **Job Information > Run Immediately** → **Submit**.

**Job Scheduler** → iş **Completed** ama **Job Results**: 2 işlendi, **1
başarısız**. **Log File Name**'e tıkla:
`HQ-CIPC-1  HQ is not a valid status in device` — yani **device pool `HQ`
geçersiz**. (Status "Completed" yanıltıcıdır; sonucu log belirler.)

**12–14. Şimdi DOĞRU dosyayla iş:** **Insert Phones** → **File Name:**
`IMP_CIPC.txt` → **Run Immediately** → **Submit** → Job Scheduler → iş sonucu
yine **1 başarısız**. Log:
`HQ-CIPC-2 DEVICE :: Could not insert new row - duplicate value in a UNIQUE
INDEX column` — yani **HQ-CIPC-2 zaten eklenmiş** (hatalı iş onu başarıyla
eklemişti; sadece HQ-CIPC-1 kalmıştı).

**16.** **Device > Phone > Find**: **iki telefon da listede**. (HQ-CIPC-2
hatalı işten, HQ-CIPC-1 doğru işten geldi.)

---

## Doğrulama (lab tamam)

- İki telefon CSV ile dışa aktarıldı, maske düzenlendi, geri içe aktarıldı.
- Hatalı device pool → log "HQ is not a valid status in device".
- Tekrar ekleme → log "duplicate value in UNIQUE INDEX" (zaten var).
- Sonuçta iki telefon da CUCM'de kayıtlı.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| İş "Completed" ama telefon yok | Status yanıltıcı; **Job Results** ve **Log File**'a bak (passed/failed). |
| "… is not a valid status in device" | CSV'deki device pool/alan değeri CUCM'de yok; birebir eşleşmeli. |
| "duplicate value in UNIQUE INDEX" | Kayıt zaten var; önce sil ya da Override seç. |
| İçe aktarma hiç çalışmıyor | CSV biçimi bozulmuş (Notepad++'ta kaydet, Excel formatını bozma). |
| Telefonlar çakışıyor | Autoreg kapalı mı, eski kayıtlar silindi mi? |

## Eğitmen Notu

İki dersi vurgulayın: (1) **"Completed" ≠ başarılı** — her zaman Job Results
ve log dosyasına bakın; (2) CSV'deki her değer CUCM'de **birebir** var olmalı
(device pool, CSS, şablon adları). İki hata senaryosu gerçek dünyanın aynası:
yanlış alan değeri ve mükerrer kayıt. Self-Provisioning (Discovery 5) "kullanıcı
kendi sağlar", BAT ise "yönetici toplu sağlar" — ikisini karşılaştırın.

---

### Sonraki

**Discovery 7: Uç Nokta Adresleme ve Çağrı Yönlendirme** — DN ile line
appearance farkı, Directory URI, çakışan DN'ler ve Urgent Priority.
