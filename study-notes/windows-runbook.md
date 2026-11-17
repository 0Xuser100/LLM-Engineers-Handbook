# دليل تشغيل المشروع — Windows PowerShell

ملاحظات شخصية لتطبيق Chapter 2 وChapter 3 من LLM Engineer’s Handbook (2024).

## الإعداد لأول مرة

ثبت Git وDocker Desktop وpyenv-win أولًا. الأوامر التالية تفترض توفر pyenv. افتح Docker Desktop باستخدام Linux containers.

```powershell
pyenv --version
pyenv install 3.11.8
Set-Location D:\LLM-Engineers-Handbook
pyenv local 3.11.8
pyenv rehash
python --version
```

لو python يشير لإصدار آخر، أصلح ترتيب PATH للجلسة:

```powershell
$env:Path = "$env:USERPROFILE\.pyenv\pyenv-win\shims;$env:USERPROFILE\.pyenv\pyenv-win\bin;$env:Path"
pyenv rehash
python --version
```

ثبت Poetry 1.8.3 إن لم يكن مثبتًا:

```powershell
python -m pip install --user poetry==1.8.3
```

في إعداد الجهاز الذي تحققنا منه، Poetry موجود هنا. تحقق قبل إضافة المسار:

```powershell
$poetryDir = "$env:APPDATA\Python\Scripts"
Test-Path "$poetryDir\poetry.exe"
$env:Path = "$poetryDir;$env:Path"
poetry --version
```

لو المسار غير موجود بعد تثبيت جديد، استخدم python -m site --user-base لتحديد Scripts الفعلي. للإعداد الدائم أضف مجلد Poetry الصحيح ومجلدي pyenv-win\shims وpyenv-win\bin إلى User Path من Environment Variables دون استبدال بقية المسارات. راجع أي Python في Machine Path يسبقها. أغلق التطبيق الأب للـTerminal وافتحه مجددًا حتى يرث PATH الجديد. لا تعِد كتابة PATH بالكامل باستخدام setx.

## تثبيت dependencies

```powershell
Set-Location D:\LLM-Engineers-Handbook
poetry env use (pyenv which python)
poetry install --without aws
poetry run python --version
poetry env info
poetry self add 'poethepoet[poetry_plugin]'
poetry poe --help
if (-not (Test-Path .env)) { Copy-Item .env.example .env }
```

المؤكد في التجربة: Python 3.11.8 وPoetry 1.8.3 وValid: True. Poe plugin 0.49.0 مختلف عن poethepoet 0.29.0 داخل المشروع. لا تحدث lock أو dependencies لمجرد اتباع الدليل. راجع .env محليًا ولا ترفعه مع PR.

## بداية كل جلسة

```powershell
Set-Location D:\LLM-Engineers-Handbook
poetry --version
poetry run python --version
docker info --format '{{.OSType}}'
docker compose up -d
docker compose ps
Test-NetConnection 127.0.0.1 -Port 8237
```

المتوقع linux وحاويتا MongoDB وQdrant بحالة Up. لو منفذ ZenML غير متاح:

```powershell
poetry poe local-zenml-server-up
```

اترك النافذة مفتوحة لو السيرفر يعمل في المقدمة واستخدم نافذة ثانية. ZenML: http://localhost:8237 وQdrant: http://localhost:6333/dashboard.

```powershell
poetry run zenml status
```

لو العميل يشير لقاعدة محلية مختلفة عن السيرفر، راجع الاتصال ويمكن ربطه بالسيرفر الموجود:

```powershell
poetry run zenml login http://localhost:8237
```

## تشغيل ETL

```powershell
poetry poe --dry-run run-digital-data-etl-maxime
poetry poe run-digital-data-etl-maxime
poetry poe run-digital-data-etl-paul
```

كل أمر شغله بعد انتهاء السابق. مثال الأمر الذي يختصره Poe:

```powershell
poetry run python -m tools.run --run-etl --no-cache --etl-config-filename digital_data_etl_maxime_labonne.yaml
```

--no-cache يعيد تنفيذ ZenML Steps، لكنه لا يمسح MongoDB. Crawler يتخطى المقال الموجود بنفس الرابط، فتوقع Article already exists عند إعادة التشغيل.

Maxime يحتوي 26 رابطًا. Paul يحتوي 50 في الإعداد الأصلي. إذا فشل Chrome برسالة SessionNotCreatedException / chrome not reachable، حل الكتاب هو تعليق روابط Medium الخمسة في configs/digital_data_etl_paul_iusztin.yaml بوضع # قبل أسطرها، وترك Substack. أعد أمر Paul؛ المتوقع 45 رابطًا. هذا تجاوز Selenium وليس إصلاح Chrome.

الأمر الجامع يشغل Maxime ثم Paul، فلا يلزم تشغيله أيضًا:

