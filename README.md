# Video Upload & Analysis App

Bulut tabanlı video işleme prototipi: video yükle, Azure Video Indexer analiz
etsin, sonuçları tarayıcıda gör.

**Demo videosu:** https://www.youtube.com/watch?v=oBoHbbkwjj0

---

## Türkçe

Kullanıcının bir `.mp4` dosyası yükleyip otomatik analiz aldığı bir Flask
uygulaması. Yüklenen video Azure Video Indexer'a gönderiliyor, servis
transkripsiyon / konu tespiti / anahtar kelime çıkarımı yapıyor; videonun meta
verisi ise MongoDB Atlas'ta saklanıyor. Ayrıca WebRTC ile tarayıcıdan canlı
kamera önizlemesi gösteren bir demo bölümü var.

Ankara Üniversitesi Bilgisayar Mühendisliği **Bulut Bilişim ve Uygulamaları**
dersi kapsamında geliştirildi.

### Ne yapıyor

- `.mp4` video yükleme, Azure Video Indexer'a aktarma
- Video meta verisini (dosya adı, Azure video ID, yükleme tarihi, durum)
  MongoDB Atlas'ta saklama
- Yüklenmiş videoları listeleme
- Bir videonun analiz sonucunu (tespit edilen konular ve anahtar kelimeler)
  çekip gösterme, aynı anda MongoDB'deki durumu `Analyzed` olarak güncelleme
- WebRTC (`getUserMedia`) ile canlı kamera önizlemesi — kayıt veya sunucuya
  aktarım yok, tamamen istemci tarafında

### Veri akışı

```
Tarayıcı
   |  1. .mp4 seç, POST /upload
   v
Flask (app.py)
   |  2. Azure'dan access token al (Ocp-Apim-Subscription-Key ile)
   |  3. Videoyu Video Indexer'a yükle (privacy=Private)
   |  4. Dönen video_id'yi MongoDB'ye yaz (status: Uploaded)
   v
Azure Video Indexer  -->  asenkron işleme (transkripsiyon, konu/nesne tanıma)
   ^
   |  5. GET /result/<video_id> ile Index endpoint'inden analizi çek
   |  6. keywords + topics ayıkla, MongoDB'de status: Analyzed
   v
Tarayıcı  -- analiz sonuçları ekranda
```

Analiz asenkron olduğu için uygulama Azure'u sürekli sorgulamıyor; kullanıcı
"Analizi Gör" dediğinde o anki güncel durum çekiliyor.

### Kullanılan teknolojiler

| Katman | Teknoloji |
|---|---|
| Backend | Python 3.9+, Flask |
| Video analizi | Azure Video Indexer API (`api.videoindexer.ai`) |
| Veritabanı | MongoDB Atlas (M0), `pymongo` |
| Frontend | HTML / CSS / JavaScript (`fetch`, `FormData`), WebRTC |
| Konfigürasyon | `python-dotenv` |
| HTTP istemcisi | `requests` |

### Uç noktalar

| Metot | Yol | Açıklama |
|---|---|---|
| `GET` | `/` | Ana sayfa: yükleme formu, video listesi, WebRTC demosu |
| `POST` | `/upload` | `video` alanıyla dosya alır, Azure'a yükler, MongoDB'ye kaydeder |
| `GET` | `/result/<video_id>` | Azure'dan analizi çeker, konu/anahtar kelime döner, durumu günceller |

### Kurulum

Gereksinimler: Python 3.9+, bir Azure Video Indexer hesabı (ücretsiz trial
yeterli) ve bir MongoDB Atlas cluster'ı (M0 ücretsiz katman yeterli).

```bash
git clone https://github.com/oguzordu/video-upload-analysis-app.git
cd video-upload-analysis-app
python -m venv .venv
source .venv/bin/activate    # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Ardından proje kökünde bir `.env` dosyası oluştur (`.env.example` dosyasını
kopyalayabilirsin):

```env
VIDEO_INDEXER_SUBSCRIPTION_KEY=azure_video_indexer_primary_key
VIDEO_INDEXER_ACCOUNT_ID=azure_video_indexer_account_id
VIDEO_INDEXER_LOCATION=trial
MONGODB_CONNECTION_STRING=mongodb+srv://kullanici:parola@cluster.mongodb.net/
MONGODB_DB_NAME=video_app_db
```

- **Azure değerleri:** Video Indexer portalı, *Account Settings* bölümündeki
  Primary Key ve Account ID. Trial hesapta `VIDEO_INDEXER_LOCATION` değeri
  `trial` olur.
- **MongoDB:** Atlas'ta *Database Access* bölümünden bir kullanıcı, *Network
  Access* bölümünden IP izni tanımla, *Connect* altındaki *Drivers* bağlantı
  dizesini al.

Çalıştır:

```bash
python app.py
```

Tarayıcıdan `http://127.0.0.1:5000/` adresini aç.

> WebRTC canlı kamera önizlemesi tarayıcıdan kamera izni ister. `getUserMedia`
> güvenli bağlam gerektirdiği için `localhost` üzerinde çalışır; başka bir
> makineden erişilecekse HTTPS gerekir.

