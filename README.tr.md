# Hisar API Ağ Geçidi

[English](README.md) · [Web sitesi](https://hisar-gateway.com) · [Sürümler](https://github.com/hisar-gateway/hisar-demo/releases)

Elinizdeki API'ler ve hiç yazmadığınız API'ler için ağ geçidi. Tek bir Rust
ikilisi REST, SOAP, GraphQL, gRPC ve WebSocket servislerinin önünde durur, eski
SOAP'ı düz JSON'a çevirir, veritabanlarınızdan doğrudan CRUD API'leri üretir ve
LLM sağlayıcılarını ölçülen tek bir uç noktanın arkasına koyar. Kendi
makinelerinizde çalışır, çevrimdışı lisanslanır.

Bu repo **ücretsiz sürümdür**: aynı ağ geçidi, **5 rota** ve **rota başına
saniyede 10 istek** tavanıyla; lisanslı modüller kapalı. Hesap yok, süre sınırı
yok. Daha sonra yönetim konsolundan lisans yükleyin; aynı kurulum yeniden
kurmadan, yeniden başlatmadan tam kapasiteye geçer.

> Ağ geçidinin kaynak kodu burada yayınlanmaz. İmajlar
> `ghcr.io/hisar-gateway/apigateway-backend` ve `-frontend` paketlerinden gelir; bu
> repoda yalnızca Compose dosyası, `.env` ve bu kılavuz bulunur.

## Neler var

### Ücretsiz sürümde

- **Çok protokollü yönlendirme** — REST, SOAP, GraphQL, gRPC ve WebSocket tek
  hiyerarşinin arkasında: Kuruluş → Uygulama → API sürümü → Servis → Rota.
  Kullanımdan kaldırılan sürümler `Deprecation` / `Sunset` başlıklarıyla yanıt
  verir; geçmiş bir sunset tarihi kendiliğinden `410 Gone` döner.
- **SOAP ↔ JSON köprüsü** — istemci JSON gönderir; ağ geçidi SOAP zarfını kurar,
  action ve sürümü uygular, fault'ları yapılandırılmış JSON'a çevirir. WS-Trust /
  WS-Security jetonlarını ağ geçidi alır, önbelleğe koyar ve yeniler; çağıran
  hiç XML güvenlik başlığıyla uğraşmaz.
- **Çağıran kimlik doğrulaması** — dönen yenileme jetonlu ve yeniden kullanım
  tespitli RS256 JWT, kapsamlı API anahtarları, servis başına erişim modu
  (herkese açık, API anahtarı, JWT) ve zorunlu roller.
- **Girdi doğrulama, güvenlik başlıkları, IP izin/engel listeleri**,
  Elasticsearch'e istek ve denetim logları, OpenAPI içe aktarma.
- **Yönetim konsolu** Türkçe ve İngilizce; değişiklikler Redis Pub/Sub ile her
  replikaya anında ulaşır — yeniden başlatma yok, bekleme yok.

### Lisanslı modüller

Lisans, açtığı modülleri adıyla belirtir; on yedi modül şunlardır, hepsi aynı
kurulumda ve yeniden başlatmadan devreye girer.

| Modül | Ne ekler |
|---|---|
| **WAF** | 10 saldırı sınıfında (SQLi, XSS, dizin geçişi, komut, LDAP, NoSQL, SSRF, SSTI, XXE…) 57 yerleşik kurallı anomali puanlayan motor, kaçınmaya karşı üç katmanlı çözme, global / servis / rota / DB2API tablosu düzeyinde politika, Engelle ya da Yalnızca tespit |
| **DDoS koruması** | IP başına hız ve patlama tavanları, otomatik geçici yasak; bellek içi itibar anlık görüntüsü sayesinde uygulama Redis'e gitmez |
| **Coğrafi engelleme** | MaxMind ülke sorgusu, izin ve engel listeleri |
| **Hız sınırlama** | Yediye kadar kademe — kullanıcı, rol, rota, servis, API sürümü, uygulama, kuruluş — paylaşımlı, anahtar başına, kullanıcı başına ya da IP başına kovalar ve Redis'te kayan pencereler; API anahtarına bağlı abonelik planları, rota başına kota ve ayrılmış kapasite; mesai saati ve kampanya çarpanları |
| **Yük dengeleme** | Round-robin, ağırlıklı, en az bağlantı ya da rastgele; sağlık probu backend'i otomatik çıkarır ve geri alır |
| **Canary / blue-green** | Ağırlıkla ya da sabitleme başlığıyla yayın varyantları, varyant başına izlenir |
| **Dayanıklılık** | Kümede paylaşılan devre kesiciler, sunucu başına bulkhead, idempotent çağrılar için geri çekilmeli yeniden deneme |
| **Yanıt önbelleği** | Rota başına Redis önbelleği: TTL, sorgu parametresi anahtarı, başlığa göre varyasyon; kiracılar arası sızıntı olmasın diye yalnızca herkese açık rotalarda; AI yanıtları için tam ve anlamsal önbellek |
| **Karşılıklı TLS** | Upstream'e istemci sertifikası ve TLS sonlandıran proxy arkasındaki çağıranlar için istemci sertifikası politikaları |
| **CAPTCHA** | Tekrarlanan başarısız denemelerden sonra giriş formunda doğrulama; sağlayıcı ve eşikler çalışma zamanı ayarı |
| **DB2API** | PostgreSQL, MySQL, MariaDB, MSSQL, Oracle, MongoDB ya da SurrealDB'den üretilen CRUD uç noktaları — filtreleme, sıralama, toplama, bildirimsel ilişkiler, satır düzeyi güvenlik. Uygulama kodu yok |
| **AI ağ geçidi** | OpenAI, Azure OpenAI, Anthropic ve özel sağlayıcılar tek uç noktanın arkasında; çağıran başına ön ödemeli jeton cüzdanı, model başına günlük ücretsiz hak, prompt injection ve kişisel veri korkulukları, akış, yedek sağlayıcılar, maliyet metrikleri |
| **Veri maskeleme** | Yanıtlara ve loglara ayrı ayrı uygulanan beş strateji — tam, önek, sonek, kenarlar, kararlı hash |
| **Gelişmiş kimlik doğrulama** | OIDC, LDAP / Active Directory, SAML 2.0, güvenilir proxy ve harici REST girişleri; ilk girişte hesap açma ve dizinden gelen roller; TOTP iki faktör. Düz JWT / API anahtarı doğrulaması çekirdekte kalır |
| **Gelişmiş gözlemlenebilirlik** | OpenTelemetry izleri, Prometheus `/metrics`, hazır Grafana panoları |
| **Bildirimler** | Slack, Discord, webhook ve SMTP; önem derecesine göre uyarılar, lisans bitiş hatırlatmaları, günlük güvenlik raporu |
| **Geliştirici portalı** | Swagger UI, servis başına OpenAPI dışa aktarma, API kataloğu |

## Ücretsiz sürüm ve lisanslı

| | Ücretsiz sürüm | Lisansla |
|---|---|---|
| Rota | 5 | lisansa göre (0 = sınırsız) |
| Saniyede istek | rota başına 10 | lisanstaki gateway-geneli `max_rps` (0 = sınırsız) |
| Lisanslı modüller | kapalı | lisansınızdaki modüller |
| Süre sınırı | yok | lisans süresi, 60 gün önceden uyarı |
| Etkinleştirme | pakette | `.lic` dosyasını **Sistem → Lisans** ekranından yükleyin — anında geçerli |

Tavanı aşan istek `429 Lisans RPS limiti aşıldı (rota başına …)` alır; altıncı
rota `403 Route limiti aşıldı: 5/5` ile reddedilir.

## Gereksinimler

- Docker 24+ ve Compose v2 (Linux, macOS veya WSL2'li Windows)
- x86-64 host (imajlar `linux/amd64`; Apple Silicon'da öykünmeyle çalışır, daha yavaştır)
- 8080 (yönetim konsolu) ve 3000 (ağ geçidi) portları boş
- Tüm yığın için yaklaşık 2 GB RAM; Elasticsearch olmadan 1 GB daha az

## Başlatma

Dosyaları iki yoldan biriyle alın:

```bash
git clone https://github.com/hisar-gateway/hisar-demo.git && cd hisar-demo
# ya da: Releases sayfasından hisar-demo.zip indirip açın
```

Sonra:

```bash
docker compose up -d
docker compose logs apigateway | grep "INITIAL ADMIN CREATED"
```

Kayıt defterine giriş gerekmez — imajlar herkese açıktır. İlk başlatma yaklaşık
700 MB indirir.

http://localhost:8080 adresini açın, `demo@hisar-gateway.com` ve log satırındaki
parolayla giriş yapın; parolayı hesap menüsü → **Profil** ekranından değiştirin.

| Bileşen         | Adres                        |
|-----------------|------------------------------|
| Yönetim konsolu | http://localhost:8080        |
| Ağ geçidi       | http://localhost:3000        |
| Sağlık          | http://localhost:3000/health |

## İlk rota

1. **Kuruluşlar → Yeni** — hiyerarşinin tepesi (ör. `Demo`); her uygulama bir kuruluşa bağlıdır.
2. **Uygulamalar → Yeni** — o kuruluşu seçin; uygulama servisleri gruplar (slug ör. `demo`).
3. **Servisler → Yeni** — bir upstream gösterin, ör. `https://httpbin.org` (REST)
   ya da bir SOAP uç noktası; protokolü ve upstream'in istediği kimlik yöntemini seçin.
4. **Rotalar → Yeni** — slug `get`, yol `/get`, backend yolu `/get`, metot GET.
5. Ağ geçidi üzerinden çağırın: `curl http://localhost:3000/demo/<servis-slug>/get`.

Ya da **Servisler → İçe aktar** ile OpenAPI dosyası yükleyin; 5 rota tavanı tüm
içe aktarmaya uygulanır.

## Durdurma, sıfırlama, yükseltme

```bash
docker compose down              # durdur, veriyi koru
docker compose down -v           # durdur ve tüm veriyi sil
docker compose pull && docker compose up -d   # yeni imaja geç
```

`main` dalı en yeni sürümü izler (`HISAR_VERSION=latest`); Releases sayfasındaki
her kaydın zip'inde `.env` o sürüme sabitlidir. Kayıt defteri yalnızca yakın
sürümleri tutar; eski bir sabitleme zamanla çekilemez olur — belirli bir sürüm
gerekmiyorsa `latest`'te kalın.

## Notlar

- Swagger UI (`/swagger-ui`) ve `/metrics` lisanslı modüllere aittir (Geliştirici
  Portalı, Gelişmiş Gözlemlenebilirlik); ücretsiz sürümde 404 döner.
- Elasticsearch, panodaki log ve güvenlik panellerini besler. Bellek için
  `docker-compose.yml` içinden servisi silebilirsiniz; ağ geçidi onsuz da çalışır.
- JWT anahtarları süreç başına üretilir: `apigateway` konteyneri yeniden başlayınca
  herkesin oturumu kapanır. Oturumları korumak için PEM çifti bağlayın (bkz. `.env`).
- Bu düzen değerlendirme içindir. Üretim, ayrı kontrol/veri giriş noktaları ve
  yuvarlanan replikalarla Traefik arkasında çalışır — kullanım kılavuzu anlatır.

## Lisans

Lisanslar kuruluş başınadır, çevrimdışı imzalanır (Ed25519) ve yönetim
konsolundan yüklenir; lisans adını verdiği modülleri ve kapasiteyi açar, bitişten
60 gün önce uyarır. Fiyat ve lisans dosyası için
[hisar-gateway.com](https://hisar-gateway.com).
