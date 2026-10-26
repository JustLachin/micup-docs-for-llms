# Micup Geliştirici Dokümantasyonu — LLM'ler ve AI Ajanları için

Micup'ın herkese açık **HTTP API**, **Gateway**, **webhook**, **ses/medya** ve Micup'a özel platform özelliklerini AI kodlama araçlarının doğru kullanması için hazırlanmış, tek dosyada yoğunlaştırılmış teknik referans.

> **Sürüm:** 1.0 · **Canlı doğrulama:** 26 Ekim 2026
> **Canlı dokümanlar:** <https://micup.gg/docs/> · **Keşif:** <https://micup.gg/.well-known/fluxer>

Bu depo, [jubbio-docs-for-llms](https://github.com/saintberat/jubbio-docs-for-llms) yaklaşımının Micup için uyarlanmış halidir: modellerin Micup yerine Discord API'si uydurmasını engellemek için doğrulanmış uç noktalar, kimlik doğrulama kuralları, Gateway opcode'ları, etkileşim akışı ve Micup'a özel özellikler tek bir referansta toplanır.

Teknik referans, alan adları, başlıklar ve hata kodlarının birebir korunması için İngilizce yazılmıştır. README ve kullanım notları Türkçedir. Bu depo resmî bir Micup yayını değildir; canlı dokümanların yerini almaz.

---

## İçindekiler

| Dosya | Ne işe yarar |
| --- | --- |
| [MICUP_DEV_REFERENCE.md](./MICUP_DEV_REFERENCE.md) | Ana teknik referans. Bot, OAuth2, HTTP API, Gateway, webhook, ses, medya ve Micup özellikleri. |
| [llms.txt](./llms.txt) | LLM'ler için kısa indeks: canlı doküman bağlantıları ve yerel dosyalar. |
| [llms-full.txt](./llms-full.txt) | README + ana referansın düz metin birleşimi. Claude Projeleri ve ChatGPT Knowledge için uygundur. |
| [LICENSE](./LICENSE) | AGPL-3.0-or-later lisans metni. |

---

## Neden bu depo?

Micup yeni bir platformdur ve genel amaçlı modellerin eğitim verisinde Micup API'sinin ayrıntıları yoktur. Bu yüzden modeller sık sık:

- Discord'un `discord.com/api` adresini veya `gateway.discord.gg` Gateway'ini kullanır,
- Micup'ta **bulunmayan** Gateway intent'lerini ve ayrıcalıklı intent onayını varsayar,
- Etkileşimleri HTTP endpoint'i ile almaya çalışır ve Ed25519 imza doğrulaması ekler,
- `applications.commands` kapsamını ister,
- Snowflake'leri JSON sayısı olarak ayrıştırır,
- Micup'a özel `MicupBOT`, feed, masa oyunları ve mağaza rotalarını botlarla çağırmaya çalışır.

Bu referans, bu hataların çoğunu kaynağında kapatır.

---

## Hızlı gerçekler

| Bileşen | Değer |
| --- | --- |
| Keşif | `GET https://micup.gg/.well-known/fluxer` |
| HTTP API | `endpoints.api_public` + `/v1` (canlı: `https://micup.gg/api/v1`) |
| Gateway | `endpoints.gateway` + `?v=1&encoding=json` (canlı: `wss://micup.gg/gateway`) |
| Bot kimlik doğrulama | `Authorization: Bot <token>` |
| Kullanıcı oturumu | `Authorization: <token>` (şema öneki yok) |
| OAuth2 | `Bearer <token>`, yalnızca açıkça desteklenen rotalarda |
| API sürümü | Yalnızca `/v1` |
| Gateway sürümü | Yalnızca `v=1` |
| Resmî Micup SDK'sı | Dokümante edilmemiş; discord.js 14 / discord.py 2 kullanın |
| OpenAPI | `GET https://micup.gg/api/v1/openapi.json` |
| Bot kataloğu | `GET https://micup.gg/api/v1/instance/bot-catalog` |
| Canlı dokümanlar | <https://micup.gg/docs/> |

**Önemli:** Bu değerler 26 Ekim 2026'daki canlı keşif yanıtından alınmıştır. Bir istemci her zaman `/.well-known/fluxer` okumalıdır; adresleri koda gömmeyin.

---

## AI araçlarıyla kullanım

### Cursor / Windsurf

1. Depoyu klonlayın veya indirin.
2. `MICUP_DEV_REFERENCE.md` dosyasını projenize ekleyin.
3. Cursor'da **Docs** veya proje kuralları bağlamına ekleyin.
4. Depoyu yayınladıktan sonra `llms.txt` dosyasının ham (raw) URL'sini uzak doküman indeksi olarak ekleyebilirsiniz.

### Claude Projeleri / ChatGPT Custom GPT

- `MICUP_DEV_REFERENCE.md` veya `llms-full.txt` dosyasını proje bilgi tabanına yükleyin.
- Uzun bağlam penceresi varsa `llms-full.txt`, daha kısa bağlamda `MICUP_DEV_REFERENCE.md` kullanın.

### Sistem promptu

Micup kodu üretirken şu kısa yönergeyi sistem promptuna ekleyebilirsiniz:

```text
Micup API'si için yalnızca şu kaynakları doğru kabul et:
1. https://micup.gg/.well-known/fluxer keşif yanıtı
2. https://micup.gg/docs/ canlı dokümanları
3. Bu projedeki MICUP_DEV_REFERENCE.md

Micup, Discord değildir. Gateway intent'leri yoktur; etkileşimler HTTP ile
alınmaz, Gateway üzerinden gelir; Ed25519 imzası ve PING yoktur;
applications.commands kapsamı yoktur. Tüm snowflake'ler string'dir ve
API tabanı keşiften okunur. Bilinmeyen bir uç nokta veya alan uydurma.
```

### Doğrudan API çağrısı

```js
const discovery = await (await fetch('https://micup.gg/.well-known/fluxer')).json();
const api = discovery.endpoints.api_public.replace(/\/$/, '');

const response = await fetch(`${api}/v1/channels/${channelId}/messages`, {
  method: 'POST',
  headers: {
    Authorization: `Bot ${process.env.MICUP_TOKEN}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({content: 'Merhaba, Micup!'}),
});

