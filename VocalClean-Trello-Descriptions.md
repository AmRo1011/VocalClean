# VocalClean — بطاقة وصف المهام (Sprints 1–4)

> تقسيم حسب السبرنت والتراك (Backend / Workers / Flutter).

---

## Sprint 1

### Backend
#### A1 — تعريف Pydantic models + error envelope
- **الهدف:** توحيد نماذج طلب/استجابة API وتعريف حEnvelope أخطاء قياسي.
- **المخرجات:** BaseModels للـ endpoints + Error schema موحّد + أمثلة Swagger.
- **معايير القبول:**
  - كل Endpoint له Request/Response واضح.
  - خطأ موحّد `{code, message, details?, requestId}`.
  - Swagger يعرِض الأمثلة.
- **اعتماديات:** لا شيء.  
- **ملاحظات:** راعِ i18n للرسائل مستقبلًا.

#### A3 — رفع ملف + Validation (size/MIME)
- **الهدف:** استقبال ملفات آمنة مع تحقق حجم/نوع.
- **المخرجات:** نقطة رفع تقبل MP3/MP4 فقط + قيود حجم مذكورة في SRS.
- **معايير القبول:** رفض غير المدعوم برسالة واضحة؛ حفظ مؤقت آمن في مجلد job.
- **اعتماديات:** A1 (نموذج خطأ).

#### A4 — POST /jobs (enqueue + idempotency)
- **الهدف:** إنشاء مهمة معالجة مع مفتاح Idempotency لمنع التكرار.
- **المخرجات:** سجل DB + إدراج في الطابور المناسب (cpu/gpu/merge).
- **معايير القبول:** رد `{id,status=queued}`؛ نفس المفتاح → نفس `id`.
- **اعتماديات:** Redis/Celery جاهزين، A3/A2.

#### A5 — GET /jobs/{id} + download token
- **الهدف:** إرجاع حالة/تقدم وروابط تنزيل مؤقتة.
- **المخرجات:** progress, eta_seconds, outputs(+ token 10m).
- **معايير القبول:** بعد أول تنزيل ناجح يتم حذف مخرجات job.
- **اعتماديات:** A4.

---

### Workers / Audio
#### B3 — FFmpeg wrapper (progress/timeout)
- **الهدف:** التفاف آمن حول ffmpeg مع قراءة progress وإدارة timeouts.
- **المخرجات:** وظيفة تشغيل + Parsing + إرسال callbacks إلى Redis/DB.
- **معايير القبول:** إلغاء إذا توقف >30s؛ Timeouts حسب نوع المهمة.
- **اعتماديات:** Redis/DB.

#### B2 — Demucs CPU runner + mix outputs
- **الهدف:** فصل الصوت (Acapella) على CPU وإنتاج WAV/MP3.
- **المخرجات:** تشغيل demucs؛ مزج/تحويل إلى 44.1kHz/16-bit (افتراضي).
- **معايير القبول:** جودة ≥ SI-SDR 6 dB على عيّنات Golden.
- **اعتماديات:** B3، FFmpeg.

#### B5 — progress → Redis/DB + states
- **الهدف:** تحديث الحالة: queued/processing/merging/done/error/canceled.
- **معايير القبول:** تحديث كل 1–2s؛ اتساق Redis↔DB.
- **اعتماديات:** B3.

#### B6 — retries/backoff + kill hung ffmpeg
- **الهدف:** إعادة المحاولة 3 مرات 1m→3m→7m مع قتل العمليات المعلّقة.
- **معايير القبول:** سجلات محاولات واضحة؛ لا إعادة لمحفوظات مستخدم خاطئة/DRM.
- **اعتماديات:** B3.

---

### Flutter
#### C1 — Skeleton + AR/EN + RTL
- **الهدف:** مشروع Flutter مُهيّأ مع Localization (AR/EN) وRTL/LTR.
- **معايير القبول:** تبديل اللغة من النظام؛ نصوص أساسية مترجمة.

#### C2 — Upload (file_picker) + POST /probe
- **الهدف:** رفع ملف/رابط YouTube وتشغيل `/probe`.
- **معايير القبول:** عرض title/thumbnail/duration/DRM; رسائل واضحة للأخطاء.

#### C5 — Tasks screen (progress/ETA/Cancel)
- **الهدف:** قائمة مهام بحالة/Progress/ETA + زر إلغاء.
- **معايير القبول:** Polling أو SSE؛ انتقال الحالات منطقي؛ إلغاء ينعكس بالـ API.

#### C6 — History (Hive) + Snackbars
- **الهدف:** حفظ آخر 20 عملية محليًا؛ تنبيهات سياقية.
- **معايير القبول:** استرجاع سريع؛ حذف أقدم عنصر عند الامتلاء.

---

## Sprint 2

### Backend
#### A2 — /probe endpoint (metadata + DRM)
- **الهدف:** فحص رابط/ملف (ffprobe/yt-dlp) واكتشاف DRM/المدّة.
- **معايير القبول:** رد `{ok, duration_sec, has_audio, is_drm, title, thumbnail}`؛ رفض playlists.

#### A6 — /jobs events (SSE/WebSocket)
- **الهدف:** بث لحظي للتقدم عبر SSE/WebSocket.
- **معايير القبول:** أحداث منتظمة؛ معالجة انقطاع الاتصال بإعادة الاشتراك.

