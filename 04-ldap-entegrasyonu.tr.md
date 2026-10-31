# Discovery 4 — LDAP Entegrasyonu: Yapılandırma ve Sorun Giderme

**CLCOR-OP 2.0 · Türkçe Çalışma Kılavuzu (companion)**

> Resmi Fast Lane Lab Guide'ın yanında kullanılır. Adımlar/"neden"ler Türkçe;
> menü/alan adları, değerler, DN'ler ve kimlikler **birebir**. Süre: **60–75 dk**.

---

## Giriş

CUCM'i Microsoft Active Directory ile birleştirdiğinde kullanıcı sağlama ve
oturum açma, **LDAP senkronizasyonu** ve **kimlik doğrulama** ayarlarının
doğruluğuna bağlıdır. Bu lab'da bir CUCM yöneticisi gibi LDAP sync ve auth'u
kurar, doğrular ve sorun giderirsin.

- **Senkronizasyon (DirSync):** Kullanıcıları AD'den CUCM'e kopyalar. CUCM
  kullanıcıyı "okur", AD'de yönetirsin.
- **Kimlik doğrulama (Authentication):** Parola kontrolünü AD'ye devreder;
  CUCM parola tutmaz, AD'ye sorar.

## Ortam

| Öğe | Değer |
|---|---|
| CUCM (CM Admin / Serviceability) | `https://10.1.5.5/ccmadmin` · `/ccmservice` |
| CUCM Self-Care | `https://10.1.5.5/ucmuser` |
| LDAP/AD sunucusu (WIN-SERVER01) | `10.1.5.100` · `administrator` / `1234QWer` |
| CUCM admin | `admin` / `!234QWer` |
| LDAP bağlama (bind) | `Administrator@cll-collab.internal` / `1234QWer` |
| Kullanıcı arama tabanı (doğru) | `cn=Users, dc=cll-collab, dc=internal` |

---

## Görev 1: Config Yükle

PC-1 → **Classfiles > Discoveries** → **lab4.py** çift tık → "successfully
loaded" → Enter.

## Görev 2: Elle (Yerel) Kullanıcı Oluştur

CUCM 9.0'dan beri LDAP ve yerel kullanıcılar aynı veritabanında yaşar.
Yerel bir kullanıcı oluşturup ileride LDAP'ın onu nasıl etkilediğini göreceğiz.

`https://10.1.5.5/ccmadmin` (`admin`/`!234QWer`) → **User Management >
End User > Add New**:
- **User ID:** `jdoe` · **Password/Confirm:** `!234QWer`
- **PIN/Confirm:** `12345` · **Last Name:** `Doe` · **First Name:** `John`
- **Save** → Related Links **Back to Find List Users > Go** → **Find**:
  jdoe, **User Status = Enabled Local User**.

## Görev 3: LDAP Senkronizasyonu

**1–6. DirSync'i aç:** `https://10.1.5.5/ccmservice` (`admin`/`!234QWer`)
→ **Tools > Service Activation** → Server **CUCM-PUB...—CUCM Voice/Video**
→ **Go** → **Directory Services** altında **Cisco DirSync**'i işaretle →
**Save** → OK. Sonra Navigation **Cisco Unified CM Administration > Go**.

**7–8. LDAP System:** **System > LDAP > LDAP System**:
- ☑ **Enable Synchronizing from LDAP Server**
- **LDAP Server Type:** `Microsoft Active Directory`
- **LDAP Attribute for User ID:** `sAMAccountName` → **Save**

**9–14. LDAP Directory:** **System > LDAP > LDAP Directory > Add New**:
- **LDAP Configuration Name:** `LDAP Server`
- **LDAP Manager Distinguished Name:** `Administrator@cll-collab.internal`
- **LDAP Password:** `1234QWer`
- **LDAP User Search Base:** `cn=Users, dc=cll-collab, dc=internal`
- **Perform a Re-sync Every:** `6 HOUR`
- **User Fields:** Middle Name `middleName`, Phone Number `ipPhone`, Directory URI `mail`
- **Add to Access Control Group** → **Find** → ☑ **Standard CCM End Users** → **Add Selected**
- **Host Name or IP Address for Server:** `10.1.5.100` (LDAP Port `389`) → **Save**

> Access Control Group "Standard CCM End Users" sayesinde kullanıcılar
> Self-Care Portal'a erişebilir. Üretimde kök yerine daha dar bir arama
> tabanı/filtre ve ayrı bir bind kullanıcısı kullanılır.

**15–17.** **Perform Full Sync Now** → OK. Birkaç dk sonra **User
Management > End User**: kullanıcılar replike olmuş (dgrey, jwhite, jgreen,
jdoe, mbrown, mblack) — hepsi **Active Enabled LDAP Synchronized User**.

**18.** **jdoe**'yi aç: artık **LDAP Synchronized** — ad/soyad/Directory URI
AD'den gelir, değiştirilemez. **Parola ve PIN değişmedi** (sync yalnız; auth
ayrı). Bu, yerel kullanıcının LDAP'a "dönüşmesi"dir.

## Görev 4: LDAP Kimlik Doğrulama

**1–3.** `https://10.1.5.5/ccmadmin` → **System > LDAP > LDAP Authentication**:
- ☑ **Use LDAP Authentication for End Users**
- **Manager DN:** `Administrator@cll-collab.internal` · **Password:** `1234QWer`
- **User Search Base:** `cn=Users, dc=cll-collab, dc=internal`
- **Server:** `10.1.5.100` (`389`) → **Save**