console.log(response.status, await response.text());
```

Tam örnekler ve hata yönetimi için [MICUP_DEV_REFERENCE.md](./MICUP_DEV_REFERENCE.md) §16'ya bakın.

---

## İlk botu oluşturma

1. Micup'ta **Ayarlar → Uygulamalar** bölümünü açın.
2. **Uygulama oluştur** ile bir ad verin. Uygulamayla birlikte bot hesabı da oluşturulur.
3. **Bot token** değerini hemen kopyalayın. Tam token yalnızca oluşturma veya yenileme yanıtında gösterilir.
4. Botu davet edin:

```text
https://micup.gg/oauth2/authorize?client_id=UYGULAMA_ID&scope=bot&permissions=3072
```

`3072`, `VIEW_CHANNEL` (1024) + `SEND_MESSAGES` (2048) toplamıdır.

5. Kanal kimliğini `https://micup.gg/channels/SUNUCU_ID/KANAL_ID` bağlantısından alın.
6. Mesaj gönderin:

```bash
curl -X POST "https://micup.gg/api/v1/channels/KANAL_ID/messages" \
  -H "Authorization: Bot $MICUP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content":"Merhaba, Micup!"}'
```

Slash komut, buton, menü ve modal için referansın **§8 Application commands and interactions** bölümüne, Gateway bağlantısı için **§9 Gateway** bölümüne bakın.

---

## Sık yapılan hatalar

| Hata | Doğrusu |
| --- | --- |
| `https://discord.com/api` kullanmak | `endpoints.api_public` + `/v1` |
| `wss://gateway.discord.gg` kullanmak | `endpoints.gateway` + `?v=1&encoding=json` |
| Gateway intent'leri göndermek | Micup'ta intent yoktur; `ignored_events` ve `Lazy Request` kullanın |
| Etkileşimler için HTTP sunucusu açmak | Etkileşimler Gateway'den `INTERACTION_CREATE` olarak gelir |
| Ed25519 imzası doğrulamak | Micup imza kullanmaz; etkileşim token'ı kimlik bilgisidir |
| `applications.commands` istemek | `scope=bot` yeterlidir |
| Snowflake'i `Number` ile ayrıştırmak | String olarak tutun, tam sayı olarak karşılaştırın |
| Bot token'ı URL'ye koymak | `Authorization: Bot <token>` kullanın |
| MicupBOT/feed/mağaza rotalarını botla çağırmak | Bu rotalar kullanıcı oturumu ister; bot `403 ACCESS_DENIED` alır |
| `403`/`401` yanıtını aynı istekle tekrar denemek | `code` alanını okuyun; kimlik bilgisini veya isteği düzeltin |
| `429` yanıtını beklemeden tekrarlamak | `retry_after` / `Retry-After` kadar bekleyin |