#### A7 — DELETE /jobs (cancel)
- **الهدف:** إلغاء ناعم + قتل ffmpeg إذا علّق.
- **معايير القبول:** الحالة `canceled`؛ لا تسرّب ملفات مؤقتة.

---

### Workers / Audio
#### B1 — CPU/GPU split (CUDA detection)
- **الهدف:** اختيار المسار تلقائيًا بحسب المدّة وتوفر GPU.
- **معايير القبول:** ≥10 دقائق + GPU → gpu queue؛ غير ذلك cpu.

#### B4 — merge/normalize/remux
- **الهدف:** التطبيع وإعادة دمج الصوت للفيديو (إن لزم).
- **معايير القبول:** إخراج WAV/MP3/MP4 حسب اختيار المستخدم؛ Limiter اختياري.

#### B7 — Golden set basic test
- **الهدف:** مجموعة عينات (10–20) لاختبارات قبول سريعة.
- **معايير القبول:** Acapella: تقليل الخلفية ≥15 dB؛ Instrumental: bleed < −18 dB.

---

### Flutter
#### C3 — Job Settings UI
- **الهدف:** اختيار model/mode/sampleRate/bitDepth/normalize.
- **معايير القبول:** حفظ الخيارات مع كل Job؛ قيم افتراضية سليمة (Demucs, 44.1kHz, 16-bit).

#### C4 — Preview generator (10–15s)
- **الهدف:** توليد معاينة من البداية وتشغيلها داخل التطبيق.
- **معايير القبول:** زر “Generate Preview”؛ تشغيل/إيقاف؛ حذف المعاينة عند الانتهاء.

#### C7 — Watermark handling
- **الهدف:** علامة مائية على الفيديوهات المجانية؛ الإزالة عبر 3 إعلانات أو Premium.
- **معايير القبول:** منطق واضح؛ حفظ حالة المستخدم محليًا.

#### C8 — Theme toggle
- **الهدف:** تبديل النظام/فاتح/داكن.
- **معايير القبول:** حفظ الاختيار؛ انسجام مع ألوان العلامات.

---

## Sprint 3

### Backend
#### A8 — Temp download links (delete-on-download)
- **الهدف:** توليد رابط تنزيل صالح 10 دقائق؛ حذف عند أول تنزيل ناجح.
- **معايير القبول:** توقيع token آمن؛ عمليات متزامنة محمية.

#### A9 — Cleaner job
- **الهدف:** حذف دورِي لملفات tmp المنتهية.
- **معايير القبول:** لا يترك مخلفات؛ سجلات تنظيف مفهومة.

---

### Workers / Audio
#### B2 — Demucs GPU runner
- **الهدف:** تمكين Demucs على GPU مع fallbacks لـ CPU.
- **معايير القبول:** اختيار تلقائي؛ قياس وقت المعالجة؛ نفس الجودة.

#### B5 — Progress callbacks (تعزيز)
- **الهدف:** توحيد صيغة progress/eta وإرسالها دوريًا.
- **معايير القبول:** فرق زمني ≤2s؛ تحمل انقطاع Redis.

#### B6 — Retries + kill hung ffmpeg (تحسين)
- **الهدف:** صقل السلوك وحدود الوقت؛ تغطية حالات Network/Timeout.
- **معايير القبول:** backoff مضبوط؛ لا إعادة لمحفوظات خاطئة/DRM.

---

### Flutter
#### C5 — Tasks screen (تحسين)
- **الهدف:** تحسين التجربة، فرز/فلترة، أيقونات حالات.
- **معايير القبول:** فلترة بالحالة والـ labels؛ انتقالات سلسة.

#### C6 — History (تحسين)
- **الهدف:** تفاصيل أكثر (إعدادات job/نتائج) وإعادة تنزيل إن متاح.
- **معايير القبول:** عرض الإعدادات المستخدمة؛ روابط صالحة إذا لم تُحذف بعد.

---

## Sprint 4 (Stabilization & UX)

### Backend
#### A10 — Docker + compose.dev
- **الهدف:** صور Docker (api, worker-cpu, worker-gpu, redis, postgres).
- **معايير القبول:** `docker-compose up` يعمل محليًا؛ non-root؛ مجلدات معزولة.

### Workers / Audio
#### B3 — FFmpeg wrapper (صقل نهائي)
- **الهدف:** تقارير أخطاء محسنة + مقاييس زمنية.
- **معايير القبول:** رموز أخطاء معيارية؛ تغطية حالات edge.

### Flutter
#### C9 — SSE/WebSocket progress
- **الهدف:** دمج بث حي للتقدم بدل Polling حيثما أمكن.
- **معايير القبول:** سقوط تلقائي لـ Polling عند فشل SSE؛ مؤشرات اتصال.

---

## ملاحظات عامة (لجميع التراكات)
- **قانوني/حقوق:** شاشة موافقة قبل أول استخدام؛ رفض DRM تلقائي برسالة ودّية.
- **الأمن:** فحص MIME/الامتداد/الحجم قبل الحفظ؛ حجب التوكنات في الـ Logs.
- **الأداء:** CPU=2 jobs/instance، GPU=1؛ Timeouts واضحة؛ Cleaner دوري.
- **المراقبة (لاحقًا):** Queue wait, Processing time, Success/Failure, Resource usage.

