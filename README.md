# YoutubeExplode for C++

[YoutubeExplode](https://github.com/Tyrrrz/YoutubeExplode) (C#) kütüphanesinin C++17 portu.
Kütüphane API anahtarı, OAuth ya da Google istemci kütüphaneleri kullanmıyor; YouTube'un kendi
web ve mobil uygulamalarının kullandığı dahili uç noktalarla konuşuyor.

- **Videolar**: meta veriler (başlık, kanal, tarih, süre, açıklama, anahtar kelimeler, izlenme ve beğeni sayısı)
- **Akışlar**: muxed, yalnız ses (opus/webm, AAC/m4a) ve yalnız video akışları. Bitrate, codec,
  container, boyut, örnekleme hızı ve kanal sayısı bilgileriyle birlikte **doğrudan çalınabilir URL**'ler.
  Kısıtlamayı (throttling) aşan, ileri-geri sarılabilen akış okuma ve dosyaya indirme de var.
- **İmza çözme**: `s` → `sig` dönüşümü için yerleşik minimal yorumlayıcı (reverse/splice/swap).
  İsteğe bağlı bir JS motoru takılırsa `n` parametresi ve modern oynatıcılar da çözülüyor.
- **Altyazılar**: parça listesi, zaman damgalı altyazılar, SRT'ye yazma
- **Oynatma listeleri**, **kanallar** (ID, @handle, /c/, /user/, yüklemeler)
- **Arama**: video, oynatma listesi ve kanal, sayfalama ile
- Çerezle oturum açma (yaş kısıtlı ve özel içerik için), iptal (`CancellationToken`), ilerleme
  bildirimi, `std::future` tabanlı async sarmalayıcılar
- Windows öncelikli (MSVC + WinHTTP, harici bağımlılık yok); Linux/macOS'ta libcurl

## Hızlı başlangıç

```cpp
#include <YoutubeExplode.hpp>
using namespace YoutubeExplode;

int main() {
    YoutubeClient youtube;

    // Arama
    for (const auto& v : youtube.search().getVideos("lofi hip hop", 10))
        std::cout << v.id() << " " << v.title() << " " << v.author().channelTitle() << "\n";

    // Video meta verisi (ID ya da herhangi bir URL biçimi olabilir)
    auto video = youtube.videos().get("https://youtu.be/dQw4w9WgXcQ");

    // Akışlar
    auto manifest = youtube.videos().streams().getManifest(video.id());
    auto audio = manifest.tryGetBestAudioOnlyStream();        // en yüksek bitrate'li yalnız-ses (genelde opus)
    auto m4a   = manifest.tryGetBestAudioOnlyStream("mp4");   // AAC/m4a tercih edilirse
    std::cout << audio->url() << "\n";                        // doğrudan çalınabilir

    // İndirme (ilerleme bildirimi ile)
    youtube.videos().streams().download(*audio, "song.webm", [](double p) { /* 0..1 */ });
}
```

C# API'siyle karşılaştırma:

| C# | C++ |
|---|---|
| `youtube.Videos.GetAsync(id)` | `youtube.videos().get(id)` / `getAsync(id)` |
| `youtube.Videos.Streams.GetManifestAsync(id)` | `youtube.videos().streams().getManifest(id)` |
| `manifest.GetAudioOnlyStreams().GetWithHighestBitrate()` | `getWithHighestBitrate(manifest.getAudioOnlyStreams())` |
| `youtube.Videos.Streams.GetAsync(info)` | `youtube.videos().streams().get(info)` → `MediaStream` (read/seek) |
| `youtube.Videos.Streams.DownloadAsync(info, path, progress)` | `download(info, path, progress, ct)` |
| `youtube.Videos.Streams.GetHttpLiveStreamUrlAsync(id)` | `getHttpLiveStreamUrl(id)` |
| `youtube.Videos.ClosedCaptions.GetManifestAsync(id)` | `youtube.videos().closedCaptions().getManifest(id)` |
| `youtube.Playlists.GetVideosAsync(id)` (`await foreach`) | `getVideos(id, max)` veya `getVideoBatches(id, handler)` |
| `youtube.Channels.GetByHandleAsync("@x")` | `youtube.channels().getByHandle("@x")` |
| `youtube.Channels.GetUploadsAsync(id)` | `youtube.channels().getUploads(id, max)` |
| `youtube.Search.GetResultsAsync(q)` | `youtube.search().getResults(q, max)` |
| `youtube.Search.GetVideosAsync(q)` | `youtube.search().getVideos(q, max)` |
| `CancellationToken` | `YoutubeExplode::CancellationToken` / `CancellationTokenSource` |

Kısayollar da var: `youtube.getVideo(id)`, `youtube.getStreamManifest(id)`, `youtube.searchVideos(q)`.
`Client`, `YoutubeClient`'in takma adı.

`IAsyncEnumerable` karşılığı olarak sayfalı sonuçlar geri çağırma (callback) ile döner:
`getVideoBatches(id, [](const Batch<PlaylistVideo>& b) { ...; return true; })`. `false` döndürmek,
C#'taki `break` gibi erken durur.

### Yapılandırma

```cpp
YoutubeClientOptions options;
options.initialCookies = { {"SAPISID", "..."}, {"__Secure-3PAPISID", "..."}, /* ... */ }; // oturum açmış hesap
options.jsEngine = std::make_shared<MyQuickJsEngine>();   // isteğe bağlı: n parametresi ve modern oynatıcılar
options.httpClient = Http::createDefaultHttpClient({ std::chrono::seconds(15), "http://proxy:8080" });
options.validateStreamUrls = true;   // C# ile aynı: her URL'nin son byte'ı yoklanır (ek istek maliyeti)
YoutubeClient youtube(options);
```

`YoutubeClient` kopyalaması ucuz ve thread-safe bir tutamaç. Kopyalar aynı oturumu (çerezler,
visitor data, önbelleğe alınmış oynatıcı) paylaşıyor.

## Derleme

Gereksinimler: CMake ≥ 3.16 ve C++17 derleyici (MSVC 2019+, GCC 9+, Clang 10+).

**Windows (MSVC):**
```bat
cmake -S . -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
build\example\Release\yte-example.exe dQw4w9WgXcQ
```
Çıktı `build\Release\YoutubeExplode.lib` ve `include\` altındaki başlık dosyaları. Tek sistem
bağımlılığı `winhttp.lib` (Windows ile birlikte gelir, CMake otomatik bağlar).

**Linux / macOS:** `libcurl` geliştirme paketi gerekir (`apt install libcurl4-openssl-dev`).
```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build -j
./build/tests/yte-tests
```

Kendi projene eklemek için:
```cmake
add_subdirectory(third_party/YoutubeExplode)   # veya FetchContent
target_link_libraries(myapp PRIVATE YoutubeExplode::YoutubeExplode)
```

CMake seçenekleri: `YTE_HTTP_BACKEND=WINHTTP|CURL` (Windows'ta curl de seçilebilir),
`YTE_BUILD_EXAMPLES`, `YTE_BUILD_TESTS`.

## Mimari

```
include/YoutubeExplode/          Public API (başlıklar; nlohmann/json veya platform tipleri sızdırmaz)
  YoutubeClient.hpp              Giriş noktası + seçenekler
  Videos/ Streams/ ClosedCaptions/ Playlists/ Channels/ Search/ Common/
  Http/HttpClient.hpp            IHttpClient: kendi HTTP katmanını takabilmek için
  JavaScript/IJsEngine.hpp       İsteğe bağlı JS motoru arayüzü
  Exceptions.hpp

src/
  ClientContext.*                YoutubeHttpHandler karşılığı: API anahtarı, hl, çerez kavanozu,
                                 SAPISIDHASH, 429 → RequestLimitExceeded, 5xx yeniden deneme,
                                 visitorData ve oynatıcı önbelleği
  Http/WinHttpClient.cpp         Windows'a özgü tek dosya (WinHTTP)
  Http/CurlHttpClient.cpp        Diğer platformlar (libcurl)
  Bridge/                        YouTube yanıtlarının ayrıştırıcıları (C# sürümündeki "Bridge" ile aynı rol)
    PlayerResponse.*             /youtubei/v1/player JSON'u → StreamData
    PlayerSource.*               base.js çözümlemesi: sts, dönüşüm planı, n fonksiyonu, tüm-oynatıcı çözücü
    Cipher/CipherManifest.*      reverse / splice / swap yorumlayıcısı
    Cipher/JsScanner.*           JS'i çalıştırmadan tarayan küçük yardımcılar (string/regex/template
                                 literal'lerini atlayarak parantez eşleme, ifade bölme)
    Pages.*, Manifests.*, BrowseResponses.*   İzleme sayfası, kanal sayfası, DASH, altyazı XML, arama, playlist
  Videos/VideoController.*       Innertube istemci profilleri (VISIONOS / ANDROID / TV)
  Videos/Streams/StreamClient.*  Manifest oluşturma, URL çözme, indirme
  Utils/                         URL, JSON yardımcıları, küçük XML ayrıştırıcı, SHA-1, base64, zaman
```

### Akış URL'leri nasıl elde ediliyor

C# YoutubeExplode ile aynı strateji uygulanıyor:

1. **VISIONOS** innertube istemcisi (`/youtubei/v1/player`). Şifresiz, doğrudan oynatılabilir URL
   döndürür ve çoğu video için çalışır.
2. Oynatılamazsa **ANDROID** istemcisi (örneğin çocuk içeriği).
3. Hâlâ oynatılamıyorsa (tipik olarak **yaş kısıtlaması**): oynatıcı JS'i indirilir, imza zaman
   damgası (`sts`) okunur ve **TVHTML5_SIMPLY_EMBEDDED_PLAYER** istemcisi çağrılır. Bu istemci yaş
   kapısını çoğunlukla aşar, ama URL'leri `signatureCipher` ile korunur. Bu URL'lerin imzası çözülür
   ve (JS motoru varsa) `n` parametresi dönüştürülür.
4. Varsa DASH manifestindeki akışlar da eklenir. URL'si olmayan (SABR) formatlar atlanır.

Her istekte `sw.js_data`'dan alınan visitorData ve AB onay çerezi (`SOCS`) gönderilir.

## İmza ve n parametresinin çıkarılması

`src/Bridge/PlayerSource.cpp` kütüphanenin **en kırılgan ve en önemli** parçası. Adımlar:

**1. Oynatıcının bulunması.** `https://www.youtube.com/iframe_api` küçük ve kararlı bir dosya.
İçindeki `.../s/player/<8 hex>/...` yolundan sürüm alınır ve
`/s/player/<sürüm>/player_ias.vflset/en_US/base.js` indirilir (~2-3 MB). Oynatıcı istemci ömrü
boyunca önbellekte tutulur.

**2. sts.** `signatureTimestamp:20668` (veya `sts:`) doğrudan metinde aranır. TV istemcisine
`playbackContext.contentPlaybackContext.signatureTimestamp` olarak gönderilir; böylece dönen
şifreler bu oynatıcı sürümüyle eşleşir.

**3. Global arama dizisi.** Yeni oynatıcılar `'use strict';var l="set{clone{...".split("{")` gibi
bir dizi tanımlar ve metod adlarına `l[12]` biçiminde başvurur. Bu dizi çözülür ve `a[l[3]]()`
gibi kod `a["reverse"]()` olarak okunabilir hale gelir.

**4. Dönüşüm planı (yerleşik yorumlayıcı, JS motoru gerektirmez).**
- *Çağrı noktası:* `.split(` geçen her yer için yakındaki kod küçük bir regex ile doğrulanır:
  `Xy=function(a){a=a.split("");Ab.cd(a,3);Ab.ef(a,41);return a.join("")}`.
  Dosyanın tamamında regex çalıştırılmaz: hem yavaş olur hem MSVC'nin `std::regex`'inde yığın taşmasına yol açabilir.
- *Yardımcı nesne:* `var Ab={cd:function(a,b){a.splice(0,b)},ef:function(a){a.reverse()},gh:function(a,b){var c=a[0];a[0]=a[b%a.length];a[b%a.length]=c}}`.
  Her üye gövdesine göre sınıflandırılır: `reverse` → **Reverse**, `splice` → **Splice(n)**,
  `%`/`[0]` → **Swap(n)**.
- Sonuç: `CipherManifest` (örn. `[Swap (64), Splice (1), Reverse]`). C++'ta doğrudan uygulanır.

**5. n fonksiyonu (JS motoru gerekir).** `.get("n"))&&(b=Xy[0](b)`,
`String.fromCharCode(110)`, `"nn"[+...]`, `d=Xy[0](c),a.set("n",d)` desenlerinden biriyle ya da
`..._w8_` hata dönüşünden fonksiyon adı bulunur. `Xy[0]` ise `var Xy=[Gercek]` çözülür. Gövde
parantez eşlenerek aynen kopyalanır, global diziyle birlikte bir betik haline getirilir.
`;if(typeof X==="undefined")return a;` erken dönüş koruması kaldırılır.

**6. Tüm-oynatıcı çözücü (JS motoru gerekir; modern oynatıcılar için tek güvenilir yol).**
2025'ten itibaren YouTube imza ve n kodunun akışını düzleştiriyor (örn. `P[l[h^315]](l[h^273])`).
Bu yapıdan hiçbir kod parçası güvenilir şekilde kesilip alınamıyor. yt-dlp'nin "EJS" çözücüsüyle
aynı yöntem, AST ayrıştırıcı olmadan metin düzeyinde uygulanıyor:
- `var _yt_player={};(function(g){var window=this; ... })(_yt_player);` IIFE gövdesi ifadelere bölünür,
- `var window=this` ve atama olmayan tüm üst düzey ifade deyimleri (tarayıcı gerektiren yan etkiler) atılır,
- gövdesinde `x.set("alr","yes")` deyimi bulunan URL yardımcısı bulunur,
- `_yte_solve(sig, n)` üretilir: bu yardımcıyla `s=<sig>` içeren bir URL nesnesi kurar, `n`'i atar,
  URL nesnesinin ilk prototip metodunu çağırır ve dönüştürülmüş `s` ile `n` değerlerini okur.

Bir manifestteki tüm imza ve n değerleri **tek bir** JS çalıştırmasında çözülür. Öncelik sırası:
tüm-oynatıcı çözücü (motor varsa) → yerleşik dönüşüm planı → kod parçacıkları (eski oynatıcılar).

### JS motoru takmak

```cpp
class QuickJsEngine : public YoutubeExplode::JavaScript::IJsEngine {
    std::string evaluate(const std::string& script) override { /* son ifadenin değerini string döndür */ }
};
```
Uygulamaya gömmek için önerilen motor **QuickJS-ng** (MIT, MSVC ile derlenir, ~1 MB).
`example/NodeJsEngine.hpp` harici `node` sürecini kullanan basit bir örnek; test ve deneme amaçlı.

JS motoru olmadan da kütüphane tam çalışır: birincil istemciler şifresiz URL döndürür. Motor
yalnızca TV yedeğinden gelen URL'lerde `n` dönüşümü ve yerleşik planın çıkarılamadığı modern
oynatıcılar için gerekir. Motor yoksa bu URL'ler orijinal `n` ile döner; çalınabilir, ama
yavaşlatılabilir ya da bazı oynatıcılarda reddedilebilir.

## Kırılgan kısımlar ve bakım rehberi

YouTube bu API'leri resmî olarak sunmuyor ve düzenli olarak değiştiriyor. Bozulma olasılığına göre:

| Ne | Nerede | Belirti | Ne yapmalı |
|---|---|---|---|
| **Innertube istemci profilleri** (ad, sürüm, cihaz, User-Agent) | `src/Videos/VideoController.cpp` | Her video için "unplayable" / "Sign in to confirm you're not a bot" | Güncel değerleri YoutubeExplode / yt-dlp'den alın (`clientVersion` vb.) |
| **Dönüşüm planı regex'leri** (çağrı noktası, yardımcı nesne, sınıflandırma) | `PlayerSource.cpp` → `extractSignatureCipher` | `CipherExtractionException` (motor yokken) | Yeni base.js'te `.split("")`/`join("")` fonksiyonunu inceleyip regex'i güncelleyin |
| **n fonksiyonu desenleri** | `PlayerSource.cpp` → `extractNFunction` | Yavaş indirme / 403 (yalnız eski yöntem) | Tüm-oynatıcı çözücü genelde yeterli; gerekirse desen ekleyin |
| **Tüm-oynatıcı çözücü** (IIFE biçimi, `"alr","yes"` imzası, prototip metodu) | `PlayerSource.cpp` → `buildPlayerSolver` | `solverScript` oluşmuyor; tanılama mesajı "URL helper ... not found" | yt-dlp/ejs'teki `nsig.ts` / `solvers.ts` değişikliklerini izleyin |
| **Oynatıcı sürüm tespiti** | `ClientContext.cpp` → `playerSource` | "Failed to extract the player version" | `/iframe_api` biçimini kontrol edin |
| **Arama / playlist JSON şekilleri** | `src/Bridge/BrowseResponses.cpp` | Boş sonuçlar | Soyundan-arama (`findDescendants`) dayanıklı, ama yeni "viewModel" adları eklenebilir |
| **visitorData** (`sw.js_data` içindeki konum) | `ClientContext.cpp` | Sessizce atlanır | `[0][2][0][0][13]` yolunu güncelleyin |
| **SOCS onay çerezi** | `ClientContext.cpp` | AB'de onay sayfasına yönlendirme | Değer yaklaşık 13 ayda bir yenilenmeli |
| **Innertube API anahtarı** | `ClientContext.cpp` | 400/403 | Web istemcisinden güncel anahtarı alın |

`CipherExtractionException` mesajı her zaman oynatıcı URL'sini ve hangi adımın başarısız olduğunu
içerir. Hata ayıklamaya en hızlı başlama yolu bu mesaj.

**Regresyon testleri:** gerçek oynatıcı dosyalarını bir klasöre `base.js-<id>` adıyla koyup
`YTE_TEST_PLAYERS_DIR=<klasör> YTE_TEST_NODE=1 ./build/tests/yte-tests` çalıştırın.
`tests/CipherTests.cpp` içindeki beklenen değerler üç bağımsız yöntemle çapraz doğrulandı:
yerleşik yorumlayıcı, Node.js'te kod parçası ve Node.js'te tüm oynatıcı. Kapsanan oynatıcılar:
2022-02, 2022-04 ve 2026-08 (`854a788e`). Yeni bir oynatıcı geldiğinde listeye ekleyin.

## Hata yönetimi

Tüm hatalar `Exceptions::YoutubeExplodeException`'dan türer:

| İstisna | Anlamı |
|---|---|
| `VideoUnavailableException` | Video yok, silinmiş ya da özel |
| `VideoAgeRestrictedException` | Yaş kapısı aşılamadı (oturum açmış hesap çerezi verin) |
| `VideoRequiresPurchaseException` | Ücretli içerik; `previewVideoId()` fragmanı verir |
| `VideoUnplayableException` | Diğer oynatılamama sebepleri (YouTube'un gerekçesi mesajda) |
| `CipherExtractionException` | Oynatıcıdan imza/n çıkarılamadı (YouTube oynatıcıyı değiştirdi) |
| `RequestLimitExceededException` | HTTP 429 |
| `PlaylistUnavailableException` | Oynatma listesi yok ya da özel |
| `HttpRequestException` | Ağ hatası ya da beklenmeyen HTTP durumu (`statusCode()`) |

İptal edilen işlemler `OperationCanceledException` fırlatır.

## Müzik uygulamasında kullanım notları

- URL'ler yaklaşık 6 saat geçerli ve isteği yapan IP'ye bağlı. URL'yi değil video ID'yi saklayın;
  çalmadan hemen önce `getManifest` çağırın.
- Kütüphane ses çözmüyor. Opus (webm, itag 251) için nestegg + libopus + miniaudio hafif bir
  kombinasyon; AAC (m4a, itag 140) Windows'ta Media Foundation ile ek bağımlılık olmadan çözülür.
- `MediaStream` ~10 MB'lık aralık istekleriyle okur; bu hem kısıtlamayı aşar hem ileri-geri sarmayı destekler.

## Bağımlılıklar ve lisanslar

| Bağımlılık | Nerede | Lisans | Not |
|---|---|---|---|
| [nlohmann/json](https://github.com/nlohmann/json) 3.11.3 | `third_party/nlohmann/` (gömülü, tek başlık dosyası) | MIT | Public API'ye sızmaz |
| WinHTTP | Windows sistem kütüphanesi | Windows'un parçası | Windows'ta varsayılan HTTP katmanı |
| [libcurl](https://curl.se/) | Windows dışı (veya `YTE_HTTP_BACKEND=CURL`) | curl lisansı (MIT/X türevi) | Sistem paketi |
| Node.js | *yalnızca* `example/` ve isteğe bağlı testler | MIT | Kütüphanenin bağımlılığı değil |

Başka bağımlılık yok. JS motoru isteğe bağlı ve kullanıcı tarafından sağlanıyor (öneri: QuickJS-ng, MIT).
`tests/fixtures/SyntheticPlayer.hpp` bu proje için elle yazıldı ve YouTube kodu içermez.

Referans alınan projeler (kod kopyalanmadı, mantık ve uç nokta biçimleri incelendi):
[YoutubeExplode](https://github.com/Tyrrrz/YoutubeExplode) (LGPL-3.0),
[youtube_explode_dart](https://github.com/Hexer10/youtube_explode_dart) (BSD-3),
[yt-dlp/ejs](https://github.com/yt-dlp/ejs) (Unlicense).

## Yasal not

YouTube Hizmet Şartları, içeriğe YouTube'un kendi oynatıcısı dışında erişmeyi ve indirmeyi
kısıtlıyor. Bu kütüphanenin kullanımından doğan sorumluluk kullanıcıya ait.