**4–5. Test:** Çıkış yap → `https://10.1.5.5/ucmuser` → `jdoe` / `!234QWer`.
**Giriş BAŞARISIZ.** Çünkü parola artık AD'de doğrulanıyor ve AD'deki jdoe
parolası `1234QWer`. (CUCM'deki eski yerel parola geçersiz.)

## Görev 5: LDAP Sunucusunda Nesne Özniteliği Değiştir

LDAP açıkken kullanıcı değişiklikleri **AD tarafında** yapılır.

**1–3.** PC-1 → **Remote Desktop Connection** → `10.1.5.100`
(`Administrator`/`1234QWer`) → **Active Directory Users and Computers** →
**Joe Green**'e sağ tık **Rename**:
- Last Name `Brown` · Display Name `Joe Brown` · User logon name `jbrown` → **OK**

**4–5.** CUCM'e dön → **LDAP Directory > LDAP Server > Perform Full Sync Now**
→ birkaç dk → **End User**'da artık **jbrown / Joe Brown** görünür.
(Değişiklik AD'de yapılır, sync ile CUCM'e taşınır.)

## Görev 6: Farklı Namespace Uygula (userPrincipalName)

Çok ağaçlı AD ormanında `sAMAccountName` benzersiz olmayabilir; orman
genelinde benzersiz olan **userPrincipalName** tercih edilir.

> **Not:** LDAP System bilgisini değiştirmeden **tüm LDAP directory'leri
> silmelisin.**

**1.** **System > LDAP > LDAP Directory** → **LDAP Server**'ı **Delete** → OK
(bu, sync edilmiş kullanıcıları da siler).

**2.** **System > LDAP > LDAP System:** **LDAP Attribute for User ID** →
`userPrincipalName` → **Save**.

**3–4.** **LDAP Directory > Add New** → aynı değerlerle (Manager DN, pw,
search base, 10.1.5.100, Standard CCM End Users) yeniden oluştur → **Save**
→ **Perform Full Sync Now**.

**Etki:** Kullanıcı ID'leri artık UPN biçiminde (`jdoe@cll-collab.internal`).

## Görev 7: Sorun Giderme — Senkronizasyonu Doğrula (RTMT ile baseline)

**1–5.** PC-1 → **Cisco Unified Real-Time Monitoring Tool** → Host
`10.1.5.5` → `admin`/`!234QWer` → Configuration **Default** → **System >
Tools > Trace & Log Central**. Sorun çıkmadan **önce** sağlıklı DirSync
kaydını görmek baseline kurar.

## Görev 7 (b): Sorun Giderme — Yanlış Yapılandır ve Teşhis Et

Klasik bir LDAP hatasını bilerek üretip RTMT loglarından teşhis edeceğiz:
**yanlış arama tabanı**.

**1–3.** **LDAP Directory > LDAP Server**'ı sil → **Add New** ile **YANLIŞ**
tabanla oluştur:
- **LDAP User Search Base:** `cn=User, dc=cll-collab, dc=local` *(hatalı — doğrusu `cn=Users, …, dc=internal`)*
- Diğer alanlar aynı → **Save** → **Perform Full Sync Now**.

**4.** **End Users > Find:** tüm kullanıcılar **Inactive LDAP Synchronized
User** — sync başarısız, kullanıcılar pasifleşti.

**5–7. RTMT ile teşhis:** Trace & Log Central → `cucm-pub.cll-collab.internal`
→ **Refresh All** → **System > Cisco DirSync > log4j** → en yüksek numaralı
**dirsync_err** logunu aç (**Cisco Generic Viewer**) → arama kutusuna
`error` yaz → **Search**:
`LdapReferralException: [LDAP: error code 10 … remaining name
'cn=User, dc=cll-collab, dc=local'`.
Log, kullanılan **yanlış arama tabanını** açıkça gösterir.

**8. Düzelt:** **LDAP Directory > LDAP Server**'ı sil → doğru tabanla
yeniden oluştur: `cn=Users, dc=cll-collab, dc=internal` → **Save** →
**Perform Full Sync Now**. Kullanıcılar tekrar **Active** olur.

---

## Doğrulama (lab tamam)

- DirSync aktif; kullanıcılar AD'den replike (Active Enabled LDAP Synchronized).
- LDAP auth açık; Self-Care parolası AD'de doğrulanıyor.
- AD'de yapılan ad değişikliği sync ile CUCM'e taşınıyor.
- RTMT dirsync logundan yanlış arama tabanı teşhis edildi ve düzeltildi.

## Sık Karşılaşılan Sorunlar

| Belirti | Sebep / çözüm |
|---|---|
| Sync çalışmıyor | **Cisco DirSync** aktif mi? (Serviceability > Service Activation) |
| Kullanıcılar Inactive | Yanlış arama tabanı/DN; RTMT dirsync_err logunda `remaining name`'e bak. |
| Self-Care girişi başarısız | Auth açıksa parola **AD'de**; CUCM'deki eski yerel parola geçmez. |
| Kullanıcı alanı CUCM'de değişmiyor | LDAP açıkken değişiklik **AD'de** yapılır, sonra sync. |
| Attribute değiştirilemiyor | Önce tüm LDAP directory'leri sil, sonra LDAP System'i değiştir. |

## Eğitmen Notu

İki kavramı net ayırın: **sync** (kim var — AD'den okunur) ve **auth** (parola
doğru mu — AD'ye sorulur). jdoe örneği ikisini birden gösterir: yerel kullanıcı
sync ile LDAP'a döner, parola auth açılınca AD'ye geçer. Sorun gidermede
refleks: **RTMT > Trace & Log Central > DirSync log > `error` ara** →
`remaining name` yanlış arama tabanını ele verir. "Arama tabanı tek karakter
yanlışsa (Users→User, internal→local) herkes pasifleşir" mesajını verin.

---

### Sonraki

**Discovery 5: Cisco Self-Provisioning ile IP Telefon Dağıtımı** — otomatik
kayıt, evrensel şablonlar, self-provisioning IVR.
