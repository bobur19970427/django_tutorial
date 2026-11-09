# 3. Field turlari — to'liq qo'llanma

Bu darsda Django dagi barcha asosiy field turlarini, ularning **bazadagi SQL turi**, **maxsus parametrlari**, **qachon ishlatish** va **tuzoqlarini** ko'rib chiqamiz.

> Jadvallardagi SQL turlari **PostgreSQL** uchun berilgan. SQLite/MySQL da farq qilishi mumkin.

## Mundarija

1. [Avtomatik ID fieldlar](#1-avtomatik-id-fieldlar)
2. [Matn (string) fieldlar](#2-matn-string-fieldlar)
3. [Raqamli fieldlar](#3-raqamli-fieldlar)
4. [Boolean](#4-booleanfield)
5. [Sana va vaqt fieldlari](#5-sana-va-vaqt-fieldlari)
6. [Fayl fieldlari](#6-fayl-fieldlari)
7. [JSONField](#7-jsonfield)
8. [Maxsus fieldlar: UUID, IP, Binary](#8-maxsus-fieldlar)
9. [GeneratedField](#9-generatedfield-django-50)
10. [CompositePrimaryKey](#10-compositeprimarykey-django-52)
11. [PostgreSQL ga xos fieldlar](#11-postgresql-ga-xos-fieldlar)
12. [Qaysi fieldni tanlash kerak — cheat sheet](#12-cheat-sheet)

---

## 1. Avtomatik ID fieldlar

| Field | SQL (PostgreSQL) | Diapazon |
|-------|------------------|----------|
| `AutoField` | `integer` identity | 1 … 2 147 483 647 |
| `BigAutoField` | `bigint` identity | 1 … 9 223 372 036 854 775 807 |
| `SmallAutoField` | `smallint` identity | 1 … 32 767 |

```python
class Country(models.Model):
    id = models.SmallAutoField(primary_key=True)   # mamlakatlar soni kichik — joy tejaymiz
    name = models.CharField(max_length=100)
```

- Odatda o'zingiz yozmaysiz — `settings.DEFAULT_AUTO_FIELD` yoki `AppConfig.default_auto_field` belgilaydi.
- **Tavsiya:** yangi loyihalarda doim `BigAutoField`. `AutoField` ning 2.1 mlrd chegarasi katta jadvallarda (loglar, eventlar) tugab qolishi mumkin va keyin PK turini o'zgartirish juda og'ir migratsiya.

---

## 2. Matn (string) fieldlar

### 2.1. `CharField`

```python
title = models.CharField(max_length=200)
```

| | |
|--|--|
| SQL | `varchar(max_length)` |
| Majburiy parametr | `max_length` (PostgreSQL da Django 4.2+ dan beri ixtiyoriy — cheklovsiz `varchar`) |
| Qo'shimcha | `db_collation` — tartiblash/solishtirish qoidasi |
| Forma widgeti | `<input type="text">` |

**Qachon:** qisqa, chegaralangan matn — ism, sarlavha, kod, telefon.

`max_length` ham baza darajasida (`varchar(200)`), ham validatsiya darajasida tekshiriladi.

```python
# Katta-kichik harfni farqlamaydigan unique (PostgreSQL), db_collation orqali:
# avval migratsiyada CreateCollation qilish kerak, keyin:
username = models.CharField(max_length=150, unique=True, db_collation="case_insensitive")
```

### 2.2. `TextField`

```python
body = models.TextField()
```

| | |
|--|--|
| SQL | `text` |
| `max_length` | Faqat forma widgetiga (`textarea`) ta'sir qiladi, bazada cheklov **yo'q** |
| Forma widgeti | `<textarea>` |

**Qachon:** uzun, chegarasiz matn — maqola, izoh, tavsif.

> PostgreSQL da `varchar(n)` va `text` tezlikda farq qilmaydi. Farq — faqat uzunlik cheklovida.

### 2.3. `SlugField`

```python
slug = models.SlugField(max_length=255, unique=True)
```

| | |
|--|--|
| SQL | `varchar(50)` (default `max_length=50`) + **avtomatik index** (`db_index=True`) |
| Ruxsat etilgan belgilar | harflar, raqamlar, `-`, `_` |
| `allow_unicode` | `True` bo'lsa lotin bo'lmagan harflarga ham ruxsat (masalan, `книга-1`) |

**Qachon:** URL uchun: `/blog/django-modellar-asoslari/`.

Slug ni avtomatik yaratish:

```python
from django.utils.text import slugify

class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=220, unique=True, blank=True)

    def save(self, *args, **kwargs):
        if not self.slug:
            base = slugify(self.title) or "post"
            slug, n = base, 1
            while Post.objects.filter(slug=slug).exclude(pk=self.pk).exists():
                n += 1
                slug = f"{base}-{n}"
            self.slug = slug
        super().save(*args, **kwargs)
```

> ⚠️ `slugify("O'zbekiston")` → `"ozbekiston"` (apostrof tashlab yuboriladi). Lotin bo'lmagan matnda `slugify(text, allow_unicode=True)` ishlating.

### 2.4. `EmailField`

```python
email = models.EmailField(unique=True)
```

- SQL: `varchar(254)` (default `max_length=254` — RFC 3696 bo'yicha).
- Validatsiya: `EmailValidator`.
- ⚠️ Email katta-kichik harfga sezgir saqlanadi: `Ali@mail.com` va `ali@mail.com` — ikki xil qiymat! Saqlashdan oldin normalizatsiya qiling:

```python
def save(self, *args, **kwargs):
    self.email = self.email.lower().strip()
    super().save(*args, **kwargs)
```

Yoki baza darajasida `UniqueConstraint(Lower("email"), name="unique_lower_email")` (5-dars).

### 2.5. `URLField`

```python
website = models.URLField(max_length=500, blank=True)
```

- SQL: `varchar(200)` (default).
- Validatsiya: `URLValidator` (`http`, `https`, `ftp`, `ftps`).
- ⚠️ Zamonaviy URL lar ko'pincha 200 belgidan uzun — `max_length` ni oshiring.

### 2.6. `GenericIPAddressField`

```python
ip = models.GenericIPAddressField(protocol="both", unpack_ipv4=False, null=True, blank=True)
```

- SQL: `inet` (PostgreSQL), boshqa bazalarda `char(39)`.
- `protocol`: `"both"`, `"IPv4"`, `"IPv6"`.
- `unpack_ipv4=True` → `::ffff:192.0.2.1` ni `192.0.2.1` ga aylantiradi.
- ⚠️ `blank=True` bo'lsa `null=True` ham **shart** (bo'sh string IP sifatida saqlanmaydi).

### 2.7. `FilePathField`

Serverdagi ma'lum papkadagi fayl nomlaridan tanlash (kam ishlatiladi):

```python
template = models.FilePathField(path="/srv/templates", match=r".*\.html$", recursive=True)
```

---

## 3. Raqamli fieldlar

### 3.1. Butun sonlar

| Field | SQL | Diapazon |
|-------|-----|----------|
| `SmallIntegerField` | `smallint` | -32 768 … 32 767 |
| `IntegerField` | `integer` | -2 147 483 648 … 2 147 483 647 |
| `BigIntegerField` | `bigint` | ±9.2 × 10¹⁸ |
| `PositiveSmallIntegerField` | `smallint` + `CHECK >= 0` | 0 … 32 767 |
| `PositiveIntegerField` | `integer` + `CHECK >= 0` | 0 … 2 147 483 647 |
| `PositiveBigIntegerField` | `bigint` + `CHECK >= 0` | 0 … 9.2 × 10¹⁸ |

```python
class Product(models.Model):
    stock = models.PositiveIntegerField(default=0)          # manfiy bo'lishi mumkin emas
    views = models.PositiveBigIntegerField(default=0)       # juda katta bo'lishi mumkin
    age_limit = models.PositiveSmallIntegerField(default=0) # 0..18
    temperature = models.SmallIntegerField()                # manfiy bo'lishi mumkin
```

- `Positive*` fieldlar bazada **CHECK constraint** yaratadi — bu haqiqiy himoya.
- ⚠️ Nomiga qaramay `Positive*` **0 ni ham** qabul qiladi. Qat'iy musbat kerak bo'lsa → `MinValueValidator(1)` + `CheckConstraint`.
- To'g'ri o'lchamni tanlash joy tejaydi: 100 mln qatorli jadvalda `smallint` (2 bayt) vs `bigint` (8 bayt) — 600 MB farq.

### 3.2. `FloatField`

```python
latitude = models.FloatField()
```

- SQL: `double precision`.
- Python `float` — **taxminiy** son (IEEE 754).

```python
>>> 0.1 + 0.2
0.30000000000000004
```

**Qachon:** ilmiy hisob, koordinata, sensor ma'lumotlari — kichik xatolik muhim bo'lmagan joyda.
**Qachon EMAS:** ❌ **Pul!** Hech qachon pulni `FloatField` da saqlamang.

### 3.3. `DecimalField`

```python
price = models.DecimalField(max_digits=12, decimal_places=2)
```

| | |
|--|--|
| SQL | `numeric(12, 2)` |
| Python turi | `decimal.Decimal` — **aniq** son |
| `max_digits` | Jami raqamlar soni (butun + kasr) |
| `decimal_places` | Kasr qismidagi raqamlar soni |

`max_digits=12, decimal_places=2` → eng katta qiymat `9 999 999 999.99`.

```python
from decimal import Decimal

p = Product(price=Decimal("19.99"))      # ✅ string dan Decimal
p = Product(price=19.99)                 # ⚠️ float dan — aniqlik yo'qolishi mumkin
Decimal(19.99)                           # Decimal('19.989999999999998436805981327779591083526611328125')
Decimal("19.99")                         # Decimal('19.99')
```

**Qoida:** pul, foiz, valyuta kursi, og'irlik (aniq hisob kerak bo'lsa) → `DecimalField`.

> Alternativa: pulni eng kichik birlikda `BigIntegerField` sifatida saqlash (tiyin/sentlarda: 1999 = $19.99). Stripe kabi to'lov tizimlari shunday qiladi.

---

## 4. `BooleanField`

```python
is_active = models.BooleanField(default=True)
is_verified = models.BooleanField(null=True)   # True / False / None (noma'lum) — "uch holatli"
```

- SQL: `boolean`.
- `default` berilmasa, qiymat `None` bo'ladi va `null=False` bo'lgani uchun saqlashda xato chiqadi → **doim `default` bering**.
- Eski `NullBooleanField` o'chirilgan → `BooleanField(null=True)` ishlating.

> **Dizayn maslahati:** bir nechta boolean (`is_draft`, `is_published`, `is_archived`) bir-birini istisno qilsa, ularni bitta `status` (TextChoices) ga birlashtiring. Aks holda "ham draft, ham archived" kabi noto'g'ri holatlar paydo bo'ladi.

---

## 5. Sana va vaqt fieldlari

| Field | SQL | Python turi |
|-------|-----|-------------|
| `DateField` | `date` | `datetime.date` |
| `TimeField` | `time` | `datetime.time` |
| `DateTimeField` | `timestamp with time zone` | `datetime.datetime` |
| `DurationField` | `interval` | `datetime.timedelta` |

### 5.1. `auto_now` va `auto_now_add`

```python
class Post(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)   # faqat YARATILGANDA bir marta
    updated_at = models.DateTimeField(auto_now=True)       # HAR save() da yangilanadi
```

| | `auto_now_add=True` | `auto_now=True` |
|--|---------------------|-----------------|
| Qachon yoziladi | Birinchi `save()` (INSERT) | Har `save()` |
| Qo'lda o'zgartirsa bo'ladimi | ❌ (e'tiborsiz qoldiriladi) | ❌ |
| Admin da | Ko'rinmaydi (`editable=False`) | Ko'rinmaydi |

⚠️ **Tuzoqlar:**
- `QuerySet.update()` da `auto_now` **ishlamaydi** (chunki `save()` chaqirilmaydi):
  ```python
  Post.objects.filter(pk=1).update(title="X")                         # updated_at o'zgarmaydi!
  Post.objects.filter(pk=1).update(title="X", updated_at=timezone.now())  # qo'lda berish kerak
  ```
- `save(update_fields=["title"])` qilsangiz ham `updated_at` yangilanmaydi — uni ro'yxatga qo'shing: `update_fields=["title", "updated_at"]`.
- `auto_now_add` qiymatini testlarda yoki import da o'zgartirib bo'lmaydi. Moslashuvchanlik kerak bo'lsa → `default=timezone.now` ishlating (bu holda qo'lda ham berish mumkin).
- `auto_now`, `auto_now_add` va `default` — uchalasidan faqat bittasini ishlatish mumkin.

### 5.2. Timezone (vaqt mintaqasi) — juda muhim!

```python
# settings.py
USE_TZ = True                # (default True) — bazada UTC da saqlanadi
TIME_ZONE = "Asia/Tashkent"  # template va forma uchun lokal vaqt
```

```python
from django.utils import timezone
import datetime

timezone.now()               # ✅ aware datetime (UTC)
datetime.datetime.now()      # ❌ naive datetime — RuntimeWarning: received a naive datetime
timezone.localtime()         # Toshkent vaqti
timezone.localdate()         # bugungi sana (Toshkent bo'yicha) — date.today() o'rniga
```

> ⚠️ Kun bo'yicha filtrlashda e'tibor bering: `Post.objects.filter(created_at__date=date(2026, 1, 1))` — Django `__date` ni joriy faol timezone bo'yicha hisoblaydi.

### 5.3. `DurationField`

```python
from datetime import timedelta

class Lesson(models.Model):
    duration = models.DurationField(default=timedelta(minutes=45))

Lesson.objects.filter(duration__gt=timedelta(hours=1))
```

PostgreSQL da `interval`, boshqalarda `bigint` (mikrosekund) sifatida saqlanadi.

---

## 6. Fayl fieldlari

### 6.1. `FileField`

```python
def user_directory_path(instance, filename):
    # MEDIA_ROOT/uploads/user_<id>/<filename>
    return f"uploads/user_{instance.owner_id}/{filename}"


class Document(models.Model):
    owner = models.ForeignKey("auth.User", on_delete=models.CASCADE)
    file = models.FileField(upload_to=user_directory_path, max_length=255)
    attachment = models.FileField(upload_to="attachments/%Y/%m/%d/", blank=True)
```

| | |
|--|--|
| SQL | `varchar(100)` — **faylning o'zi emas, faqat yo'li (path)** saqlanadi |
| `upload_to` | Papka nomi (strftime formatlari bilan) yoki callable `(instance, filename)` |
| `storage` | Saqlash backend (lokal disk, S3 ...) — callable ham bo'lishi mumkin |
| `max_length` | Default 100 — chuqur papkalar va uzun nomlar uchun oshiring |

```python
# settings.py
MEDIA_ROOT = BASE_DIR / "media"
MEDIA_URL = "/media/"
```

Fayl bilan ishlash:

```python
doc.file            # FieldFile obyekti
doc.file.name       # 'uploads/user_1/report.pdf'  (bazadagi qiymat)
doc.file.url        # '/media/uploads/user_1/report.pdf'
doc.file.path       # '/abs/path/media/uploads/...' (faqat lokal storage)
doc.file.size       # baytlarda
doc.file.open() / doc.file.read()

from django.core.files.base import ContentFile
doc.file.save("hello.txt", ContentFile(b"Salom"), save=True)

doc.file.delete(save=False)   # faylni diskdan o'chiradi
```

⚠️ **Muhim tuzoqlar:**
1. **Obyekt o'chirilganda fayl diskdan o'chMAYDI.** Buni o'zingiz qilishingiz kerak (`post_delete` signal yoki `django-cleanup` paketi).
2. Fayl yangilanganda eski fayl ham o'chmaydi.
3. `upload_to` callable da `instance.pk` hali **None** bo'ladi (obyekt saqlanmagan) — PK ga tayanmang, `uuid` ishlating.
4. Xavfsizlik: foydalanuvchi faylining kengaytmasini tekshiring (`FileExtensionValidator`) va hajmini cheklang.

```python
import uuid, os
from django.core.validators import FileExtensionValidator

def unique_upload(instance, filename):
    ext = os.path.splitext(filename)[1].lower()
    return f"docs/{uuid.uuid4().hex}{ext}"

file = models.FileField(
    upload_to=unique_upload,
    validators=[FileExtensionValidator(["pdf", "docx"])],
)
```

### 6.2. `ImageField`

```python
class Profile(models.Model):
    avatar = models.ImageField(
        upload_to="avatars/",
        width_field="avatar_width",     # ixtiyoriy: o'lchamlarni avtomatik saqlaydi
        height_field="avatar_height",
        blank=True,
    )
    avatar_width = models.PositiveIntegerField(null=True, blank=True)
    avatar_height = models.PositiveIntegerField(null=True, blank=True)
```

- `FileField` dan meros oladi + faylning haqiqiy rasm ekanligini tekshiradi.
- **Pillow** kutubxonasi kerak: `pip install Pillow`.
- `width_field` / `height_field` — rasm o'lchamlari yoziladigan field nomlari.

---

## 7. `JSONField`

```python
class Product(models.Model):
    name = models.CharField(max_length=200)
    attributes = models.JSONField(default=dict, blank=True)
    # {"color": "qora", "size": ["S", "M"], "specs": {"ram": 16, "cpu": "M3"}}
```

| | |
|--|--|
| SQL | `jsonb` (PostgreSQL), `json` (MySQL), `text` (SQLite, JSON1 bilan) |
| `encoder` / `decoder` | Maxsus turlarni (Decimal, datetime) serializatsiya qilish uchun, masalan `DjangoJSONEncoder` |

### JSON bo'yicha so'rovlar

```python
Product.objects.filter(attributes__color="qora")                 # kalit bo'yicha
Product.objects.filter(attributes__specs__ram__gte=16)           # ichma-ich kalit
Product.objects.filter(attributes__size__0="S")                  # massiv indeksi
Product.objects.filter(attributes__has_key="color")
Product.objects.filter(attributes__has_keys=["color", "size"])
Product.objects.filter(attributes__has_any_keys=["color", "weight"])
Product.objects.filter(attributes__contains={"color": "qora"})   # PostgreSQL, MySQL
Product.objects.filter(attributes__isnull=True)                  # SQL NULL
Product.objects.filter(attributes=None)                          # JSON null
```

```python
from django.core.serializers.json import DjangoJSONEncoder

meta = models.JSONField(encoder=DjangoJSONEncoder, default=dict)   # Decimal, datetime, UUID ni saqlay oladi
```

⚠️ **Qachon JSONField ishlatMASLIK kerak:**
- Ma'lumot doim bir xil tuzilmaga ega bo'lsa → oddiy fieldlar yozing (validatsiya, index, constraint, FK ishlaydi).
- Unga tez-tez filter/JOIN qilinsa.

**Qachon yaxshi:** har xil mahsulot turlarining har xil atributlari, tashqi API javoblari, sozlamalar, loglar.

> Advanced: PostgreSQL da JSONField bo'yicha tez qidiruv uchun `GinIndex(fields=["attributes"])` qo'ying.

---

## 8. Maxsus fieldlar

### 8.1. `UUIDField`

```python
import uuid

class Order(models.Model):
    id = models.BigAutoField(primary_key=True)                     # ichki ishlatish uchun
    public_id = models.UUIDField(default=uuid.uuid4, unique=True, editable=False)  # URL/API uchun
```

- SQL: `uuid` (PostgreSQL), boshqalarda `char(32)`.
- `default=uuid.uuid4` — **qavssiz**!

**Nima uchun UUID?** `/orders/15/` URL i raqiblarga buyurtmalar sonini ochib beradi va keyingi ID ni taxmin qilish oson (IDOR zaifligi). `/orders/3f2a.../` — taxmin qilib bo'lmaydi.

**UUID ni PK qilishmi?** 
- ➕ Taqsimlangan tizimlar, ID ni oldindan yaratish mumkin.
- ➖ 16 bayt (bigint 8 bayt), tasodifiy UUIDv4 B-tree indexni parchalaydi → INSERT sekinlashadi.
- Ko'p hollarda yaxshi yechim: yuqoridagidek **ichki BigAutoField PK + tashqi UUID**. Python 3.14+ dagi `uuid.uuid7` (vaqt bo'yicha tartiblangan) bu muammoni kamaytiradi.

### 8.2. `BinaryField`

```python
data = models.BinaryField()
```

- SQL: `bytea`. Python turi `bytes` / `memoryview`.
- Default `editable=False`.
- ⚠️ Fayllarni bazada saqlash uchun **ishlatmang** — `FileField` + storage ishlating. Faqat kichik binar ma'lumot (hash, shifrlangan token) uchun.

---

## 9. `GeneratedField` (Django 5.0+)

Qiymati boshqa ustunlardan **bazaning o'zida** avtomatik hisoblanadigan ustun.

```python
from django.db import models
from django.db.models import F
from django.db.models import Value
from django.db.models.functions import Concat


class OrderItem(models.Model):
    price = models.DecimalField(max_digits=10, decimal_places=2)
    quantity = models.PositiveIntegerField()

    total = models.GeneratedField(
        expression=F("price") * F("quantity"),
        output_field=models.DecimalField(max_digits=12, decimal_places=2),
        db_persist=True,     # True = STORED (diskda saqlanadi), False = VIRTUAL
    )


class Person(models.Model):
    first_name = models.CharField(max_length=50)
    last_name = models.CharField(max_length=50)
    full_name = models.GeneratedField(
        expression=Concat("first_name", Value(" "), "last_name"),
        output_field=models.CharField(max_length=101),
        db_persist=True,
    )
```

```python
item = OrderItem.objects.create(price=10, quantity=3)
item.refresh_from_db()   # generated qiymatni olish uchun (ba'zi bazalarda)
item.total               # Decimal('30.00')
OrderItem.objects.filter(total__gt=100).order_by("-total")   # index ham qo'ysa bo'ladi
```

- PostgreSQL faqat `db_persist=True` ni qo'llab-quvvatlaydi.
- Qiymatni qo'lda berib bo'lmaydi.
- `@property` dan farqi: bazada mavjud → filter, order_by, index ishlaydi.

---

## 10. `CompositePrimaryKey` (Django 5.2+)

Bir nechta ustundan iborat primary key:

```python
class Enrollment(models.Model):
    pk = models.CompositePrimaryKey("student_id", "course_id")
    student = models.ForeignKey("Student", on_delete=models.CASCADE)
    course = models.ForeignKey("Course", on_delete=models.CASCADE)
    enrolled_at = models.DateTimeField(auto_now_add=True)
```

```python
e = Enrollment.objects.get(pk=(1, 5))
e.pk     # (1, 5)
```

⚠️ Cheklovlar: admin da to'liq qo'llab-quvvatlanmaydi, boshqa modellar bunday modelga `ForeignKey` bilan bog'lana olmaydi, mavjud model PK sini migratsiya orqali composite ga o'zgartirib bo'lmaydi. Asosan legacy bazalar uchun. Yangi loyihada odatda **oddiy `id` + `UniqueConstraint(fields=["student", "course"])`** yaxshiroq.

---

## 11. PostgreSQL ga xos fieldlar

`django.contrib.postgres` (`INSTALLED_APPS` ga qo'shish tavsiya etiladi):

```python
from django.contrib.postgres.fields import ArrayField, HStoreField, DateTimeRangeField, IntegerRangeField


class Article(models.Model):
    tags = ArrayField(models.CharField(max_length=30), default=list, blank=True, size=10)
    matrix = ArrayField(ArrayField(models.IntegerField(), size=3), size=3)   # 3x3 matritsa


class Booking(models.Model):
    period = DateTimeRangeField()           # [boshlanish, tugash)
    age_range = IntegerRangeField()
```

```python
Article.objects.filter(tags__contains=["django"])        # barcha berilganlarni o'z ichiga oladi
Article.objects.filter(tags__overlap=["django", "drf"])  # kamida bittasi bor
Article.objects.filter(tags__len=3)
Article.objects.filter(tags__0="python")

from datetime import datetime
Booking.objects.filter(period__contains=datetime(2026, 5, 1, 10))
Booking.objects.filter(period__overlap=(start, end))
```

> `ArrayField` va JSONField — denormalizatsiya. Teglar bilan ko'p ishlansa (teg sahifasi, statistika) → alohida `Tag` modeli + `ManyToManyField` yaxshiroq.

Range fieldlar + `ExclusionConstraint` — "bir xona bir vaqtda ikki marta band qilinmasin" kabi qoidalarni bazada kafolatlaydi (5-dars).

---

## 12. Cheat sheet

| Ma'lumot | Field |
|----------|-------|
| Ism, sarlavha, qisqa kod | `CharField(max_length=...)` |
| Uzun matn | `TextField` |
| URL dagi qism | `SlugField(unique=True)` |
| Email | `EmailField(unique=True)` + lowercase |
| Telefon | `CharField(max_length=20)` (❌ IntegerField emas — `+998`, boshidagi 0 yo'qoladi) |
| Pochta indeksi, pasport seriyasi | `CharField` (raqamga o'xshasa ham — matematik amal qilinmaydi) |
| Soni (manfiy bo'lmaydi) | `PositiveIntegerField` |
| Ko'rishlar, katta hisoblagich | `PositiveBigIntegerField` |
| **Pul** | `DecimalField(max_digits=..., decimal_places=2)` |
| Koordinata, ilmiy qiymat | `FloatField` |
| Ha / yo'q | `BooleanField(default=...)` |
| Holat (status) | `CharField(choices=TextChoices)` |
| Tug'ilgan kun | `DateField` |
| Yaratilgan / yangilangan vaqt | `DateTimeField(auto_now_add / auto_now)` |
| Davomiylik | `DurationField` |
| Fayl / rasm | `FileField` / `ImageField` |
| Dinamik atributlar | `JSONField(default=dict)` |
| Tashqi (public) ID | `UUIDField(default=uuid.uuid4, unique=True)` |
| IP manzil | `GenericIPAddressField` |
| Hisoblanadigan ustun | `GeneratedField` |
| Teglar ro'yxati (oddiy) | `ArrayField` (PostgreSQL) |

---

## Topshiriqlar

1. Internet-do'kon uchun `Product` modelini yozing: `name`, `slug` (avtomatik), `sku` (unikal), `description`, `price` (Decimal), `discount_percent` (0–100), `stock`, `weight_kg`, `is_active`, `attributes` (JSON), `image`, `created_at`, `updated_at`, `public_id` (UUID). Har bir field turini nima uchun tanlaganingizni izoh bilan yozing.
2. `Decimal(0.1) + Decimal(0.2)` va `Decimal("0.1") + Decimal("0.2")` natijalarini solishtiring.
3. `OrderItem` ga `GeneratedField` bilan `total` qo'shing va `sqlmigrate` natijasini o'qing.
4. `JSONField` ga 10 ta turli mahsulot yozing va quyidagilarni toping: rangi "qora" bo'lganlar; `ram >= 16` bo'lganlar; `weight` kaliti yo'q bo'lganlar.
5. `updated_at = DateTimeField(auto_now=True)` bo'lgan modelda `QuerySet.update()` qilib, `updated_at` o'zgarmasligini isbotlang.
6. `Document` o'chirilganda faylni diskdan ham o'chiradigan `post_delete` signal yozing.

⬅️ [02-field-options.md](02-field-options.md) | ➡️ [04-relationship-fieldlar.md](04-relationship-fieldlar.md)