---

## Kapsam ve doğrulama

**Doğrulananlar (26 Ekim 2026):**

- `GET /.well-known/fluxer` → HTTP 200, canlı endpoint ve özellik değerleri.
- `GET /api/v1/instance/bot-catalog` → HTTP 200, resmî Micup bot kartları.
- `GET /api/v1/gateway/bot` (kimlik bilgisi olmadan) → HTTP 401.
- Canlı doküman ana sayfası, Gateway ve HTTP API sayfaları okundu.
- OpenAPI dokümanı → 402 path, 894 şema.
- Micup kaynak arşivindeki doküman kaynakları, izin kaydı, Gateway kaydı ve Micup'a özel rotalar.

**Doğrulanmayanlar (kimlik bilgisi/test hesabı gerektirir):**

- Gerçek bot ile giriş, mesaj gönderme ve etkileşim yanıtlama.
- Tam OAuth2 yetkilendirme/kod/token/refresh döngüsü.
- Ses yerleşimi ve LiveKit medyası.
- Masa oyunları, mağaza ödeme akışı, referral ve ilgi seçimi.
- Admin API davranışı.

Bu bölümler protokol dokümantasyonudur; uçtan uca canlı test raporu değildir.

---

## Depo yapısı ve bakım

```text
micup-docs-for-llms/
├── README.md
├── MICUP_DEV_REFERENCE.md
├── llms.txt
├── llms-full.txt
└── LICENSE
```

Bakım kuralları:

- Uç noktaları değiştirmeden önce `/.well-known/fluxer` yanıtını yeniden okuyun.
- `GET /v1/openapi.json` ile referansı karşılaştırın.
- `MICUP_DEV_REFERENCE.md` değiştiğinde `llms-full.txt` dosyasını yeniden üretin.
- Doğrulama tarihini ve canlı keşif anlık görüntüsünü güncelleyin.

`llms-full.txt` üretimi:

```bash
cat README.md MICUP_DEV_REFERENCE.md > llms-full.txt
```

---

## Lisans ve atıf

Bu depodaki dokümantasyon içeriği **AGPL-3.0-or-later** altında dağıtılır. Micup'ın dokümantasyonu ve kaynak kodu, Fluxer projesinden (<https://github.com/fluxerapp/fluxer>) türetilmiştir ve aynı lisansı taşır. Ayrıntılar için [LICENSE](./LICENSE).

"Micup" ve "Fluxer" adları ile logoları lisans kapsamında değildir; lisans marka hakkı vermez.

`X-Fluxer-*` başlıkları, `/.well-known/fluxer` yolu ve `flx_` token öneki gibi protokol tanımlayıcıları tel uyumluluğu için korunmuştur; Fluxer ile bir bağlantı iddiası değildir.

---

## English summary

This repository is an LLM-oriented technical reference for **Micup** (`micup.gg`), a Turkish chat and voice community platform built on the Fluxer protocol. It condenses Micup's public HTTP API, Gateway, OAuth2, webhooks, voice, media, and Micup-specific features into one verified document.

Micup is **Discord-shaped but not Discord**: there are no Gateway intents, no HTTP interaction receiver, no Ed25519 signatures, and no `applications.commands` scope. Always read base URLs from `https://micup.gg/.well-known/fluxer`; never hard-code Discord domains.

Files:

- [`MICUP_DEV_REFERENCE.md`](./MICUP_DEV_REFERENCE.md) — the complete technical reference.
- [`llms.txt`](./llms.txt) — compact index and live documentation links.
- [`llms-full.txt`](./llms-full.txt) — README plus the complete reference as plain text.
- [`LICENSE`](./LICENSE) — AGPL-3.0-or-later.

The reference was verified against live Micup discovery and documentation on **26 October 2026**. It is not an official Micup publication, and it does not replace the live documentation at <https://micup.gg/docs/>.
