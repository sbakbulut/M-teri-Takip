# KURULUM.md — Drive senkronu (Müşteri Takip / Tiffany Lead Tracker v2.1)

Bu uygulama **kendi Apps Script projesini** kullanmalıdır. Para Takip ile aynı URL'i kullanırsanız iki uygulama **tek bir kaydı** paylaşır ve birbirinin verisini ezer.

## Kim ne yapabilir

| Görev | Kim |
|---|---|
| Google hesabına giriş, "İzin ver / Allow" tıklaması | **İnsan** (zorunlu) |
| Apps Script editöründe kod/property düzenleme, Deploy | Tarayıcı otomasyonu olan ajan **veya** insan |
| `curl` ile uçtan uca doğrulama (tarayıcı gerekmez) | **Ajan** |

**Yasaklar (ajan için):** gizli kelimeyi (token) repoya, commit'e, issue'ya, log dosyasına yazma; token'ı URL'de kullanma; token'ı ekrana basma.

## Adım 1 — Proje ve kod

1. https://script.google.com → **Yeni proje** (adı ör. `Musteri-Takip Drive Sync`).
2. Bu depodaki **`gas/Code.gs`** içeriğini editöre tamamen yapıştır (Ctrl+A → Ctrl+V) → **Ctrl+S**.

## Adım 2 — Gizli kelimeyi (token) kaydet

Gizli kelime: **en az 16 karakter**, tahmin edilemez. Örnek: `musteri-takip-2026-K9m2xQ7p`

1. Sol kenar → **⚙️ Proje ayarları**
2. Aşağı kaydır → **Betik özellikleri** (Script Properties)
3. **Betik özelliği ekle** → **Ad:** `token` · **Değer:** gizli kelime
4. **Kaydet**

> Aynı kelime Adım 4'te uygulamaya da yazılacak; iki taraf birebir aynı olmalı.

## Adım 3 — Yayınla

1. **Dağıt → Yeni dağıtım** → ⚙️ (tür seç) → **Web uygulaması**
2. *Şu kullanıcı olarak çalıştır:* **Ben** · *Erişimi olanlar:* **Herkes** → **Dağıt**
3. Çıkan **Web uygulaması URL'ini** kopyala: `https://script.google.com/macros/s/.../exec`

> Var olan bir dağıtımı güncelliyorsan **Dağıt → Dağıtımları yönet → ✏️ → Sürüm: Yeni sürüm → Dağıt** kullan; böylece URL değişmez.

## Adım 4 — Doğrula (curl; tarayıcı gerekmez)

```bash
URL='https://script.google.com/macros/s/BURAYA_ID/exec'
TOKEN='gizli-kelimen'

# 1) Güvenli protokol çalışıyor mu? -> {"ok":true,"rev":"3",...}
curl -sL -X POST "$URL" -H 'Content-Type: text/plain' \
  -d "{\"action\":\"ping\",\"token\":\"$TOKEN\"}"

# 2) Kayıt okuma -> {"ok":true,...,"data":"..."}   (kayıt yoksa data boş)
curl -sL -X POST "$URL" -H 'Content-Type: text/plain' \
  -d "{\"action\":\"get\",\"token\":\"$TOKEN\"}"

# 3) GÜVENLİK KONTROLÜ: URL'de token yolu kapalı olmalı
#    Beklenen: {"ok":false,"error":"method_not_allowed"}
curl -sL "$URL?token=$TOKEN"

# 4) Yanlış token reddedilmeli -> {"ok":false,"error":"unauthorized"}
curl -sL -X POST "$URL" -H 'Content-Type: text/plain' \
  -d '{"action":"ping","token":"yanlis-token-123456"}'
```

Beklenen: 1 ve 2 `ok:true`, 3 `method_not_allowed`, 4 `unauthorized`.

## Adım 5 — Uygulamaya yaz

1. Uygulama → **Yönetici → Drive Ayarları**
2. **URL** + **aynı gizli kelime** → **Test Et & Kaydet**
   - `✓ Güvenli protokol çalışıyor (script rev 3, kayıt …)`
   - `✓ URL'de token yolu kapalı (güvenli)`
3. Uzak boşsa yerel veri otomatik gönderilir (yerelden uzağa). Uzakta veri varsa ve çok küçükse **çakışma onayı** çıkar — *Uzağı uygula / Yereli gönder / Yok say*.

## Sorun giderme

| Belirti | Anlamı | Çözüm |
|---|---|---|
| `server_token_missing` | Script Property `token` yok | Adım 2 |
| `method_not_allowed` | Yayınlanan sürüm eski ya da istek GET | Adım 3 (Yeni sürüm → Dağıt) |
| `unauthorized` | İki taraftaki kelime farklı | Adım 2 + Adım 5'te aynı kelime |
| `rate_limited` | Dakikada 60 istek aşıldı | 1 dakika bekle |
| `too_large` | Veri 1 MB sınırını aştı | Kayıtları azalt / arşivle |
| Uygulama "Bu URL başka bir uygulamaya ait" diyor | URL Para Takip'e ait | Bu uygulamanın kendi URL'ini gir (Adım 3) |
| Testte `⚠ GÜVENLİK: …` | Script hâlâ URL'deki token'ı kabul ediyor | Adım 3 (yeni sürümü yayınla) |

## Protokol özeti

| İstek | Gövde | Yanıt |
|---|---|---|
| `POST <url>` | `{"action":"ping","token":"…"}` | `{ok:true, rev:"3", hasData}` |
| `POST <url>` | `{"action":"get","token":"…"}` | `{ok:true, rev, dataRev, data:"<json>"}` |
| `POST <url>` | `{"action":"put","token":"…","data":"<json>"}` | `{ok:true, rev, dataRev, bytes}` |
| `GET <url>` | — | `{ok:false, error:"method_not_allowed"}` |

- Token **yalnızca istek gövdesinde** taşınır (URL'de asla).
- Sunucu: min 16 karakter token, sabit zamanlı karşılaştırma, dakikada 60 istek, 1 MB gövde/veri sınırı.