```powershell
poetry poe run-digital-data-etl
```

## القراءة باستخدام ODM

```powershell
poetry run python -c "from llm_engineering.domain.documents import UserDocument, ArticleDocument; u = UserDocument.find(first_name='Paul', last_name='Iusztin'); articles = ArticleDocument.bulk_find(author_id=str(u.id)) if u else []; print('Paul articles:', len(articles)); print('First link:', articles[0].link if articles else 'No articles found')"
```

المؤكد في التطبيق: 45 مقالًا لـPaul. author_id محفوظ كنص؛ لذلك str(u.id). استخدم poetry run zenml وليس poetry zenml. تحذير ZenML secret store ثم الرجوع إلى .env ليس فشلًا لو الاتصال والتنفيذ نجحا.

## فحص MongoDB

اقرأ المستخدم من MONGO_INITDB_ROOT_USERNAME وكلمة المرور من MONGO_INITDB_ROOT_PASSWORD داخل docker-compose.yml المحلي. لا تحفظ كلمة المرور في الدليل. الأمر يطلبها تفاعليًا:

```powershell
$mongoUser = Read-Host 'MongoDB username from docker-compose.yml'
docker compose exec mongo mongosh --username $mongoUser --authenticationDatabase admin twin --password
```

admin قاعدة المصادقة وtwin قاعدة التطبيق. تغيير Environment بعد أول تهيئة لا يغير كلمة المرور القائمة تلقائيًا لأن البيانات في mongo_data Volume.

عند ظهور twin> نفذ JavaScript التالي داخل mongosh، وليس PowerShell:

```javascript
show collections
db.users.countDocuments({})
db.articles.countDocuments({})
db.articles.aggregate([
  { $group: { _id: "$author_full_name", articles: { $sum: 1 } } },
  { $sort: { _id: 1 } }
]).toArray()
db.articles.aggregate([
  { $group: { _id: "$link", count: { $sum: 1 } } },
  { $match: { count: { $gt: 1 } } }
]).toArray()
db.articles.find({}, {
  _id: 0, author_full_name: 1, link: 1,
  "content.Title": 1, "content.Content": 1
}).limit(2).toArray()
db.articles.countDocuments({ $or: [
  { "content.Content": { $exists: false } },
  { "content.Content": null },
  { "content.Content": /^\s*$/ }
] })
```

المؤكد في التجربة: مستخدمان، 71 مقالًا (26 Maxime و45 Paul)، قائمة التكرار فارغة، وعدد النصوص الفارغة 0. العينة بها محتوى مقالات وقوائم مواقع؛ لا يعني النص غير الفارغ صحة كل المقالات.

find: أول معامل Filter والثاني Projection؛ 1 يعرض و0 يخفي، وcontent.Title حقل متداخل. limit(2) لا يعني الأحدث دون sort. toArray يجمع Cursor للعرض. الأوامر السابقة قراءة فقط.

للرجوع إلى PowerShell:

```javascript
exit
```

## مراجعة ZenML

Pipelines → digital_data_etl → أحدث Run → user / crawled_links → Metadata. المؤكد: Maxime 24 Substack + رابطان مدونة، وPaul 45 Substack. crawled_links قائمة روابط؛ النصوص في MongoDB. successful يعني انتهاء extract دون استثناء ويشمل تخطي المقال الموجود. اللون الأخضر وحده لا يثبت نجاح كل رابط أو جودة النص.

## إيقاف الجلسة

```powershell
docker compose stop
```

اضغط Ctrl+C في نافذة ZenML لو شغال في المقدمة. local-zenml-server-down ينفذ logout --local؛ لا تفترض أنه يقتل أي سيرفر يدوي. لا تستخدم docker compose down -v لأن -v يحذف Volumes والبيانات.

## Backup وPR

لا نحتاج run-import-data-warehouse-from-json بعد نجاح الجمع. الاستيراد يستخدم insert_many وليس upsert وقد يسبب تكرار IDs وإدخالًا جزئيًا. أرقام الكتاب 88 مقالًا و3 مستخدمين تخص بياناته؛ ملفات النسخة المفحوصة تحتوي 76 مقالًا ومستخدمين اثنين. التصدير الافتراضي قد يستبدل JSON المرفقة؛ استخدم مجلدًا مستقلًا للنسخ الاحتياطية.

```powershell
git status --short
git diff -- configs/digital_data_etl_paul_iusztin.yaml
git diff --check
```

راجع تعديل Paul المحلي منفصلًا عن الدليل قبل PR. لا ترفع .env أو كلمات مرور أو Logs. لم يتم إنشاء Commit أو PR. التالي بعد اكتمال هذا التطبيق Chapter 4: RAG Feature Pipeline.
