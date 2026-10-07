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

**Kurulum: [Linux](#linux) · [macOS](#macos) · [Windows](#windows)** — birkaç
dakika, çoğu ilk indirmeyle geçer.

> Ağ geçidinin kaynak kodu burada yayınlanmaz. İmajlar
> `ghcr.io/hisar-gateway/apigateway-backend` ve `-frontend` paketlerinden gelir;
> bu repoda yalnızca Compose dosyası, `.env` ve bu kılavuz bulunur.

## Kurulum

Docker Compose altında beş container çalışır: ağ geçidi, yönetim konsolu,
SurrealDB (yapılandırma), Redis (sayaçlar, oturumlar, değişiklik olayları) ve
Elasticsearch (istek, denetim ve güvenlik logları).

Gerekenler:

- **`docker compose` komutuyla Docker** — Linux'ta Docker Engine 24 ya da
  üstü ve Compose eklentisi, macOS ve Windows'ta Docker Desktop.
- **x86-64 bir makine ya da Apple silicon'lı bir Mac.** İmajlar şimdilik
  yalnızca `linux/amd64` için üretiliyor; Apple silicon'da Docker Desktop
  onları Rosetta ile çalıştırır. ARM Linux (Raspberry Pi, AWS Graviton) ve Arm
  tabanlı Windows henüz desteklenmiyor.
- **Yaklaşık 2 GB boş bellek ve 4 GB disk.** İlk başlatma yaklaşık 1 GB
  indirir.
- **8080 ve 3000 portları** boş — ya da [seçtiğiniz başka portlar](#portlar).

### Linux

1. Docker Engine'i ve Compose eklentisini kurun —
   [dağıtımınıza göre talimatlar](https://docs.docker.com/engine/install/) ya da
   Docker'ın kolaylık betiği:

   ```bash
   curl -fsSL https://get.docker.com | sudo sh
   sudo usermod -aG docker "$USER"     # sudo'suz docker: oturumu kapatıp yeniden açın
   ```

   `docker compose version` v2 ya da üstünü göstermeli.

2. Paketi indirin, başlatın ve admin parolasını okuyun:

   ```bash
   curl -LO https://github.com/hisar-gateway/hisar-demo/releases/latest/download/hisar-demo.zip
   unzip hisar-demo.zip && cd hisar-demo
   docker compose up -d
   docker compose logs apigateway | grep "INITIAL ADMIN"
   ```

   `unzip` yok mu? `python3 -m zipfile -e hisar-demo.zip .` aynı işi görür.

Başkalarının erişebildiği bir sunucuya mı kuruyorsunuz? İlk başlatmadan önce
[Sunucuda çalıştırma](#sunucuda-çalıştırma) bölümünü okuyun.

### macOS

1. [Docker Desktop](https://docs.docker.com/desktop/setup/install/mac-install/)'ı
   kurun — Mac'inize uyan Apple silicon ya da Intel sürümünü — açın ve motorun
   çalıştığını bildirmesini bekleyin. Apple silicon'da **Settings → General →
   Use Rosetta for x86_64/amd64 emulation on Apple Silicon** seçeneğini açık
   bırakın.

2. Terminal'de:

   ```bash
   curl -LO https://github.com/hisar-gateway/hisar-demo/releases/latest/download/hisar-demo.zip
   unzip hisar-demo.zip && cd hisar-demo
   docker compose up -d
   docker compose logs apigateway | grep "INITIAL ADMIN"
   ```

### Windows

1. [Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/)'ı
   **WSL 2** altyapısıyla kurun — kurulumun varsayılanıdır; Windows 10 22H2 ya
   da Windows 11 ve açık sanallaştırma ister — sonra başlatın ve motorun
   çalıştığını bildirmesini bekleyin.

2. PowerShell'de:

   ```powershell
   Invoke-WebRequest https://github.com/hisar-gateway/hisar-demo/releases/latest/download/hisar-demo.zip -OutFile hisar-demo.zip -UseBasicParsing
   Expand-Archive hisar-demo.zip -DestinationPath .
   cd hisar-demo
   docker compose up -d
   docker compose logs apigateway | Select-String "INITIAL ADMIN"
   ```

   Ubuntu gibi bir WSL dağıtımının içinde — Docker Desktop'ın WSL entegrasyonu
   o dağıtım için açıkken — Linux komutları olduğu gibi çalışır.

git'i mi tercih ediyorsunuz? `git clone https://github.com/hisar-gateway/hisar-demo.git`
aynı dosyaları getirir; `main` dalı en yeni sürümü izler.

## Giriş

İlk başlatma birkaç dakika sürer, çoğu indirmedir. Hazır olunca
`docker compose ps` `apigateway`'i `healthy` gösterir ve parola komutu tek
satır yazar:

```text
… 🔐 INITIAL ADMIN CREATED — email: demo@hisar-gateway.com  password: …
```

**http://localhost:8080** adresini açın, bu e-posta ve parolayla giriş yapın,
sonra hesap menüsü → **Profil** ekranından kendi parolanızı belirleyin.

Parola yalnızca ilk başlatmada, bir kez yazılır ve yalnızca logda durur —
`.env`'i değiştirmek container'ı yeniden oluşturur ve logu boşaltır. Giriş
yapmadan kaybettiniz mi? `docker compose down -v` her şeyi siler, ardından
`docker compose up -d` yeni bir parola yazar.

| | Adres |
|---|---|
| Yönetim konsolu | http://localhost:8080 |
| Ağ geçidi — API trafiğiniz | http://localhost:3000 |
| Sağlık kontrolü | http://localhost:3000/health |

### Portlar

8080 ya da 3000'de başka bir şey mi çalışıyor? `.env` içinde başka host
portları seçip `docker compose up -d`'yi yeniden çalıştırın; yukarıdaki
adresler de buna göre değişir:

```bash
HISAR_UI_PORT=18080        # yönetim konsolu → http://localhost:18080
HISAR_GATEWAY_PORT=13000   # ağ geçidi       → http://localhost:13000
```

`.env`'i düz metin düzenleyiciyle açın: Linux ve macOS'ta `nano .env`,
Windows'ta `notepad .env`. Finder, adı noktayla başlayan dosyaları göstermez.

### Sunucuda çalıştırma

İki port da düz HTTP konuşur ve varsayılan olarak makinenin bağlı olduğu her
ağdan bağlantı kabul eder — Docker onları `ufw` gibi host güvenlik
duvarlarının önünden geçirerek yayınlar. İnternetten erişilebilen bir makinede
onları yalnızca loopback arayüzünde yayınlayın:

```bash
HISAR_UI_PORT=127.0.0.1:8080
HISAR_GATEWAY_PORT=127.0.0.1:3000
```

ve konsola kendi bilgisayarınızdan SSH tüneliyle ulaşın:

```bash
ssh -L 8080:127.0.0.1:8080 siz@sunucunuz    # sonra http://localhost:8080 adresini açın
```

## İlk rota

Her rota `http://localhost:3000/{uygulama}/{servis}/{rota}` adresinde,
her birine verdiğiniz slug'lardan oluşan yolda yayınlanır. Konsolda:

1. **Katalog → Kuruluşlar → Yeni kuruluş** — ad `Demo`, slug `demo`.
2. **Katalog → Uygulamalar → Yeni uygulama** — kuruluş `Demo`, ad `Demo`,
   slug `demo`.
3. **Katalog → Servisler → Yeni servis** — uygulama `Demo`, ad `Placeholder`,
   slug `placeholder`, servis türü REST, temel URL
   `https://jsonplaceholder.typicode.com`, kimlik doğrulama türü None.
4. **Proxy → Rotalar → Yeni rota** — servis `Placeholder`, ad `Users`, slug
   `users`, yöntem GET, yol `/users`, arka uç yolu `/users`.
5. Ağ geçidi üzerinden çağırın, tarayıcıda ya da curl ile:

   ```bash
   curl http://localhost:3000/demo/placeholder/users
   ```

   Windows PowerShell'de `curl.exe` yazın — oradaki düz `curl` başka bir
   komuttur.

Yol parametreleri rotanın yolundan gelir: slug'ı `user`, yolu `/users/:id`,
arka uç yolu `/users/:id` olan ikinci bir rota `/demo/placeholder/user/1`
isteğine tek bir kullanıcıyla yanıt verir.

Bir OpenAPI ya da Swagger dosyası, servisi bütün rotalarıyla tek seferde
oluşturur: **Katalog → Servisler → OpenAPI İçe Aktar**. 5 rota tavanı içe
aktarmanın tamamını sayar.

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
- **Yönetim konsolu** Türkçe ve İngilizce; değişiklikler Redis Pub/Sub ile her ağ
  geçidi örneğine anında ulaşır — yeniden başlatma yok, bekleme yok.

### Lisanslı modüller

Lisans, açtığı modülleri adıyla belirtir; on sekiz modül şunlardır, hepsi aynı
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
| **Kimlik sağlayıcı** | Size ait bir kullanıcı dizini — kullanıcılar, gruplar, iki faktörlü giriş — uygulamalarınıza OpenID Connect (PKCE'li yetkilendirme kodu, onay, dönen yenileme jetonları, oturum kapatma) ve yalnızca LDAP konuşan uygulamalar için LDAPS üzerinden sunulur; her uygulama listelediğiniz grupları kabul eder |
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
| Etkinleştirme | pakette | lisans dosyasını **Operasyonlar → Lisans** ekranından yükleyin — anında geçerli |

Tavanı aşan istek `429 License RPS limit exceeded (… req/s per route — restricted mode)` alır; altıncı
rota `403 Route limit exceeded: 5/5` ile reddedilir.

## Günlük komutlar

`hisar-demo` klasöründe çalıştırın.

| Komut | Ne yapar |
|---|---|
| `docker compose ps` | Her container'ın durumunu gösterir |
| `docker compose logs -f apigateway` | Ağ geçidi logunu izler (Ctrl+C izlemeyi bırakır) |
| `docker compose stop` | Her şeyi durdurur, veriyi korur |
| `docker compose start` | Yeniden başlatır |
| `docker compose down -v` | Container'ları **ve tüm veriyi** siler |

**Yükseltme.** `.env` içindeki `HISAR_VERSION`'ı yeni sürüme — ya da
`latest`'e — ayarlayın, sonra `docker compose pull` ve `docker compose up -d`
çalıştırın; veri korunur. Her [sürüm kaydındaki](https://github.com/hisar-gateway/hisar-demo/releases)
zip kendi sürümüne sabitlidir. Kayıt defteri yalnızca yakın sürümleri tutar;
eski bir sürüm zamanla indirilemez olur.

**Kaldırma.** `docker compose down -v --rmi all`, sonra klasörü silin.

## Sorun giderme

| Belirti | Ne yapmalı |
|---|---|
| `port is already allocated` ya da `address already in use` | 8080 ya da 3000'i başka bir program kullanıyor — [başka portlar seçin](#portlar) |
| `Cannot connect to the Docker daemon` ya da `error during connect` | Docker çalışmıyor: Docker Desktop'ı başlatın ya da Linux'ta `sudo systemctl start docker` |
| `exec format error` | ARM bir makine — imajlar şimdilik yalnızca amd64 |
| `INITIAL ADMIN` satırı yok | Ağ geçidi hâlâ açılıyor — bir dakika bekleyip komutu yeniden çalıştırın. Veride önceki bir başlatmadan kalan bir kullanıcı varsa [Giriş](#giriş) bölümüne bakın |
| `apigateway` `starting`'de kalıyor ya da `unhealthy` oluyor | Nedenini `docker compose logs apigateway` söyler. Önce SurrealDB, Redis ve Elasticsearch'ü bekler; Apple silicon'da ilk başlatma daha uzun sürer |
| `elasticsearch` 137 koduyla çıkıyor | Belleği yetmedi — Docker'a daha fazla bellek verin (macOS'ta: Docker Desktop → Settings → Resources) |
| Elasticsearch logu: `max virtual memory areas vm.max_map_count [65530] is too low` | Linux: `sudo sysctl -w vm.max_map_count=262144`; kalıcı olması için `/etc/sysctl.conf` dosyasına `vm.max_map_count=262144` ekleyin |
| Yeniden başlatmadan sonra oturum kapandı | Ağ geçidi container'ını yeniden başlatmak bütün oturumları bitirir — yeniden giriş yapın |

## Notlar

- Swagger UI (`/swagger-ui`) ve `/metrics` lisanslı modüllere aittir (Geliştirici
  Portalı, Gelişmiş Gözlemlenebilirlik); ücretsiz sürümde 404 döner.
- Bu bir değerlendirme düzenidir: tek ağ geçidi, düz HTTP ve başlangıçta
  üretilen bir imza anahtarı. Üretim kurulumları birden çok ağ geçidini bir TLS
  kenarının arkasında, yönetim konsolu ile API trafiği ayrı giriş noktalarında
  çalıştırır.

## Lisans

Lisanslar kuruluş başınadır, çevrimdışı imzalanır (Ed25519) ve yönetim
konsolunda **Operasyonlar → Lisans** ekranından yüklenir; lisans adını verdiği
modülleri ve kapasiteyi açar, bitişten 60 gün önce uyarır. Fiyat ve lisans için
[hisar-gateway.com](https://hisar-gateway.com).