### Proje yapısı

```
app.py             Flask uygulaması: rotalar, Azure ve MongoDB entegrasyonu,
                   render_template_string ile gömülü HTML/JS arayüz
requirements.txt   Python bağımlılıkları
.env.example       Gerekli ortam değişkenlerinin şablonu
```

### Bilinen sınırlar

Bu bir ders projesi prototipi:

- Kimlik doğrulama yok, tek kullanıcılı varsayılıyor
- Flask'ın geliştirme sunucusu kullanılıyor, üretim için WSGI sunucusu gerekir
- Analiz durumu otomatik takip edilmiyor; kullanıcı istediğinde çekiliyor
- Arayüz `render_template_string` ile `app.py` içine gömülü; proje büyürse ayrı
  `templates/` ve `static/` klasörlerine taşınmalı

---

## English

A Flask application that lets a user upload an `.mp4` file and get automated
analysis back. The uploaded video is sent to Azure Video Indexer, which handles
transcription, topic detection and keyword extraction; the video's metadata is
stored in MongoDB Atlas. A separate section demonstrates a live camera preview
in the browser via WebRTC.

Built for the **Cloud Computing and Applications** course at Ankara University,
Department of Computer Engineering.

### What it does

- Uploads an `.mp4` video and forwards it to Azure Video Indexer
- Stores video metadata (filename, Azure video ID, upload date, status) in
  MongoDB Atlas
- Lists previously uploaded videos
- Fetches a video's analysis (detected topics and keywords) on demand and
  updates its status to `Analyzed` in MongoDB
- Shows a live camera preview through WebRTC (`getUserMedia`) — nothing is
  recorded or sent to the server, it is entirely client-side

### Data flow

```
Browser
   |  1. pick .mp4, POST /upload
   v
Flask (app.py)
   |  2. get an access token from Azure (Ocp-Apim-Subscription-Key)
   |  3. upload the video to Video Indexer (privacy=Private)
   |  4. write the returned video_id to MongoDB (status: Uploaded)
   v
Azure Video Indexer  -->  asynchronous processing
   ^
   |  5. GET /result/<video_id> pulls the index from Azure
   |  6. extract keywords + topics, set status: Analyzed in MongoDB
   v
Browser  -- analysis rendered on the page
```

Because indexing is asynchronous, the app does not poll Azure — the current
state is fetched when the user asks for it.

### Tech stack

| Layer | Technology |
|---|---|
| Backend | Python 3.9+, Flask |
| Video analysis | Azure Video Indexer API (`api.videoindexer.ai`) |
| Database | MongoDB Atlas (M0), `pymongo` |
| Frontend | HTML / CSS / JavaScript (`fetch`, `FormData`), WebRTC |
| Configuration | `python-dotenv` |
| HTTP client | `requests` |

### Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | Home page: upload form, video list, WebRTC demo |
| `POST` | `/upload` | Takes a file in the `video` field, uploads to Azure, records it in MongoDB |
| `GET` | `/result/<video_id>` | Pulls the analysis from Azure, returns topics/keywords, updates status |

### Setup

Requirements: Python 3.9+, an Azure Video Indexer account (the free trial is
enough) and a MongoDB Atlas cluster (the free M0 tier is enough).

```bash
git clone https://github.com/oguzordu/video-upload-analysis-app.git
cd video-upload-analysis-app
python -m venv .venv
source .venv/bin/activate    # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Then create a `.env` file in the project root (you can copy `.env.example`):

```env
VIDEO_INDEXER_SUBSCRIPTION_KEY=your_azure_video_indexer_primary_key
VIDEO_INDEXER_ACCOUNT_ID=your_azure_video_indexer_account_id
VIDEO_INDEXER_LOCATION=trial
MONGODB_CONNECTION_STRING=mongodb+srv://user:password@cluster.mongodb.net/
MONGODB_DB_NAME=video_app_db
```

- **Azure values:** Video Indexer portal, *Account Settings* gives the Primary
  Key and Account ID. On a trial account `VIDEO_INDEXER_LOCATION` is `trial`.
- **MongoDB:** in Atlas, create a user under *Database Access*, allow your IP
  under *Network Access*, and copy the connection string from *Connect* then
  *Drivers*.

Run it:

```bash
python app.py
```

Open `http://127.0.0.1:5000/` in a browser.

> The WebRTC preview asks for camera permission. `getUserMedia` requires a
> secure context, so it works on `localhost`; serving it to another machine
> requires HTTPS.

### Project structure

```
app.py             Flask app: routes, Azure and MongoDB integration, and the
                   HTML/JS UI embedded via render_template_string
requirements.txt   Python dependencies
.env.example       Template for the required environment variables
```

### Known limitations

This is a course-project prototype:

- No authentication; a single user is assumed
- Runs on Flask's development server — a WSGI server is needed for production
- Indexing status is not tracked automatically, only fetched on demand
- The UI is embedded in `app.py` via `render_template_string`; it should move to
  proper `templates/` and `static/` directories if the project grows
