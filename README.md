# Tiffany Lead Tracker (Müşteri Takip)

Tiffany & Co. potansiyel müşteri (lead) takip uygulaması. **Tek dosya** (`index.html`), kurulum gerekmez, veriler tarayıcıda (`localStorage`) durur.

- **Canlı:** https://sbakbulut.github.io/M-teri-Takip/
- **Sürüm:** `v2.1`

## v2.1 — Drive senkron düzeltmesi + DeepSeek AI

| # | Değişiklik | Ayrıntı |
|---|---|---|
| 1 | **Drive senkron çalışmıyordu → düzeltildi** | Uygulama **eski protokolü** kullanıyordu: `GET /exec?token=…&key=…` ve `POST /exec?token=…`. Güvenli script `doGet`'i her zaman `method_not_allowed` ile reddediyor ve token'ı **yalnızca istek gövdesinden** okuyordu → senkron tamamen başarısız oluyordu (`method_not_allowed` + `unauthorized`). Artık istemci Para Takip v11.1 ile aynı protokolü kullanır: **sadece POST**, token **gövdede** → `{action:"ping"\|"get"\|"put", token, data}`. |
| 2 | **Ayrı Apps Script zorunlu** | Yeni sunucu veriyi tek kayıtta tutar (`key` ayrımı yoktur). Para Takip'in script'i paylaşılırsa **iki uygulama birbirinin verisini ezer**. Bu uygulama için ayrı proje + ayrı gizli kelime kullanılır. |
| 3 | **Veri korumaları eklendi** | Uzak veri uygulanmadan önce **doğrulanır** (tip / zorunlu alan / boyut sınırları, beyaz liste), **anlık görüntü** alınır (**↩ Son senkron öncesine dön**), uzak veri boş/çok küçükse **çakışma onayı** sorulur (*Uzağı uygula / Yereli gönder / Yok say*), 1 MB veri + 1.2 MB yanıt sınırı; sunucuda dakikada 60 istek limiti. Uzak veri uygulanırken geri push bastırılır (döngü olmaz). |
| 4 | **MiniMax → DeepSeek V4.1 Flash** | Sohbet/analiz artık `https://api.deepseek.com/chat/completions`, model `deepseek-flash`. Anahtar `pk_ai_key`'de saklanır (Para Takip ile aynı anahtar paylaşılır). 401/402/403/429 için Türkçe hata mesajları. |
| 5 | **TTS tarayıcıya taşındı** | MiniMax TTS kaldırıldı; sesli okuma tarayıcının `speechSynthesis` motoruyla yapılır. Ses seçici artık cihazdaki sesleri listeler (Türkçe varsa otomatik seçilir, hız ayarı korunur). |
| 6 | **📡 Test düğmesi** | Kurulumu doğrular: güvenli protokolün çalıştığını ve **URL'de token yolunun kapalı** olduğunu ölçer. |

### Kurulum (bir kez)

1. Apps Script'te **yeni proje** aç → bu depodaki **`gas/Code.gs`** içeriğini yapıştır (iki uygulama aynı script kodunu kullanır, ama **ayrı projeler** olmalıdır — ayrıntı: [`gas/KURULUM.md`](gas/KURULUM.md)).
2. **Proje Ayarları → Komut Dosyası Özellikleri → Özellik ekle** → ad: `token`, değer: **en az 16 karakter** gizli kelime.
3. **Dağıt → Yeni dağıtım → Web uygulaması** · *Şu kullanıcı olarak çalıştır:* **Ben** · *Erişimi olanlar:* **Herkes** → `/exec` URL'ini kopyala.
4. Uygulamada **Yönetici → Drive Ayarları** → URL + aynı gizli kelime → **Test Et & Kaydet** → `✓ Güvenli protokol çalışıyor` + `✓ URL'de token yolu kapalı`.
5. **🔑 DeepSeek API Key** → `sk-…` anahtarını gir.

### Doğrulama (yayındaki uçta)

| Test | Beklenen | Sonuç |
|---|---|---|
| `POST {action:"ping"}` | `ok:true` | ✅ `{"ok":true,"rev":"3","hasData":false}` |
| `POST {action:"get"}` | `ok:true` | ✅ `{"ok":true,"rev":"3","dataRev":"","data":""}` |
| `GET <url>?token=…` | `method_not_allowed` | ✅ |
| `POST` yanlış token | `unauthorized` | ✅ |

### Notlar / bilinen sınırlar

- Tek `index.html`; React/htm CDN'den gelir — internet yoksa uygulama açılmaz.
- Veriler tarayıcıda `localStorage`'da tutulur (`tiffany-clean-ui-v2`, `…-todos`); Drive senkron yedek ve çoklu cihaz içindir.
- Drive ayarları `tiffany_drive_url` / `tiffany_drive_token`; anlık görüntü `tiffany_drive_snapshot` anahtarındadır.
- Token **hiçbir zaman URL'de taşınmaz** (ne istemcide ne sunucuda kabul edilir).

## Depo yapısı

| Dosya | Açıklama |
|---|---|
| `index.html` | Uygulamanın tamamı (tek dosya) |
| `gas/Code.gs` | Apps Script sunucusu (Drive senkron, güvenli protokol rev 3) |
| `gas/KURULUM.md` | Adım adım kurulum + 4 curl doğrulama testi |
| `publish.sh` | Doğrula (JS sözdizimi + token URL kontrolü) → commit → push |
| `README.md` | Bu dosya (sürüm notları) |

## Geliştirme / yayın

```bash
python3 -m http.server 8080     # yerel önizleme → http://localhost:8080
./publish.sh -n "deneme"        # doğrula + commit et (push yok)
./publish.sh "commit mesajı"    # doğrula + commit + push
```
