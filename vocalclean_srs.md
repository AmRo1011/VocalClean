## SRS — VocalClean (MVP)

### 1) الهدف والنطاق
- الفصل المطلوب: **Acapella** (صوت المطرب فقط)
- المنصات أولًا: **Flutter Android** (قابل للتوسّع لاحقًا)
- أوفلاين: **ملف محلي** + أونلاين: **رابط YouTube** (مع شرط الحقوق)
- المعاينة: **10–15 ثانية من البداية**
- حدّ مدّة الملف: **5 دقائق** (MVP)
- أولوية الجودة: **أفضل جودة حتى لو أبطأ**

### 2) الجمهور وحالات الاستخدام
- الاستخدامات: **ريلز/تيك توك** + **بودكاست/Voice-over** + **المسلمون عمومًا (إزالة الموسيقى)**
- مستوى المستخدم: **مبتدئ**
- التسويق/الدفع والإعلانات:
  - **إعلانات** افتراضيًا
  - إزالة العلامة المائية + بدون إعلانات = **Premium مدى الحياة**
  - **MVP:** بدون تسجيل دخول، الترخيص محفوظ محليًا

### 3) الإدخال والحقوق
- المصادر: **ملف محلي** + **YouTube via yt-dlp** بشرط الحقوق
- التوافق القانوني: شاشة موافقة + رفض DRM
- الصيغ المقبولة: **MP3 (Audio)**، **MP4 (Video)**
- حدود: **Audio ≤ 100MB**، **Video ≤ 250MB**
- DRM: **كشف مبكّر ورفض** برسالة ودّية

### 4) مواصفات الإخراج والجودة
- النماذج: **Demucs (افتراضي)** + **Spleeter** (قابل للتبديل)
- Sample Rate: **44.1kHz** افتراضي + خيار **48kHz**
- Bit Depth: **16-bit** افتراضي + خيار **24-bit** (WAV)
- القناة: **Stereo**
- التنسيقات: **WAV**, **MP3 (320kbps)**, **MP4** (مع إعادة دمج الصوت)
- Normalization/Limiter: **اختياري**

### 5) الأداء والبنية التشغيلية
- تشغيل: **CPU** (Dev/Fallback) + **GPU تلقائيًا إذا متاح**
- نشر: محلي للتطوير؛ **Cloud** للإنتاج
- Concurrency: CPU=2 jobs، GPU=1 job
- طوابير: **gpu**, **cpu**, **merge**
- Timeouts: Audio ≤ 10m, Video ≤ 15m
- Retries: **3** (exponential backoff)
- ترتيب المهام: **لا** (مع idempotency)

### 6) تجربة المستخدم (Flutter)
- إدخال: **سحب-وإفلات** + **اختيار ملف** + **لصق رابط**
- قائمة المهام: الاسم/الرابط، الحالة، % التقدم، ETA، إلغاء
- المعاينة: **قص 10–15 ثانية** من البداية + تشغيل معاينة
- History محلي: **Hive** (آخر 20 عملية)
- لغات/اتجاه: **AR + EN**، **RTL/LTR** تلقائي
- العلامة المائية: موجودة في النسخة المجانية؛ تزال بالإعلانات أو Premium

### 7) API (FastAPI)
- `POST /probe` → فحص رابط/ملف (DRM/metadata)
- `POST /jobs` → إنشاء مهمة (ملف أو URL)
- `GET /jobs/{id}` → حالة + روابط تحميل
- `GET /jobs/{id}/events` → SSE/WebSocket للتقدّم
- `DELETE /jobs/{id}` → إلغاء
- `GET /health` → فحص الخدمة

### 8) الطوابير (Celery + Redis)
- حالات: queued/processing/merging/done/error/canceled
- Progress callback → Redis كل 1–2 ثانية
- إلغاء: **Soft-cancel** + kill ffmpeg إذا علّق
- تنظيف: حذف مؤقتات عند التحميل أو بعد 10–15 دقيقة

### 9) التخزين
- محلي مؤقّت: `/tmp/vocalclean/{jobId}/input|work|output`
- روابط التحميل: **token صالح 10 دقائق** + delete-on-download
- لا نحتفظ بالفيديو الأصلي بعد التسليم

### 10) قاعدة البيانات (PostgreSQL)
- جدول `jobs`: (id, source_type, source_url, input_path, mode, model, status, progress, error, duration_ms, eta_seconds, idempotency_key, created_at, finished_at, user_id?)
- جدول `artifacts`: (id, job_id, type, path, format, duration_ms, created_at)
- فهارس: على `status`, `created_at desc`, و `idempotency_key`

### 11) الأمان والامتثال
- فحص MIME/امتداد/حجم قبل الحفظ
- DRM check + ffprobe
- Docker non-root containers
- Privacy Notice + رفض ملفات محمية DRM
- Log Redaction للتوكنات/المفاتيح
- رفض playlists + حدّ مدّة

### 12) المراقبة
- Metrics: Queue wait, Processing time, Success/Failure
- Logs: JSON بختم jobId
- Prometheus/Grafana لاحقًا

### 13) الاختبارات
- Golden set (10–20 مقطع) + قبول تقريبي (SI-SDR)
- Unit tests: FFmpeg wrapper, runners, API
- اختبارات التحمل: ملفات كبيرة/DRM/Timeout/شبكة

### 14) النشر وCI/CD
- صور Docker: api, worker, redis (+ nginx اختياري)
- GitHub Actions: build/test/push
- GPU image للـ worker لو متاح
- HTTPS/Ingress عند النشر العام

### 15) التراخيص
- Demucs/Spleeter/ffmpeg/yt-dlp (MIT)
- صفحة **About** تعرض الإصدارات

---

## المعمارية (مبسطة)
- **Flutter App** ↔ **FastAPI** ↔ **Redis** ↔ **Celery Workers** ↔ **PostgreSQL** ↔ **File system مؤقت**

---

## Backlog (MVP)

### Epic A — Backend/API
- نماذج Pydantic + `POST /probe` + رفع ملف + `/jobs` + `/jobs/{id}` + روابط تحميل مؤقتة + Cleaner job

### Epic B — Workers/Audio
- Demucs/Spleeter runners + FFmpeg wrapper + merge/normalize + progress callbacks + retries + Golden set

### Epic C — Flutter
- شاشة Upload + Probe UI + إعدادات Job + Preview + Tasks screen + History + Snackbars + Watermark logic + Theme toggle + Localization

---

## Sprint 1 (أسبوعان)
**الهدف:** رفع ملف صوتي (≤100MB) → فصل Demucs CPU → إخراج WAV/MP3 → رؤية التقدّم في Flutter → تنزيل النتيجة.

---

## قرارات UX نهائية
- زر “توليد معاينة 15 ثانية”
- Badge للحالة + Progress bar + ETA
- Watermark على الفيديوهات المجانية
- إزالة العلامة المائية عبر 3 إعلانات أو Premium

